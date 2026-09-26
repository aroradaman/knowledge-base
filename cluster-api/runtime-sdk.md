# Runtime SDK and Runtime Extensions

Operational map of CAPI's webhook-style extension mechanism: the hook catalog, the `ExtensionConfig` CRD and discovery, the runtime client/registry, how a blocking hook response becomes a requeue, where each hook is actually called from in the core controllers, and how the in-tree test extension is built and used by e2e. A Runtime Extension is an external HTTPS server; CAPI controllers POST a typed request and act on the typed response. Nothing here is a Kubernetes admission webhook — it is CAPI's own protocol with its own versioned request/response types, discovery, timeouts and failure policy.

Feature gate: `RuntimeSDK` (`feature/feature.go:51`, alpha, default false). Topology Mutation hooks additionally need `ClusterTopology`.

## Hook catalog

Hook identity is a Go function value (e.g. `runtimehooksv1.BeforeClusterCreate`) registered into a catalog; `runtimecatalog.HookName(hook)` gives the wire name. Types live in `api/runtime/hooks/v1alpha1/`.

| Group | Hooks | File |
|---|---|---|
| Lifecycle | `BeforeClusterCreate`, `AfterControlPlaneInitialized`, `BeforeClusterUpgrade`, `BeforeControlPlaneUpgrade`, `AfterControlPlaneUpgrade`, `BeforeWorkersUpgrade`, `AfterWorkersUpgrade`, `AfterClusterUpgrade`, `BeforeClusterDelete` | `lifecyclehooks_types.go` (registrations at `:361-476`) |
| Topology mutation | `GeneratePatches`, `ValidateTopology`, `DiscoverVariables` | `topologymutation_types.go:105,191,218` |
| Upgrade planning | `GenerateUpgradePlan` | `upgrade_plan_types.go:110` |
| In-place updates | `CanUpdateMachine`, `CanUpdateMachineSet`, `UpdateMachine` | `inplaceupdate_types.go:108,176,224` |
| Discovery | `Discovery` | `discovery_types.go:103` |

Common contracts (`common_types.go`):
- `RequestObject` / `CommonRequest` carry `settings map[string]string`.
- `ResponseObject` / `CommonResponse` carry `status` (`Success`/`Failure`) and `message`.
- `RetryResponseObject` / `CommonRetryResponse` add `retryAfterSeconds` — **this is what makes a hook "blocking"**. Only hooks whose response embeds it can block (all `Before*` hooks, `AfterControlPlaneUpgrade`, `BeforeWorkersUpgrade`/`AfterWorkersUpgrade`; `AfterControlPlaneInitialized` and `AfterClusterUpgrade` cannot — see the `var _ ResponseObject` assertions).
- Builtin variables for topology mutation: `Builtins` / `BuiltinsName = "builtin"` (`topologymutation_variable_types.go:24,27`).

### Catalog

`api/runtime/catalog/` — `Catalog` (`catalog.go:33`) maps `GroupVersionHook` → `HookDescriptor` (request/response GVKs). Key methods: `AddHook`, `GroupVersionHook`, `NewRequest`/`NewResponse`, `ValidateRequest`/`ValidateResponse`, `Convert`, `IsHookRegistered`. `Builder` (`builder.go:26`) is how a hooks API package registers itself in `init()`. URL path scheme: `GVHToPath` (`catalog.go:375`) → `/<group>/<version>/<lowercased hook>/<lowercased handler name>`, e.g. `/hooks.runtime.cluster.x-k8s.io/v1alpha1/beforeclusterupgrade/my-handler`. Discovery has no name segment.

Hook API versioning is real: if the extension registered `v1alpha1` but the caller uses another version, `httpCall` converts request and response through the catalog's conversion funcs (`internal/runtime/client/client.go:618-645`). `internal/runtime/test/v1alpha1` + `v1alpha2` are fake hook APIs used to test that path.

## ExtensionConfig and discovery

- CRD types: `api/runtime/v1beta2/extensionconfig_types.go` — `ExtensionConfigSpec` (`:28`) = `clientConfig` (`url` **or** `service`, plus `caBundle`), `namespaceSelector`, `settings`. `ExtensionConfigStatus.handlers` holds the discovered handlers (`ExtensionHandler` at `:164`, with `requestHook`, `timeoutSeconds`, `failurePolicy`, `settings`).
- `FailurePolicy`: `Ignore` | `Fail` (`:208`).
- Annotations (`:309-321`): `runtime.cluster.x-k8s.io/inject-ca-from-secret` (`<ns>/<name>`), `runtime.cluster.x-k8s.io/pending-hooks`, `runtime.cluster.x-k8s.io/ok-to-delete`.
- Conditions: `Discovered` (`ExtensionConfigDiscoveredCondition`, `:293`).
- Controller: `core/reconcilers/extensionconfig/extensionconfig_controller.go:145`.
  - Requeues every 10s until `RuntimeClient.IsReady()` (registry warm).
  - `reconcileCABundle` (`:299`) is a mini cainjector: reads `ca.crt` from the annotated Secret into `spec.clientConfig.caBundle` (cert-manager's cainjector doesn't handle CRs). Secrets are watched via a separate metadata-only cache (`PartialSecretCache`, built in `core/main.go:534`) and mapped back by index (`index.go`).
  - `discoverExtensionConfig` (`:273`) calls the `Discovery` hook and writes `status.handlers`; then `RuntimeClient.Register`.
  - `ReadOnly: true` mode only validates + registers without writing (used in tests).
  - Requeues every 1m for the first 30m after start (`postStartupRequeueWindow/After`, `:59,63`) because extension servers may still be coming up.
- Warmup: `core/reconcilers/extensionconfig/warmup.go` — a non-leader-election `manager.Runnable` that lists all ExtensionConfigs and calls `RuntimeClient.WarmUp` before controllers start serving, so a cold start can't silently skip hooks.

## Runtime client and registry

- Public interface: `exp/runtime/client/client.go:70 Client` — `WarmUp`, `IsReady`, `Discover`, `Register`, `Unregister`, `GetAllExtensions`, `CallAllExtensions`, `CallExtension`. Options: `WithCaching` (`:50`) with a `CallExtensionCacheEntry` keyed on extension name + ExtensionConfig resourceVersion + request.
- Implementation: `internal/runtime/client/client.go` — `New` (`:85`) also returns a `certwatcher.CertWatcher` for the client cert, added to the manager in `core/main.go:506-523` so a cert failure restarts the binary. Constructed only when `RuntimeSDK` is enabled; the shared `catalog` is the package var at `core/main.go:97`.
- Registry: `internal/runtime/registry/registry.go` — `ExtensionRegistry` keyed by `GroupHook`; `ExtensionRegistration` (`:60`) is the per-handler record (name, `GroupVersionHook`, `NamespaceSelector`, `TimeoutSeconds`, `FailurePolicy`, `Settings`, `ExtensionConfigResourceVersion`). Registrations are **not** ready until `WarmUp`.
- Handler naming: registered name is `<handlerName>.<extensionConfigName>` (`client.go:853 NameForHandler`, inverse `ExtensionNameFromHandlerName:861`). ClusterClass fields like `generatePatchesExtension` reference this full name.
- `CallExtension` (`:400`) sequence: validate request/response against catalog → look up registration → `matchNamespace` against `namespaceSelector` (mismatch is an **error**, not a skip) → resolve timeout (`registration.TimeoutSeconds`, else `runtimehooksv1.DefaultHandlersTimeoutSeconds = 10`, `discovery_types.go:26`) → merge `registration.Settings` into the request (`cloneAndAddSettings`) → optional cache hit → HTTP → failure policy → `validateResponseStatus`.
- `CallAllExtensions` (`:317`) calls matching handlers **sequentially** and returns on the first error (avoids stacking timeouts); successes are merged by `aggregateSuccessfulResponses` (`:366`): messages joined, `retryAfterSeconds` = lowest non-zero.
- `httpCall` (`:599`): POST JSON, `?timeout=` query param + `context.WithTimeout`, response body capped at `maxExtensionResponseBodyBytes = 20 MiB` (`:73`). Non-200 becomes `errCallingExtensionHandler`, which is the only error class `FailurePolicy: Ignore` swallows (internal errors and `status: Failure` are never ignored — see the comment block at `:394-399`).
- TLS: extension URLs must be `https` (`urlForExtension:727` rejects other schemes); server cert is verified against `spec.clientConfig.caBundle` with the service DNS name; client cert/key come from `--runtime-extension-client-cert-file` / `--runtime-extension-client-key-file` (`core/main.go:273-277`). HTTP clients are cached per (hostname, caBundle) (`getHTTPClient:519`).
- Metrics: `internal/runtime/metrics/metrics.go` — `requests_total`, `request_duration_seconds` under the runtime-SDK subsystem.
- Fakes for unit tests: `internal/runtime/client/fake/fake_client.go`.

### `settings` precedence

Settings reach a handler from two places and are merged, **request wins**:
- `ExtensionConfig.spec.settings` → stored on the registration → merged into every request by `cloneAndAddSettings` (`internal/runtime/client/client.go:573`), which does not overwrite keys already present on the request.
- Per-patch `ClusterClass.spec.patches[].external.settings` → written onto `req.Settings` immediately before the call and reset afterwards (`patches/external/external_patch_generator.go:52-59`; the request object is reused across patches, hence the reset). Same for `external_validator.go`.

So a per-ClusterClass-patch setting overrides the cluster-wide ExtensionConfig setting of the same key. Lifecycle hooks have no per-ClusterClass override; they only see ExtensionConfig settings.

## Blocking responses → requeue

Two mechanisms, both needed:

1. **In-memory, per reconcile**: `scope.HookResponseTracker` (`exp/topology/scope/hookresponsetracker.go`). Every hook response is `Add`ed; `IsBlocking(hook)` gates desired-state decisions; `AggregateRetryAfter()` is turned into `ctrl.Result{RequeueAfter: …}` at `core/reconcilers/topology/cluster/cluster_controller.go:412-416`.
2. **Persisted intent**: `internal/hooks/tracking.go` writes the `runtime.cluster.x-k8s.io/pending-hooks` annotation — `MarkAsPending`, `MarkObjectAsPending`, `IsPending`, `MarkAsDone`, plus `IsOkToDelete`/`MarkAsOkToDelete` for the delete annotation. This survives restarts, so an `After*` hook is guaranteed to be called even if the controller dies mid-upgrade.
3. Hook responses are also cached in `util/cache` (`HookEntry`, `NewHookEntryKey`, `cache.go:135-153`) so a blocking hook isn't re-called on every requeue within its retry window.

## Call sites

| Hook | Called from |
|---|---|
| `BeforeClusterCreate` | `core/reconcilers/topology/cluster/cluster_controller.go:447 callBeforeClusterCreateHook` (only when InfraRef and ControlPlaneRef are both unset) |
| `BeforeClusterDelete` | `core/reconcilers/topology/cluster/cluster_controller.go:556 reconcileDelete` (~`:565-604`) |
| `AfterControlPlaneInitialized` | `core/reconcilers/topology/cluster/reconcile_state.go:186` |
| `AfterClusterUpgrade` | `core/reconcilers/topology/cluster/reconcile_state.go:227` (waits until `AfterControlPlaneUpgrade` and `AfterWorkersUpgrade` are no longer pending) |
| `BeforeClusterUpgrade` | `exp/topology/desiredstate/lifecycle_hooks.go:39` |
| `BeforeControlPlaneUpgrade` | `exp/topology/desiredstate/lifecycle_hooks.go:128` |
| `AfterControlPlaneUpgrade` | `exp/topology/desiredstate/lifecycle_hooks.go:184` |
| `BeforeWorkersUpgrade` | `exp/topology/desiredstate/lifecycle_hooks.go:248` |
| `AfterWorkersUpgrade` | `exp/topology/desiredstate/lifecycle_hooks.go:314` |
| `GeneratePatches` | `core/reconcilers/topology/cluster/patches/external/external_patch_generator.go:47` |
| `ValidateTopology` | `core/reconcilers/topology/cluster/patches/external/external_validator.go:47 (externalValidator.Validate)`, driven from `patches/engine.go:135-153` |
| `DiscoverVariables` | `core/reconcilers/clusterclass/clusterclass_controller.go:281 reconcileVariables` |
| `GenerateUpgradePlan` | `exp/topology/desiredstate/upgrade_plan.go:449 GetUpgradePlanFromExtension`, selected in `desired_state.go:127` |
| `CanUpdateMachineSet` | `core/reconcilers/machinedeployment/machinedeployment_canupdatemachineset.go:89,172` |
| `CanUpdateMachine` / `UpdateMachine` | `core/reconcilers/machine/machine_controller_inplace_update.go:153,211`; pending marked in `core/reconcilers/machineset/machineset_controller.go:400`; helpers in `internal/util/inplace/inplace.go` |

Lifecycle hook ordering is enforced by the desired-state computation, not by the hook code: see `exp/topology/desiredstate/desired_state.go:548 computeControlPlaneVersion` and `:747 getHooksToMarkPending`. The Cluster webhook also refuses a version change while `AfterClusterUpgrade` is pending (`core/webhooks/admission/cluster.go:463`).

## Writing and running an extension

Server side (in-tree helpers, importable by providers):
- `exp/runtime/server/server.go` — `Server` wrapping controller-runtime's webhook server. `New(Options{Catalog, Port (default 9443), CertDir, CertName, KeyName, ...})`, then `AddExtensionHandler(ExtensionHandler{Hook, Name, HandlerFunc})` (`:168`). It auto-serves the `Discovery` hook from the registered handlers (`discoveryHandler:272`) and wraps each handler with marshalling + status handling (`wrapHandler:292`, `callHandler:308`).
- `exp/runtime/topologymutation/` — helpers for `GeneratePatches` implementations: `WalkTemplates` (`walker.go:80`, decodes each template, calls your typed mutator, emits JSON or JSON-merge patches), `GetVariable`/`GetStringVariable`/`GetObjectVariableInto`/`ToMap`/`MergeVariableMaps` (`variables.go`).

Reference implementation: `test/extension/`
- `main.go` — flags (`InitFlags:104`), then `setupTopologyMutationHookHandlers:287`, `setupLifecycleHookHandlers:322`, `setupInPlaceUpdateHookHandlers:411`, `setupUpgradePlanHookHandlers:446`. This is the file to copy when adding a hook.
- `handlers/topologymutation/handler.go` — `GeneratePatches:91`, `ValidateTopology:634`, `DiscoverVariables:642`, plus per-kind patch funcs.
- `handlers/lifecycle/handlers.go` — all nine lifecycle handlers; responses are **driven by a ConfigMap** in the workload cluster namespace (`readResponseFromConfigMap:277`) and each call is recorded back (`recordCallInConfigMap:346`). That is how e2e asserts hook ordering and blocking.
- `handlers/upgradeplan/handlers.go` (`DoGenerateUpgradePlan:56`), `handlers/inplaceupdate/handlers.go` (`DoCanUpdateMachine:117`, `DoCanUpdateMachineSet:165`, `DoUpdateMachine:215`).
- Deployment manifests: `test/extension/config/default/` (cert-manager Certificate + Service + `manager_webhook_patch.yaml`; the kustomization injects the cert-manager annotations).
- Build targets: `test-extension` appears in `ALL_GENERATE_MODULES`, `ALL_DOCKER_BUILD`, `ALL_DOCKER_BUILD_E2E` (`Makefile:290,823,829`). Tilt target at `Tiltfile:123`.

E2E usage:
- Declared as `type: RuntimeExtensionProvider` in `test/e2e/config/docker.yaml:209`, pointing at `test/extension/config/default`.
- `test/e2e/cluster_upgrade_runtimesdk.go` creates the `ExtensionConfig` with `extensionConfig(...)` (`:678`) — CA injection annotation, `Service` clientConfig, `settings` (`extensionConfigName`, `disableInPlaceUpdates`, `defaultAllHandlersToBlocking`), and a `namespaceSelector` on `kubernetes.io/metadata.name` so the extension only applies to the test namespace. Assertions via `checkLifecycleHookResponses` (`:721`).
- The ExtensionConfig name is templated into the ClusterClass (`EXTENSION_CONFIG_NAME`) because external patch fields reference handler names.
- Unit-test the handler logic directly: `test/extension/handlers/topologymutation/handler_test.go`, `handler_integration_test.go`.

OpenAPI spec for extension authors: generated by `make generate-go-openapi` from `api/runtime/hooks/v1alpha1` into `hack/tools/runtime-openapi-gen/` (`Makefile:545-558`), published as the `runtime-sdk-openapi.yaml` release asset.

## Adding a new hook (checklist)

1. Types + hook func in a new or existing file under `api/runtime/hooks/v1alpha1/`; register it in that file's `init()` via `catalogBuilder.RegisterHook(MyHook, &runtimecatalog.HookMeta{...})` (builder declared at `api/runtime/hooks/v1alpha1/groupversion_info.go:32`, exposed as `AddToCatalog`, wired in `core/main.go:164`).
2. Decide whether the response embeds `CommonResponse` (non-blocking) or `CommonRetryResponse` (blocking), and add the matching `var _ ResponseObject`/`RetryResponseObject` assertion.
3. `make generate` (deepcopy) and `make generate-go-openapi`.
4. Add the call site in the relevant controller, guarded by `feature.Gates.Enabled(feature.RuntimeSDK)`; use `GetAllExtensions` first if zero handlers must be a no-op, then `CallAllExtensions`; record into `scope.HookResponseTracker` (topology) and/or `internal/hooks` pending annotation if the intent must survive restarts.
5. Add a handler in `test/extension/handlers/...` and register it in `test/extension/main.go`, so e2e exercises it.
6. Update the relevant `docs/book/src/tasks/experimental-features/runtime-sdk/*.md` page.

## Debugging a hook call

- Is the ExtensionConfig discovered? `kubectl get extensionconfig -o yaml` → `status.handlers` + `Discovered` condition. No handlers means discovery failed or the server registered nothing.
- Is the handler name what the ClusterClass references? It must be `<handler>.<extensionConfigName>`.
- Controller logs: `CallExtension` logs at V(4) with `extensionHandler` and `hook` keys ("Calling extension handler", "Extension handler returned blocking response with retryAfterSeconds of N"). Bump `-v=5` to see cache hits and conversions.
- Metrics `capi_runtime_sdk_requests_total` / `..._request_duration_seconds` (labels include host, hook, status code) tell you whether the call left the process.
- On the Cluster, the `TopologyReconciled` condition message names the blocking hook (`ClusterTopologyReconciledHookBlockingReason`), and `runtime.cluster.x-k8s.io/pending-hooks` on the Cluster shows outstanding `After*` intents.

## Gotchas

- `RuntimeSDK` off ⇒ `runtimeClient` is nil and every hook call site is skipped behind `feature.Gates.Enabled(feature.RuntimeSDK)`. Forgetting that guard in a new call site nil-panics.
- Registry not warm ⇒ calls fail; ExtensionConfig reconcile requeues at 10s. A brand-new ExtensionConfig also needs a successful `Discovery` before any hook fires.
- `namespaceSelector` mismatch in `CallExtension` returns an error; only `GetAllExtensions`/`CallAllExtensions` treat it as "not applicable". Call `GetAllExtensions` first if zero handlers should be a no-op.
- `FailurePolicy: Ignore` does **not** cover a handler that answers `status: Failure` — that always errors out.
- A blocking response must be re-served consistently: if an extension stops returning `retryAfterSeconds`, the pending-hooks annotation is what finally clears (`MarkAsDone`), not the absence of a response.
- Requests are versioned objects; changing a hook's request/response struct requires conversion functions plus regenerating the OpenAPI + deepcopy.
- Response body over 20 MiB is rejected — relevant for `GeneratePatches` on large topologies.
- Tests: `internal/runtime/client/client_test.go`, `internal/runtime/registry/registry_test.go`, `exp/runtime/server/server_test.go`, `core/reconcilers/extensionconfig/{extensionconfig_controller,warmup}_test.go`, `internal/hooks/tracking_test.go`, `exp/topology/desiredstate/lifecycle_hooks_test.go`.

## Where to read more

- `docs/book/src/tasks/experimental-features/runtime-sdk/index.md`, `implement-extensions.md`, `implement-lifecycle-hooks.md`, `implement-topology-mutation-hook.md`, `implement-upgrade-plan-hooks.md`, `implement-in-place-update-hooks.md`, `deploy-runtime-extension.md`
- `docs/proposals/20220221-runtime-SDK.md`
- `docs/proposals/20220414-lifecycle-hooks.md`
- `docs/proposals/20220330-topology-mutation-hook.md`
- `docs/proposals/20240807-in-place-updates.md` and `20240807-in-place-updates-implementation-notes.md`
- `docs/proposals/20250513-chained-and-efficient-upgrades-for-clusters-with-managed-topologies.md`
- Companion guide: `.claude/concepts/clusterclass-topology.md`

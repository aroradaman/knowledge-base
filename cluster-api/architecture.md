# CAPI Repo Architecture

The 10,000-foot map of `kubernetes-sigs/cluster-api`: the four provider flavors, which directory implements which, what each manager binary wires up, the four Go modules, and the cross-cutting machinery (feature gates, `util/`, ClusterCache, CRD migrator) you hit within the first hour. Read this before touching code you haven't worked in. Note: `docs/book/src/developer/core/repository-layout.md` is **stale** — it still describes `internal/controllers`, root `main.go`, root `config/` and root `webhooks/`, none of which exist any more. The current target layout is defined by `docs/proposals/20260701-code-organization.md`.

## Provider flavors and where they live in this repo

CAPI is a set of cooperating controller managers, each a "provider" of one flavor. This repo ships one of each so it is self-testing.

| Flavor | Shorthand | Directory | Binary / CRDs |
|---|---|---|---|
| core | CAPI | `core/` (+ `api/core`, `api/addons`, `api/ipam`, `api/runtime`) | `core/main.go`; Cluster, Machine, MachineSet, MachineDeployment, MachinePool, MHC, ClusterClass, MachineDrainRule, ClusterResourceSet, ExtensionConfig, IPAddress(Claim) |
| bootstrap | CABPK | `bootstrap/kubeadm/` (+ `api/bootstrap/kubeadm`) | `bootstrap/kubeadm/main.go`; KubeadmConfig, KubeadmConfigTemplate |
| control plane | KCP | `controlplane/kubeadm/` (+ `api/controlplane/kubeadm`) | `controlplane/kubeadm/main.go`; KubeadmControlPlane, KubeadmControlPlaneTemplate |
| infrastructure | CAPD | `test/infrastructure/docker/` | `test/infrastructure/docker/main.go`; DevCluster, DevMachine, DevMachinePool (+Templates) |

- CAPD is dev/test only and lives in the `test/` module, so **its API types are in `test/infrastructure/docker/api/`, not in the `api/` module**.
- `test/infrastructure/inmemory/` has no `main.go`: it is a library (`test/infrastructure/inmemory/pkg`) that CAPD's `DevCluster`/`DevMachine` reconcilers use as an alternative backend to Docker containers, together with `test/infrastructure/container`.
- All real infrastructure providers (AWS, Azure, …) are out-of-tree; core talks to them only through unstructured objects + the contract in `internal/contract/`.

### The per-provider directory shape (enforced by the code-organization proposal)

Every in-tree production provider has the same subdirectories: `config/`, `pkg/`, `reconcilers/<type>/`, `setup/`, `webhooks/{admission,conversion}/`, `main.go`. So `core/reconcilers/machine/`, `bootstrap/kubeadm/reconcilers/kubeadmconfig/`, `controlplane/kubeadm/reconcilers/kubeadmcontrolplane/`, `test/infrastructure/docker/reconcilers/`. If you add a reconciler, put it in `<provider>/reconcilers/<name>/` and register it in that provider's `main.go`.

## Go modules

| Module | Path | Contains | Guarantees |
|---|---|---|---|
| `sigs.k8s.io/cluster-api/api` | `api/go.mod` | only API types, deepcopy, conversions, `api/runtime/catalog` | strongest: K8s API deprecation policy + semver |
| `sigs.k8s.io/cluster-api` | `go.mod` | everything else (core, KCP, CABPK, clusterctl, util) | best-effort; breaking changes possible even in patches |
| `sigs.k8s.io/cluster-api/test` | `test/go.mod` | e2e framework, e2e suites, CAPD, in-memory, test-extension | best-effort; keeps docker/moby deps out of the root module |
| (tools) | `hack/tools/go.mod` | codegen & CI binaries (`conversion-verifier`, `runtime-openapi-gen`, `prowjob-gen`, `mdbook`, `release`, `triage`, `tilt-prepare`) | none |

- `api/` was split out so that consumers importing CAPI types do not pull CAPI's whole dependency tree (issue #9011) and so CVE-driven Go-version bumps can be scoped. The root `go.mod:5` has `replace sigs.k8s.io/cluster-api/api => ./api`; `test/go.mod:5-8` replaces both `sigs.k8s.io/cluster-api` and `.../api` with relative paths.
- Adding a dependency to `api/go.mod` is a review-sensitive act. Keep it tiny.
- A `/utils` module (`sigs.k8s.io/cluster-api/utils`) is **planned but does not exist yet**; `util/` and `controllers/clustercache` are its future home. Expect movement.
- After any import change run `make generate-modules` (`go mod tidy` across all four modules); `make verify-modules` gates CI.

## Directory orientation

| Path | What it holds / when you touch it |
|---|---|
| `api/` | Pure API types per group/version (`api/core/v1beta1`,`v1beta2`, `api/bootstrap/kubeadm`, `api/controlplane/kubeadm`, `api/addons`, `api/ipam`, `api/runtime`). `api/deprecated/errors` = old error types. No webhooks, no clients. Change a type → `make generate`. |
| `core/` | The core provider: `main.go`, `setup/` (manager cache/client wiring), `reconcilers/*`, `webhooks/admission` + `webhooks/conversion`, `config/` (kustomize: crd, rbac, webhook, manager, certmanager, metrics, default). |
| `controllers/` | *Not* reconcilers for CAPI objects. Reusable machinery exported to providers: `clustercache/` (workload-cluster clients+caches), `crdmigrator/`, `remote/` (user-agent helpers only), `external/` (unstructured get/patch of provider objects, `tracker.go`), `noderefutil/`. |
| `internal/` | Not importable by providers, by design. `internal/contract/` (the provider contract: field paths & versions for infra/bootstrap/control-plane objects), `internal/runtime/{client,registry,metrics}` (Runtime SDK caller side), `internal/hooks/`, `internal/topology/*` (names, variables, checks, ownerrefs, upgrade), `internal/util/{ssa,patch,hash,inplace,tree,compare,taints}`, `internal/test/envtest` (envtest harness). |
| `bootstrap/kubeadm/` | CABPK. `pkg/cloudinit`, `pkg/ignition`, `pkg/locking` (control-plane init lock), `pkg/types` (kubeadm type shims). |
| `controlplane/kubeadm/` | KCP. `pkg/workload_cluster*.go` + `pkg/etcd`, `pkg/proxy` (etcd/apiserver access through the workload cluster), `pkg/desiredstate`, `pkg/filters.go`. |
| `exp/` | Experimental, **no compatibility guarantees**; graduation = moved out of `exp/` into a stable path (`exp/README.md`, CONTRIBUTING "Experiments"). What remains: `exp/runtime/{catalog,client,server,topologymutation}` (Runtime Extension *server* side, what extension authors import) and `exp/topology/{desiredstate,scope}`. Most of the old `exp/` has already been drained — do not add new dirs here without a proposal. |
| `pkg/` | Code shared *between* the in-tree providers. Today only `pkg/dynamiccache/` (a cache whose watched types are configured at runtime, used for InfraMachine/BootstrapConfig). |
| `util/` | Provider-facing helpers; see the table below. `util/deprecated/v1beta1/{conditions,patch,paused}` = v1beta1-era variants kept for providers mid-migration — don't use in new code. |
| `feature/` | `feature/feature.go` defines every gate + default; `feature/gates.go` exposes `feature.Gates` / `feature.MutableGates`. Adding a gate = one const + one entry in `defaultClusterAPIFeatureGates`. |
| `cmd/clusterctl/` | The CLI: `client/` (library API), `cmd/` (cobra), `internal/`, `api/` (Provider/metadata CRDs), `config/` (embedded provider list, cluster templates). |
| `test/` | `test/framework/` (e2e helpers, `ClusterProxy`), `test/e2e/` (reusable e2e specs + `config/`, `data/`), `test/infrastructure/` (CAPD, in-memory, container, kind), `test/extension/` (a real Runtime Extension server used in e2e). |
| `hack/` | `hack/tools/` (module of codegen/CI binaries), `hack/scripts/{ci,dev,docs,ensure,release,verify}`, `hack/observability` (Tilt-deployed grafana/loki/prometheus). |
| `docs/` | `docs/book/src/**` (the published book), `docs/proposals/` (CAEPs — read these for *why*), `docs/release/`, `docs/community/`. |
| `version/` | `version.Get()`, `version.MinimumKubernetesVersion`, `MinimumKubernetesVersionClusterTopology`. |
| `metadata.yaml`, `Tiltfile` | clusterctl contract version map; Tilt dev loop (provider contexts listed in `Tiltfile:55-130`, kept in sync with `hack/tools/internal/tilt-prepare`). |

## Manager binaries: the common shape

All four `main.go` files are deliberately near-identical. Read one, you've read all four.

1. Package-level `scheme`, `setupLog`, `controllerName`, and a block of flag vars.
2. `init()` registers schemes; core also does `runtimehooksv1.AddToCatalog(catalog)` (`core/main.go:145-165`).
3. `InitFlags(fs)` — logging flags, leader election, `--namespace`, `--watch-filter`, per-controller `--*-concurrency`, `--skip-crd-migration-phases`, `--kube-api-qps/burst`, `--clustercache-client-qps/burst`, webhook cert flags, then `flags.AddManagerOptions(fs, &managerOptions)` (`util/flags`) and `feature.MutableGates.AddFlag(fs)`.
4. `main()` → `ctrl.GetConfigOrDie()`, set QPS/Burst/UserAgent (`remote.DefaultClusterAPIUserAgent`) and `apiwarnings.DefaultHandler`, `version.CheckKubernetesVersion`, build `ctrl.Options`, `ctrl.NewManager`, then `setupChecks` / `setupIndexes` / `setupReconcilers` / `setupWebhooks`, then `mgr.Start`.
5. `concurrency(c)` returns `controller.Options{MaxConcurrentReconciles: c}`. Every reconciler is registered by `(&X{...}).SetupWithManager(ctx, mgr, concurrency(n))` — **a new reconciler is not running until you add that call.**

Per-binary specifics:

- **core** (`core/main.go`): `LeaderElectionID: "controller-leader-election-capi"`; cache/client options come from `core/setup/setup.go` (`ManagerCacheOptions`, `ManagerClientOptions`, `ClusterCacheCacheOptions`, `ClusterCacheClientOptions`, `CreateSecretCachingClient`, `NewDynamicCache`, `DynamicCacheOptions`). `setupIndexes` = `index.AddDefaultIndexes` (`util/index`). Builds an extra metadata-only `partialSecretCache` for ClusterResourceSet/ExtensionConfig secret watches. Gated blocks: `ClusterTopology` → ClusterClass + `topology/{cluster,machinedeployment,machineset}` reconcilers; `RuntimeSDK` → runtime client + ExtensionConfig reconciler; `MachinePool` → MachinePool reconciler. Validates `--remote-conditions-grace-period > --remote-connection-grace-period` and `>= 2m`.
- **CABPK** (`bootstrap/kubeadm/main.go`): one reconciler (`kubeadmconfig.Reconciler`), `--bootstrap-token-ttl`, `LeaderElectionID: "kubeadm-bootstrap-manager-leader-election-capi"`.
- **KCP** (`controlplane/kubeadm/main.go`): `--etcd-dial-timeout-duration`, `--etcd-call-timeout-duration`, `--etcd-client-log-level`; registers KCP with `ReconciliationTimeout: 3 * time.Minute` because it dials every etcd member. Under `InPlaceUpdates` it builds its own runtime client and a **read-only** `extensionconfig.Reconciler{ReadOnly: true}`.
- **CAPD** (`test/infrastructure/docker/main.go`): creates a `container.NewDockerClient()`, an `inmemoryruntime.NewManager` and an `inmemoryserver.NewWorkloadClustersMux` (fake apiservers for the in-memory backend), then DevMachine (`ReconciliationTimeout: 5 * time.Minute`), DevCluster, DevMachineTemplate, and DevMachinePool under `MachinePool`.

`ctrl.Options.Controller.UsePriorityQueue` is set from `feature.PriorityQueue` in all four; `CacheSyncTimeout` is raised to 5m in core.

## Feature gates

Defined once in `feature/feature.go:102` and shared by every provider (providers import `sigs.k8s.io/cluster-api/feature`). Set via `--feature-gates=Name=true`.

| Gate | Default | Stage |
|---|---|---|
| `PriorityQueue`, `ReconcilerRateLimiting` | true | GA |
| `MachinePool`, `MachineSetPreflightChecks` | true | Beta |
| `ClusterTopology`, `RuntimeSDK`, `InPlaceUpdates`, `MachineTaintPropagation`, `ClusterClassInlineTemplates`, `KubeadmBootstrapFormatIgnition` | false | Alpha |

CRDs for gated features are always installed; the gate only controls reconcilers and (via webhooks) whether the fields may be used. `ClusterTopology` also raises the minimum workload-cluster Kubernetes version (`version.MinimumKubernetesVersionClusterTopology`).

## Cross-cutting machinery you will hit immediately

- **Patch helper** — `util/patch/patch.go:NewHelper` / `(*Helper).Patch`. Every reconciler defers a patch of the object it owns; it computes optimistic-lock-safe status+spec patches and handles conditions in both v1beta1 and v1beta2 formats.
- **Conditions** — `util/conditions/` (`getter.go`, `setter.go`, `merge_strategies.go`, `summary.go`, `aggregate.go`, `mirror.go`). CAPI does metav1.Condition with opinionated merge/summary semantics; `util/conditions/deprecated/v1beta1` + `util/deprecated/v1beta1/conditions` hold the old `clusterv1.Conditions` API.
- **Predicates** — `util/predicates/{cluster_predicates.go,generic_predicates.go}`: `ResourceHasFilterLabel` (backs `--watch-filter`), `ResourceIsNotExternallyManaged`, cluster-paused/unpaused predicates. Every `SetupWithManager` composes these.
- **Paused / finalizers** — `util/paused/paused.go:EnsurePausedCondition` and `util/finalizers/finalizers.go:EnsureFinalizer` are the standard first two steps of a reconcile.
- **ClusterCache** — `controllers/clustercache/`. `SetupWithManager` (`cluster_cache.go:307`) returns a `ClusterCache` that is itself a controller reconciling Clusters. Interface (`cluster_cache.go:146`): `GetClient`, `GetReader`, `GetUncachedClient`, `GetRESTConfig`, `Watch`, `GetHealthCheckingState`. It health-probes workload clusters and drops accessors on failure, which is what drives `RemoteConnectionProbe` and the grace-period flags. `cluster_cache_fake.go` for tests.
- **DynamicCache** — `pkg/dynamiccache/`, configured in `core/setup/setup.go:DynamicCacheOptions`. Caches provider-owned (unstructured) InfraMachine/BootstrapConfig types whose GVKs are only known at runtime; passed to the Machine/MachineSet/MachinePool/KCP reconcilers as `DynamicCache`.
- **CRD migrator** — `controllers/crdmigrator/crd_migrator.go`. Each manager registers a `CRDMigrator` with a `map[client.Object]ByObjectConfig` listing its own CRDs, at concurrency 1. Phases: `StorageVersionMigration`, `CleanupManagedFields` (skippable via `--skip-crd-migration-phases`). **If you add a CRD to a provider you must add it to that provider's `crdMigratorConfig` map *and* to the `+kubebuilder:rbac` `resourceNames=` marker directly above `main()`** (see `core/main.go:296-302`) — they are hand-kept in sync.
- **Provider contract** — `internal/contract/` describes where in an unstructured provider object core expects fields (`infrastructure_cluster.go`, `infrastructure_machine.go`, `bootstrap.go`, `controlplane.go`, `version.go`, `references.go`). `util/contract/contract.go:CalculateCRDName` is the tiny public part. `controllers/external/` does the actual unstructured Get/Patch.
- **Conversion webhooks** — `<provider>/webhooks/conversion/`; each `main.go` calls `conversion.SetAPIVersionGetter(...)` backed by `internal/contract/version.go:GetGVKFromGK` so hub/spoke conversion can resolve a provider's current storage version at runtime.

## Management cluster vs workload cluster

- `mgr.GetClient()` / `mgr.GetAPIReader()` always mean the **management** cluster. Every CAPI CRD lives there.
- The **workload** cluster is reached only through `ClusterCache`: `r.ClusterCache.GetClient(ctx, util.ObjectKey(cluster))` (e.g. `core/reconcilers/machine/machine_controller_noderef.go:76`). Never build a workload client by hand in a reconciler.
- Chain: `Cluster` → kubeconfig Secret `<cluster>-kubeconfig` in the management cluster → `clustercache/cluster_accessor_client.go:createRESTConfig` → per-cluster `rest.Config` + HTTP client + RESTMapper + cached client and `stoppableCache`. Secrets are read via the dedicated secret-caching client (`core/setup/setup.go:CreateSecretCachingClient`), not the manager cache.
- To watch a workload-cluster resource (e.g. Nodes for MHC), use `ClusterCache.Watch` with `clustercache.NewWatcher(...)`; the watch is torn down and re-established with the accessor.
- KCP additionally needs etcd and the kube-apiserver inside the workload cluster: `controlplane/kubeadm/pkg/proxy` tunnels through the apiserver, `pkg/etcd` + `etcd_client_generator.go` build etcd clients, `pkg/workload_cluster*.go` holds all workload-side operations (CoreDNS/kubeadm-config/RBAC upgrades).

## Operational notes

- Change an API type → `make generate` (runs `generate-modules`, `generate-manifests`, `generate-go-deepcopy`, `generate-go-conversions`, `generate-go-openapi`, `generate-crd-docs`). Modules list: `Makefile:290` `ALL_GENERATE_MODULES = core kubeadm-bootstrap kubeadm-control-plane docker-infrastructure test-extension`.
- CRD/RBAC/webhook YAML is generated per provider into `<provider>/config/{crd/bases,rbac,webhook}` from markers in `api/...` and `<provider>/{reconcilers,webhooks}` (`Makefile:299`). Never hand-edit those files.
- Adding a `+kubebuilder:rbac` marker anywhere under a provider requires regenerating that provider's `config/rbac`.
- Local dev loop is Tilt (`Tiltfile`, `make tilt-up`); enabled providers are configured in `tilt-settings.yaml`.

## Where to read more

- `docs/proposals/20260701-code-organization.md` — module boundaries, guarantees per module, the target directory layout. The single most useful doc for this topic.
- `docs/book/src/developer/core/overview.md`, `docs/book/src/developer/core/controllers/overview.md` — core controllers tour.
- `docs/book/src/developer/core/repository-layout.md` — **stale**, useful only for history.
- `docs/book/src/developer/core/{tilt.md,tuning.md,logging.md,testing.md,e2e.md,support-multiple-instances.md}` — dev loop, scale tuning flags, log conventions.
- `docs/book/src/developer/providers/overview.md` and `docs/book/src/developer/providers/contracts/overview.md` (+ `infra-cluster.md`, `infra-machine.md`, `bootstrap-config.md`, `control-plane.md`, `ipam.md`) — the contracts `internal/contract/` encodes.
- `docs/book/src/developer/providers/migrations/v1.14-to-v1.15.md` — the current release's breaking-change list; check it before assuming an import path.
- `docs/book/src/reference/{glossary.md,ports.md,versions.md}` — flavor terminology, manager ports.
- `docs/proposals/20220221-runtime-SDK.md`, `20210526-cluster-class-and-managed-topologies.md`, `20191017-kubeadm-based-control-plane.md`, `20240916-improve-status-in-CAPI-resources.md` — the four proposals that most shape current code.

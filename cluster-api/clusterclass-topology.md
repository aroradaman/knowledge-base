# ClusterClass and Managed Topologies

Operational map of the `cluster.spec.topology` feature: the ClusterClass/Cluster API types, the topology reconciler's three-stage pipeline (current state -> desired state -> reconcile via server-side apply), the variables/patch engine, and the upgrade sequencing rules. This subsystem is architecturally different from the rest of CAPI: instead of one controller per object owning a narrow slice of spec, a single controller computes a *whole-cluster desired state* in memory and then applies every child object with SSA under one field manager. Bugs here are usually "desired state was computed wrong" or "SSA/dry-run decided there was (no) diff", not "reconcile loop missed an event".

Feature gates: `ClusterTopology` (required, alpha, `feature/feature.go:41`), `RuntimeSDK` (optional; enables lifecycle hooks + external patches), `ClusterClassInlineTemplates` (allows `template:` instead of `templateRef:` in a ClusterClass). Defaults are all `false` — see `feature/feature.go:108-113`.

## Mental model

- `ClusterClass` is a *template of templates*: it holds refs to provider template objects (InfraClusterTemplate, ControlPlaneTemplate, InfraMachineTemplate, BootstrapConfigTemplate) plus variable schemas and patches.
- `Cluster.spec.topology` is the *instantiation*: class ref, version, replica counts, metadata, MD/MP topologies, variable values.
- The topology controller owns every generated object (InfraCluster, ControlPlane, its InfraMachineTemplate, MachineDeployments + their templates, MachinePools, MachineHealthChecks) and stamps them with `topology.cluster.x-k8s.io/owned` (`api/core/v1beta2/common_types.go:33`).
- Everything else in CAPI still runs normally afterwards: the topology controller only produces MachineDeployments/ControlPlane objects; the regular controllers reconcile them.

## Types

| Type | File |
|---|---|
| `ClusterClass`, `ClusterClassSpec` | `api/core/v1beta2/clusterclass_types.go:78,94` |
| `InfrastructureClass`, `ControlPlaneClass`, `WorkersClass` | same file, `:158,174,426` |
| `MachineDeploymentClass`, `MachinePoolClass` | same file, `:448,780` |
| `ClusterClassVariable`, `VariableSchema`, `JSONSchemaProps`, `ValidationRule` | same file, `:895,943,957,1185` |
| `ClusterClassPatch`, `PatchDefinition`, `PatchSelector`, `JSONPatch`, `ExternalPatchDefinition` | same file, `:1303,1359,1379,1452,1507` |
| `ClusterClassUpgrade`/`…External` (`generateUpgradePlanExtension`), `spec.kubernetesVersions` | same file, `:1342,1350`, spec field at `:153` |
| `ClusterClassStatus`, `ClusterClassStatusVariable` | same file, `:1683,1728` |
| `Topology`, `ClusterClassRef`, `ControlPlaneTopology`, `WorkersTopology`, `MachineDeploymentTopology`, `MachinePoolTopology` | `api/core/v1beta2/cluster_types.go:558,595,620,859,879,1226` |
| `Cluster.spec.topology` field + `Topology.IsDefined()` | `api/core/v1beta2/cluster_types.go:508,590` |

Notes:
- Templates are referenced by `templateRef` (`ClusterClassTemplateReference`, contract-versioned GK+name) **or** inlined via `template` (`ClusterClassTemplate`) when `ClusterClassInlineTemplates` is on; CEL enforces exactly one (`clusterclass_types.go:157`).
- Conditions: `ClusterClassVariablesReadyCondition` / `ClusterClassRefVersionsUpToDateCondition` (`clusterclass_types.go:37,51`); on the Cluster, `ClusterTopologyReconciledCondition` with reasons at `cluster_types.go:74-139`.

## Controllers and wiring

Registered in `core/main.go:562-604` behind `ClusterTopology`:

- `core/reconcilers/clusterclass` — reconciles ClusterClass: resolves template refs to current contract versions (`reconcileExternalReferences`), and expands `status.variables` from inline variables + `DiscoverVariables` runtime extensions (`clusterclass_controller.go:281 reconcileVariables`). Sets `observedGeneration`.
- `core/reconcilers/topology/cluster` — the main topology reconciler (`Reconciler`, `cluster_controller.go:79`).
- `core/reconcilers/topology/machinedeployment`, `.../machineset` — garbage-collect the provider templates referenced by topology-owned MDs/MSs on deletion (see their `doc.go`). They are deliberately separate today; comments in `core/main.go:588,599` say they will merge into the main MD/MS reconcilers.

## The pipeline (`core/reconcilers/topology/cluster`)

`Reconcile` (`cluster_controller.go:267`) → returns early unless `cluster.Spec.Topology.IsDefined()`; builds `scope.New(cluster)`; defers `reconcileStatus` + patch of the Cluster. `reconcile` (`cluster_controller.go:334`) runs in this fixed order:

1. **Get ClusterClass and gate on it.** Errors out unless `VariablesReady=True` and `clusterClass.Generation == status.ObservedGeneration` (`:349-355`). A stale ClusterClass therefore blocks the whole topology — first thing to check when nothing happens.
2. **Default + validate Cluster variables** against the ClusterClass, reusing the webhook code: `coreadmission.Cluster{}.DefaultAndValidateVariables` (`core/webhooks/admission/cluster.go:773`). Needed because the webhook may have run before the ClusterClass existed.
3. **Blueprint** — `getBlueprint` (`blueprint.go:32`) reads all templates into `scope.ClusterBlueprint` (`exp/topology/scope/blueprint.go`), aligning refs to the current contract apiVersion.
4. **Current state** — `getCurrentState` (`current_state.go:39`) reads InfraCluster, ControlPlane (+ its InfraMachineTemplate, MHC), MDs, MPs into `scope.ClusterState` (`exp/topology/scope/state.go`). `alignRefAPIVersion` (`current_state.go:426`) normalizes apiVersions so SSA does not see spurious diffs.
5. **`BeforeClusterCreate` hook** (RuntimeSDK only) — `callBeforeClusterCreateHook` (`cluster_controller.go:447`); blocking response returns a `RequeueAfter` before anything is created.
6. **Dynamic watches** — `setupDynamicWatches` (`cluster_controller.go:422`) watches the InfraCluster/ControlPlane GVKs, filtered with `predicates.ResourceIsTopologyOwned` (`util/predicates/generic_predicates.go:281`).
7. **managedFields mitigation** — `migrateClusterAndMitigateManagedFieldsIssue` (`managedfieldsmitigation.go:27`) → `internal/util/ssa`. If it changed anything it requeues after 1s.
8. **Desired state** — `r.desiredStateGenerator.Generate` = `exp/topology/desiredstate.generator.Generate` (`exp/topology/desiredstate/desired_state.go:103`).
9. **Reconcile state** — `reconcileState` (`reconcile_state.go:67`).
10. **Requeue** — `s.HookResponseTracker.AggregateRetryAfter()` (`exp/topology/scope/hookresponsetracker.go:73`); any blocking hook turns into `ctrl.Result{RequeueAfter: …}`.

Deletion: `reconcileDelete` (`cluster_controller.go:556`) calls `BeforeClusterDelete` and sets `runtime.cluster.x-k8s.io/ok-to-delete` once unblocked.

### `scope.Scope` — the whole request state

Everything the pipeline computes is threaded through one struct (`exp/topology/scope/scope.go:26`). When debugging, this is the object to dump.

| Field | Type / file | Holds |
|---|---|---|
| `Blueprint` | `ClusterBlueprint`, `scope/blueprint.go` | ClusterClass + all resolved templates + `Topology`; helpers `HasControlPlaneInfrastructureMachine()`, `HasMachineDeployments()`, `IsControlPlaneMachineHealthCheckEnabled()` |
| `Current` | `ClusterState`, `scope/state.go` | live Cluster, InfraCluster, `ControlPlaneState`, `MachineDeploymentsStateMap`, `MachinePoolsStateMap` (maps keyed by topology name); `.Upgrading(ctx, c)` reports which are mid-upgrade |
| `Desired` | `ClusterState` | the computed intent |
| `UpgradeTracker` | `scope/upgradetracker.go:26` | `ControlPlaneUpgradeTracker` + per-worker `WorkerUpgradeTracker` (upgrading / pendingCreate / pendingUpgrade / deferred); concurrency seeded from the `upgrade-concurrency` annotation in `scope.New` |
| `HookResponseTracker` | `scope/hookresponsetracker.go:30` | blocking hook responses → `IsBlocking(hook)`, `AggregateRetryAfter()`, `AggregateMessage()` |

`status.go:85 reconcileTopologyReconciledCondition` turns the trackers into the `TopologyReconciled` condition — reading that condition message tells you exactly which tracker flag stopped progress.

### Desired state computation (`exp/topology/desiredstate/desired_state.go`)

Order inside `Generate`: InfraCluster → CP InfraMachineTemplate → **upgrade plan** (`ComputeUpgradePlan`, `upgrade_plan.go:48`) → mark currently-upgrading MDs/MPs on `UpgradeTracker` → ControlPlane (`computeControlPlane`, incl. `computeControlPlaneVersion:548`) → CP MachineHealthCheck → Cluster → MachineDeployments (`computeMachineDeployment:845`, `computeMachineDeploymentVersion:1118`) → MachinePools → **patch engine last** (`:229`).

Because patches run last, anything the compute step set explicitly can be clobbered by a badly written patch — see the warning comment at `desired_state.go:225-228`.

Upgrade plan sources, in precedence order (`desired_state.go:126-131`): `GetUpgradePlanFromExtension` (ClusterClass `spec.upgrade.external.generateUpgradePlanExtension`) → `GetUpgradePlanFromClusterClassVersions` (`spec.kubernetesVersions`) → `GetUpgradePlanOneMinor` (default: one minor at a time).

### Reconcile state (`reconcile_state.go:67`)

Fixed order: cluster shim → `callAfterHooks` → InfraCluster → ControlPlane → Cluster → MachineDeployments → MachinePools.

- **Cluster shim** (`reconcileClusterShim:120`, `internal/topology/clustershim/clustershim.go:31`): a `<cluster>-shim` Secret used as a temporary extra owner of InfraCluster/ControlPlane so they are GC'd if the process dies between creating them and writing the refs onto the Cluster. Deleted once the Cluster itself owns both.
- **MD/MP diffing is by topology name, not by object name**: `calculateMachineDeploymentDiff` / `calculateMachinePoolDiff` (`:1106,1128`) key on the MD/MP *topology* name and produce `toCreate/toUpdate/toDelete`. Before creating, `getCurrentMachineDeployments` (`:591`) re-lists via `r.APIReader` (live, uncached, label-selected on `cluster.x-k8s.io/cluster-name` + `topology.cluster.x-k8s.io/owned`) so a stale cache cannot cause duplicate MachineDeployments. Same pattern for MachinePools (`:906`).
- `reconcileReferencedObject` (`:1162`) for InfraCluster/ControlPlane — in-place SSA, with `contract.…IgnorePaths()` so fields owned by other actors (e.g. autoscaler-managed replicas) are not fought over (`:300,404`).
- `reconcileReferencedTemplate` (`:1249`) for templates — **template rotation**: if only metadata changed, patch in place; if the spec changed, create a new template with a generated name, repoint the ref, and let the owner roll out. This is the in-place-vs-rollout decision point, driven purely by `patchHelper.HasSpecChanges()`.
- Compatibility gates before any update: `internal/topology/check/compatibility.go` (`ObjectsAreStrictlyCompatible`, `ClusterClassesAreCompatible`, …).
- Naming of generated objects: `internal/topology/names/names.go` (templated name generators honoring `naming.template` in the class).

## Variables and patches

- Definitions live in `ClusterClass.spec.variables` (inline) and/or come from `DiscoverVariables` extensions; both are flattened into `ClusterClass.status.variables` with a `definitionFrom` of `"inline"` (`clusterv1.VariableDefinitionFromInline`) or the patch name (`core/reconcilers/clusterclass/clusterclass_controller.go:281,372`).
- Validation/defaulting of schemas and values: `internal/topology/variables/` — `clusterclass_variable_validation.go` (OpenAPI + CEL, uses apiextensions machinery), `cluster_variable_validation.go`, `cluster_variable_defaulting.go`, `schema.go`.
- Patch engine: `core/reconcilers/topology/cluster/patches/engine.go:72 (engine.Apply)`.
  - Builds one `GeneratePatchesRequest` containing every template (`createRequest:274`), then loops over `spec.patches` **in order**, accumulating results onto the same request (`applyPatchesToRequest:493`).
  - Generator selection: `createPatchGenerator:472` → `patches/inline.NewGenerator` (RFC6902/JSON-merge from `JSONPatch` entries, Go-template `valueFrom.template`, `enabledIf`) or `patches/external.NewGenerator` (`GeneratePatches` runtime hook). `api/interface.go` defines `Generator`/`Validator`.
  - After all patches: `ValidateTopology` external validators (`patches/external/external_validator.go`), then `updateDesiredState:593` copies patched template specs back onto the desired objects.
- Builtin variables: name `builtin` (`api/runtime/hooks/v1alpha1/topologymutation_variable_types.go:24`), struct `Builtins` (`:27`). Resolved per template scope by `patches/variables/variables.go`: `Global`, `ControlPlane`, `MachineDeployment`, `MachinePool`. Only variables whose definition applies to the current patch are included (`definitionsForPatch`, `engine.go:702`).

## Server-side apply and ownership

- Field manager: `capi-topology` — `structuredmerge.TopologyManagerName` (`.../structuredmerge/serversidepatchhelper.go:32`). All writes use `client.FieldOwner(TopologyManagerName)` + `client.ForceOwnership` (`:151-156`), so the topology controller deliberately steals contested fields.
- Why SSA: the controller only ever declares *its* intent (a filtered subset of the object); other managers keep their fields. `ssa.FilterObject` with `AllowedPaths`/`IgnorePaths` (`internal/util/ssa/filterintent.go`) trims the intent.
- **Dry-run then diff**: `dryRunSSAPatch` (`structuredmerge/dryrun.go:53`) SSA-dry-runs both the modified *and* the original object (the latter to apply current defaulting so defaulting drift does not look like a change), then diffs to produce `hasChanges` / `hasSpecChanges`. Results are memoized in `ssa.Cache` keyed by `ComputeRequestIdentifier(resourceVersion, modified)` (`internal/util/ssa/cache.go:111`); cache instance created at `cluster_controller.go:171` as `ssa.NewCache("topology/cluster")`.
- During dry-run the objects carry `topology.cluster.x-k8s.io/dry-run` (`clusterv1.TopologyDryRunAnnotation`, `common_types.go:179`) so provider webhooks skip immutability checks — see `util/topology/topology.go:29 IsDryRunRequest` and its consumers in `bootstrap/kubeadm/webhooks/admission/kubeadmconfigtemplate.go:58` and `test/infrastructure/docker/webhooks/admission/devmachinetemplate.go:77`. The annotation and the resulting `capi-topology` managedFields entries are stripped again by `cleanupManagedFieldsAndAnnotation` (`dryrun.go:224`).
- `drop_diff.go` drops diffs for fields the topology controller does not own, so a non-owned change never triggers a rollout.
- Labels/annotations that mark topology objects (`api/core/v1beta2/common_types.go`): `topology.cluster.x-k8s.io/owned` (33), `…/deployment-name` (37), `…/pool-name` (69), `…/hold-upgrade-sequence` (52), `…/defer-upgrade` (61), `…/upgrade-concurrency` (65), `unsafe.topology.cluster.x-k8s.io/disable-update-class-name-check` (73), `…/disable-update-version-check` (77).

## Rollouts, upgrades, rebase

- A ClusterClass edit bumps its `generation`; the ClusterClass controller re-resolves refs/variables and bumps `observedGeneration`; `clusterClassToCluster` (`cluster_controller.go:492`) enqueues every Cluster using it. Propagation then happens purely through recomputation of desired state.
- Version sequencing lives in `computeControlPlaneVersion` (`desired_state.go:548`) and `computeMachineDeploymentVersion` (`:1118`):
  - CP does not pick up a new version while provisioning or upgrading; while upgrading it re-marks pending hooks.
  - MDs/MPs only move once `s.UpgradeTracker.ControlPlane.IsControlPlaneStable()` and no `AfterControlPlaneUpgrade`/`BeforeWorkersUpgrade` hook is blocking (`:1127,1467`).
  - Worker concurrency is bounded by `WorkerUpgradeTracker` (`exp/topology/scope/upgradetracker.go:89`), default 1, raised via the `upgrade-concurrency` annotation; `hold-upgrade-sequence` / `defer-upgrade` feed `isMachineDeploymentDeferred` (`:1207`) and `isMachinePoolDeferred` (`:1547`).
  - Chained upgrades: multi-step plans from `ComputeUpgradePlan`; surfaced on the Cluster by `status.go:46 reconcileUpgradePlan`.
- **Rebase** = changing `spec.topology.classRef`. Allowed; the Cluster webhook fetches the old ClusterClass and runs `check.ClusterClassesAreCompatible` (`core/webhooks/admission/cluster.go:407-425`). The old class need not be reconciled, so you can rebase away from a broken ClusterClass.
- Webhooks: `core/webhooks/admission/clusterclass.go` (`validate:91` — variables, patches via `patch_validation.go`, naming strategies, MHC classes, `kubernetesVersions` gap checks, refuses removing classes still in use) and `core/webhooks/admission/cluster.go` (`validateTopology:236` — feature gate, version semantics/ceiling, `DefaultAndValidateVariables:773`, blocks version change while `AfterClusterUpgrade` is pending at `:463`).

## "I need to change X" → file

| Change | Edit |
|---|---|
| New field on ClusterClass or `spec.topology` | `api/core/v1beta2/clusterclass_types.go` / `cluster_types.go`, then `make generate`; wire into blueprint (`blueprint.go`) and compute (`exp/topology/desiredstate/desired_state.go`); add webhook validation in `core/webhooks/admission/` |
| New builtin variable | `api/runtime/hooks/v1alpha1/topologymutation_variable_types.go` + the matching func in `core/reconcilers/topology/cluster/patches/variables/variables.go` |
| Change which fields cause a rollout | `structuredmerge/options.go` (`IgnorePaths`), `contract.…IgnorePaths()`, `structuredmerge/drop_diff.go` |
| Change upgrade ordering/gating | `exp/topology/desiredstate/desired_state.go` (`computeControlPlaneVersion`, `computeMachineDeploymentVersion`) + `exp/topology/scope/upgradetracker.go` |
| Change the order objects are written | `core/reconcilers/topology/cluster/reconcile_state.go:67` |
| Change generated object names | `internal/topology/names/names.go` + the `naming` structs in `clusterclass_types.go` |
| Add/adjust a lifecycle hook call | `exp/topology/desiredstate/lifecycle_hooks.go` (upgrade hooks) or `reconcile_state.go:178 callAfterHooks` / `cluster_controller.go:447` — see `.claude/concepts/runtime-sdk.md` |

## Gotchas

- **Stale-cache protection is layered and easy to break**: the controller is built with `capicontrollerutil.NewControllerManagedBy` (`cluster_controller.go:114`) whose *consistency store* (`util/controller/consistency.go`) makes the reconcile wait until the informer cache has caught up with what this controller wrote; it is cleared on Cluster not-found and on deletion (`:272,325`). Separately, MD/MP creation double-checks via `r.APIReader`. If you add a new child object kind, consider both.
- **Nothing reconciles**: check `ClusterClass` `VariablesReady` + `observedGeneration`, and the `TopologyReconciled` condition reason on the Cluster.
- **Unexpected machine rollout**: something made `HasSpecChanges()` true — often a patch that rewrites a field the compute step set, a provider defaulting change, or an apiVersion mismatch not normalized by `alignRefAPIVersion`.
- **No rollout when expected**: the `ssa.Cache` short-circuits dry-run when `(resourceVersion, modified-hash)` repeats; also `drop_diff`/`IgnorePaths` may be dropping your field.
- **Dry-run side effects**: provider validating webhooks must honor `TopologyDryRunAnnotation` or they reject the dry-run apply; symptoms are immutability errors that never reach a real write.
- Patch order matters (`spec.patches` is applied in array order, accumulating).
- Event filtering is aggressive: `clusterChangeIsRelevant` / `machineDeploymentChangeIsRelevant` (`cluster_controller.go:175,223`) drop events that cannot affect the topology. A change you expect to trigger reconcile may be filtered out — check these predicates before assuming a controller bug.
- The `unsafe.topology.cluster.x-k8s.io/*` annotations are escape hatches used by tests and recovery procedures (adopting a Cluster into a topology, skipping version validation). Treat their presence in a bug report as significant.
- Tests: envtest suites in `core/reconcilers/topology/cluster/suite_test.go` (+ `cluster_controller_test.go`, `current_state_test.go`, `reconcile_state_test.go`, `blueprint_test.go`, `status_test.go`), SSA behavior in `.../structuredmerge/serversidepatchhelper_test.go` and `dryrun_test.go`, patches in `.../patches/engine_test.go`, desired state in `exp/topology/desiredstate/*_test.go`, variables in `internal/topology/variables/*_test.go`. E2E: `test/e2e/clusterclass_changes.go`, `test/e2e/clusterclass_rollout.go`, `test/e2e/cluster_upgrade_runtimesdk.go`.

## Where to read more

- `docs/book/src/tasks/experimental-features/cluster-class/index.md`, `write-clusterclass.md`, `operate-cluster.md`, `change-clusterclass.md`
- `docs/proposals/20210526-cluster-class-and-managed-topologies.md`
- `docs/proposals/20220330-topology-mutation-hook.md`
- `docs/proposals/20250513-chained-and-efficient-upgrades-for-clusters-with-managed-topologies.md`
- `docs/proposals/20260706-worker-version-pinning.md`
- `docs/proposals/20221003-In-place-propagation-of-Kubernetes-objects-only-changes.md`

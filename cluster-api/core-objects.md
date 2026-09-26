# Core CAPI Objects and Their Controllers

Operational map of the core object graph (Cluster, MachineDeployment, MachineSet, Machine, MachineHealthCheck) and the five reconcilers that drive it in this repo. Everything below is specific to this tree at `main`: reconcilers live under `core/reconcilers/<kind>/` (NOT `internal/controllers/` — that layout was moved by `docs/proposals/20260701-code-organization.md`), the served API is `api/core/v1beta2` (imported everywhere as `clusterv1`), and `api/core/v1beta1` exists only for conversion. Assume you are here to change behavior, so the emphasis is on phase ordering, deletion ordering, condition ownership, and what breaks when you reorder things.

## Object graph

```
Cluster (api/core/v1beta2/cluster_types.go)
 ├─ spec.infrastructureRef ──────────> InfraCluster        (provider, e.g. DockerCluster)
 ├─ spec.controlPlaneRef ───────────-> ControlPlane        (provider, e.g. KubeadmControlPlane)
 │                                      └─ creates Machines directly (control-plane Machines)
 ├─ ownerRef-> MachineDeployment (0..n)
 │               └─ ownerRef + label machine-template-hash -> MachineSet (1..n; 1 current + n old)
 │                        └─ ownerRef + labels cluster.x-k8s.io/{set-name,deployment-name} -> Machine
 │                                 ├─ spec.infrastructureRef ──> InfraMachine    (provider)
 │                                 ├─ spec.bootstrap.configRef -> BootstrapConfig (provider)
 │                                 ├─ spec.bootstrap.dataSecretName -> Secret (cloud-init/ignition)
 │                                 └─ status.nodeRef ─────────> Node (workload cluster, matched by providerID)
 ├─ ownerRef-> MachineSet / Machine (stand-alone, adopted directly by the Cluster)
 └─ ownerRef-> MachineHealthCheck (selects Machines by label; writes conditions onto Machines)
```

Edge facts a file listing will not tell you:

| Edge | Who creates it | Link field |
|---|---|---|
| Cluster -> InfraCluster / ControlPlane | user / topology controller | `spec.infrastructureRef`, `spec.controlPlaneRef` are `ContractVersionedObjectReference` (group+kind+name, **no apiVersion**); resolved via `controllers/external` + `internal/contract` |
| Cluster -> descendants (ownerRef + `cluster.x-k8s.io/cluster-name` label) | each descendant's own controller | `core/reconcilers/cluster/cluster_controller_phases.go:ensureOwnerRefAndLabel`, `core/reconcilers/machine/machine_controller.go:reconcileMachineOwnerAndLabels` |
| MD -> MS | MachineDeployment controller via SSA | `core/reconcilers/machinedeployment/machinedeployment_rollout_planner.go:computeDesiredMS`; selector gets `clusterv1.MachineDeploymentUniqueLabel` (`machine-template-hash`) |
| MS -> Machine + InfraMachine + BootstrapConfig | MachineSet controller | `core/reconcilers/machineset/machineset_controller.go:createMachines` (`computeDesiredMachine`, `createBootstrapConfig`, `createInfraMachine`) |
| Machine -> Node | Machine controller | `machine.spec.providerID` matched against `node.spec.providerID`, then `status.nodeRef` set |

## Shared reconcile skeleton

All five follow the same shape; learn it once:

1. `Get` object, `IsNotFound` -> return nil.
2. `finalizers.EnsureFinalizer(...)` and **return immediately if it was added** (avoids init/delete race). MHC is the exception — no finalizer.
3. `patch.NewHelper(obj, r.Client)`.
4. `paused.EnsurePausedCondition(ctx, client, cluster, obj)` -> return if paused/requeue.
5. Build a package-local `scope` struct holding cluster + object + resolved externals + `deletingReason`/`deletingMessage`.
6. `defer` -> `r.updateStatus(ctx, s)` then `patch<Kind>(ctx, patchHelper, obj, patchOpts...)`; `patch.WithStatusObservedGeneration{}` is appended **only when `reterr == nil`**.
7. Run an ordered slice of sub-reconcilers through a local `doReconcile`, which aggregates errors and folds results with `util.LowestNonZeroResult`.

`doReconcile` keeps running phases until the first error, then keeps collecting errors but stops folding results. MachineSet's variant additionally supports a `stopReconcile bool` (`machineSetReconcileBlockingFunc`) to break the chain without an error.

## Cluster controller

`core/reconcilers/cluster/cluster_controller.go` (`Reconcile` at :138), phases in `cluster_controller_phases.go`, status in `cluster_controller_status.go`.

- Watches: `Cluster` (For); `Machine` -> `controlPlaneMachineToCluster`; `MachineDeployment`; `MachinePool` (feature-gated); `ClusterCache.GetClusterSource("cluster", ..., WatchForProbeFailure(RemoteConnectionGracePeriod))`. Externals (InfraCluster/ControlPlane) get dynamic watches via `external.ObjectTracker`.
- `alwaysReconcile`: `reconcileInfrastructure` -> `reconcileControlPlane` -> `getDescendants`.
- Normal adds: `reconcileKubeconfig` -> `reconcileV1Beta1ControlPlaneInitialized` (only for clusters **without** `controlPlaneRef`).
- Topology clusters return early with "Waiting for the topology to be generated" until both refs are set.
- `reconcileInfrastructure` pulls `spec.controlPlaneEndpoint` and `status.failureDomains` off the InfraCluster and sets `status.initialization.infrastructureProvisioned`; when `infrastructureRef` is unset it marks provisioned=true to unblock the rest.
- Deletion order (`reconcileDelete`, :332): BeforeClusterDelete hook gate (`internal/hooks.IsOkToDelete`, RuntimeSDK+ClusterTopology gates) -> delete directly-owned descendants (`filterOwnedDescendants`, control-plane Machines sorted last) -> requeue every `deleteRequeueAfter` (5s) while `objectsPendingDeleteCount > 0` -> delete ControlPlane -> delete InfraCluster -> remove `clusterv1.ClusterFinalizer`. Each stage sets `s.deletingReason`/`Message`, surfaced by `setDeletingCondition`.

## Machine controller

`core/reconcilers/machine/machine_controller.go` (`Reconcile` at :189).

- Watches: `Machine`; `Cluster` (with `predicates.ClusterControlPlaneInitialized`); `ClusterCache` source; `MachineSet`; `MachineDeployment`. Workload-cluster Nodes get a watch lazily via `watchClusterNodes` inside `reconcileNode`.
- Scope preloads owning MachineSet (`getOwnerMachineSet`, from ownerRefs) and owning MachineDeployment (`getOwnerMachineDeployment`, from the MS's `cluster.x-k8s.io/deployment-name` label).
- `alwaysReconcile`: `reconcileMachineOwnerAndLabels` -> `reconcileBootstrap` -> `reconcileInfrastructure` -> `reconcileNode` -> `reconcileCertificateExpiry`. Normal appends `reconcileInPlaceUpdate`; delete appends `reconcileDelete`. Note the delete path **still runs the external/node phases first** (they tolerate NotFound while deleting).

### Bootstrap data flow
`machine_controller_phases.go:reconcileBootstrap` — if `spec.bootstrap.configRef` is unset, no-op (user-supplied secret). Otherwise read the BootstrapConfig via `contractapi.BootstrapConfig`, mirror its Ready into `BootstrapReadyV1Beta1Condition`, and when `GetDataSecretCreated()` is true copy `status.dataSecretName` -> `machine.spec.bootstrap.dataSecretName` and set `status.initialization.bootstrapDataSecretCreated`. If `spec.bootstrap.dataSecretName` is already set the phase short-circuits to ready.

### Provider ID and node linking
`reconcileInfrastructure` waits for `provisioned`, then reads `providerID`, `addresses`, `failureDomain`; only then sets `machine.spec.providerID` and `status.initialization.infrastructureProvisioned`. While providerID is empty it calls `r.controller.DeferNextReconcileForObject(...)` instead of requeueing. `machine_controller_noderef.go:reconcileNode` then does `getNode(remoteClient, spec.providerID)`, sets `status.nodeRef` + `status.nodeInfo`, and `patchNode` propagates managed labels/annotations (`util/labels.GetManagedLabels`, `util/annotations.GetManagedAnnotations`, extended by `AdditionalSyncMachineLabels/Annotations`), the `InterruptibleLabel`, Machine taints (`propagateMachineTaintsToNode`) and the outdated-revision taint (`shouldNodeHaveOutdatedTaint`). First successful label reconcile is what removes `NodeUninitializedTaint`.

### Deletion order (`reconcileDelete`, :476) — do not reorder
1. `isDeleteNodeAllowed` — refuses (and only logs) on `errClusterIsBeingDeleted`, `errControlPlaneIsBeingDeleted`, `errNilNodeRef`, `errNoControlPlaneNodes`, `errLastControlPlaneNode`; externally-managed control planes return nil early.
2. pre-drain hooks: any annotation prefixed `pre-drain.delete.hook.machine.cluster.x-k8s.io` blocks, records `status.deletion.waitForPreDrainHookStartTime`.
3. Drain, if `isNodeDrainAllowed` (not `ExcludeNodeDrainingAnnotation`, pre-terminate not started, InfraMachine present and not deleting, `spec.deletion.nodeDrainTimeoutSeconds` not exceeded). `drainNode` (:862) uses `core/reconcilers/machine/drain` + MachineDrainRule filters; retries every `drainRetryInterval` (20s). An intermediate `patchMachine` is issued before draining so progress is visible.
4. Wait for volume detach, if `isNodeVolumeDetachingAllowed` (`ExcludeWaitForNodeVolumeDetachAnnotation`, `spec.deletion.nodeVolumeDetachTimeoutSeconds`); `shouldWaitForNodeVolumes` (:965), retry `waitForVolumeDetachRetryInterval` (20s).
5. pre-terminate hooks: annotations prefixed `pre-terminate.delete.hook.machine.cluster.x-k8s.io`.
6. `reconcileDeleteInfrastructure` — wait for InfraMachine gone.
7. `reconcileDeleteBootstrap` — wait for BootstrapConfig gone.
8. Delete the Node (only now, per issue #2565), polling every 2s up to `r.nodeDeletionRetryTimeout` (10s default); gives up once `spec.deletion.nodeDeletionTimeoutSeconds` has elapsed since `deletionTimestamp`.
9. Remove `clusterv1.MachineFinalizer`.

### In-place update
`machine_controller_inplace_update.go:reconcileInPlaceUpdate` runs only under `feature.InPlaceUpdates`. Gate is the `in-place-updates.internal.cluster.x-k8s.io/update-in-progress` annotation on Machine + a pending `UpdateMachine` runtime hook (`internal/hooks`); it then calls `callUpdateMachineHook` and `completeInPlaceUpdate`. Metadata-only changes are a different mechanism (see MachineSet `syncMachines`).

## MachineSet controller

`core/reconcilers/machineset/machineset_controller.go` (`Reconcile` at :187).

- Watches: `MachineSet` (For), `Owns(Machine)`, `Machine` -> `MachineToMachineSets` (orphan adoption), `MachineDeployment`, `Cluster` (+`predicates.ClusterPausedTransitions`), `ClusterCache` source. Uses `WithDynamicCache(r.DynamicCache)` for InfraMachine/BootstrapConfig reads.
- `alwaysReconcile`: `reconcileMachineSetOwnerAndLabels` -> `reconcileInfrastructure` -> `reconcileBootstrapConfig` -> `getAndAdoptMachinesForMachineSet` (blocking).
- Normal then: `reconcileUnhealthyMachines` -> `syncMachines` (blocking) -> `triggerInPlaceUpdate` -> `syncReplicas`.
- `defer` runs `updateStatus` **and** `reconcileV1Beta1Status`, then `patchMachineSet`, then possibly overrides the result with `shouldRequeueForReplicaCountersRefresh` (15s while `readyReplicas != replicas`; `minReadySeconds` as clock-skew guard).
- `syncMachines` (:642) is in-place propagation: for deleting Machines it merge-patches only teardown-relevant fields (`readinessGates`, `spec.deletion.*`, `minReadySeconds`, `taints`); for live Machines it SSA-applies `computeDesiredMachine` with field manager `capi-machineset` and pushes labels/annotations onto InfraMachine/BootstrapConfig with the separate manager `capi-machineset-metadata` (`updateLabelsAndAnnotations`). `ssa.MitigateManagedFieldsIssue` / `ssa.MigrateManagedFields` handle objects created by CAPI <= v1.11; when mitigation fires the phase returns `RequeueAfter: 1s` and stops the chain.
- `syncReplicas` (:766) first calls `cleanupOrphanedBootstrapConfigsInfraMachines`, then delegates the math to `mdutil.SyncMachinesPlanner` which returns `MachinesToAdd` / `MachinesToMove` / `MachinesToDelete` / `MachineCreationDisabled` / `MachinesPendingAcknowledgeMove`. Scale up is blocked by the `cluster.x-k8s.io/disable-machine-create` annotation (set to `"true"` on old MSs under OnDelete).
- Scale-up runs `runPreflightChecks` (`machineset_preflight.go`): `ControlPlaneIsStable`, `KubernetesVersionSkew`, `KubeadmVersionSkew`, `ControlPlaneVersionSkew`, or `All`; opt-out per-MS via `machineset.cluster.x-k8s.io/skip-preflight-checks`. Failure -> `RequeueAfter: preflightFailedRequeueAfter` + `MachinesCreatedV1Beta1Condition=False`.
- Scale-down order is `machineset_deletion_order.go`, driven by `spec.deletion.order` (v1beta2 rename of `deletePolicy`): `Random` (default), `Newest`, `Oldest` -> `getDeletePriorityFunc`. Priorities: `mustDelete` (already deleting) > `shouldDeleteFirst` (`cluster.x-k8s.io/delete-machine`) > in-place-updating > unhealthy > rest.
- Remediation of unhealthy Machines lives here: `reconcileUnhealthyMachines` (:1617). It deletes stale `MachineOwnerRemediated` conditions from healthy Machines, then remediates Machines matching `collections.IsUnhealthyAndOwnerRemediated`. If the MS is a MachineDeployment child and its `machinedeployment.clusters.x-k8s.io/revision` != the MD's, remediation is refused (`MachineSetMachineCannotBeRemediatedReason`) because the rollout will delete those Machines anyway. `md.spec.remediation.maxInFlight` caps concurrency, decremented for each deleting Machine still carrying `MachineSetMachineRemediationMachineDeletingReason`.
- `reconcileDelete` (:561): delete every owned Machine, wait, then remove `clusterv1.MachineSetFinalizer` once the list is empty.

## MachineDeployment controller

`core/reconcilers/machinedeployment/machinedeployment_controller.go` (`Reconcile` at :145).

- Watches: `MachineDeployment` (For), `Owns(MachineSet)`, `MachineSet` -> `MachineSetToDeployments`, `Cluster`. It lists Machines itself by `spec.selector` before the defer block, because rollout decisions now need Machine-level state.
- `reconcile` (:295): ensure labels/ownerRef -> `getTemplatesAndSetOwner` -> `getAndAdoptMachineSetsForDeployment` -> `ssa.MitigateManagedFieldsIssue` per MS (returns early if mitigated) -> backfill `cluster.x-k8s.io/deployment-name` on MSs -> dispatch on strategy.
- Strategy dispatch: `spec.paused` -> `sync` (`machinedeployment_sync.go`); `RollingUpdate` -> `rolloutRollingUpdate`; `OnDelete` -> `rolloutOnDelete`; anything else is an error.
- Both rollouts use the same shape: `newRolloutPlanner(...)`, `planner.init(...)`, `plan*`, `createOrUpdateMachineSetsAndSyncMachineDeploymentRevision`, `syncDeploymentStatus`, and `cleanupDeployment` when `mdutil.DeploymentComplete`.
- **How a new MachineSet appears**: `mdutil.FindNewAndOldMachineSets` compares `md.spec.template` to each MS's `spec.template` after `mdutil.MachineTemplateDeepCopyRolloutFields` zeroes the in-place-mutable fields (labels, annotations, `minReadySeconds`, `readinessGates`, `spec.deletion.*`, `taints`, `providerID`, `clusterName`). So only `version`, `bootstrap`, `infrastructureRef` and `failureDomain` (plus an expired `spec.rollout.after`) trigger a rollout; everything else is propagated in place. If no match, `computeDesiredMS` builds one with a fresh `machine-template-hash`.
- Revisions: `mdutil.ComputeRevisionAnnotations` sets `machinedeployment.clusters.x-k8s.io/revision` = max(old revisions)+1 (never decreasing) and appends the previous value to the (unexported) revision-history annotation; `GetRevisionAnnotations` carries them over unchanged on old MSs. The MD's own revision annotation is refreshed at the end of `createOrUpdateMachineSetsAndSyncMachineDeploymentRevision`.
- RollingUpdate planning (`machinedeployment_rollout_rollingupdate.go:planRollingUpdate`): `reconcileReplicasPendingAcknowledgeMove` -> `reconcileNewMachineSet` (scale up, bounded by `mdutil.NewMSNewReplicas` / `MaxSurge`) -> `reconcileOldMachineSetsRollingUpdate` (scale down, bounded by `MaxUnavailable`) -> `reconcileInPlaceUpdateIntent` -> `reconcileDeadlockBreaker`. The planner only computes `scaleIntents`; nothing is written until `createOrUpdateMachineSetsAndSyncMachineDeploymentRevision` SSA-applies with field manager `capi-machinedeployment`.
- OnDelete (`machinedeployment_rollout_ondelete.go`): old MSs get `disable-machine-create: "true"`; `reconcileOldMachineSetsOnDelete` only shrinks `spec.replicas` to match Machines that users deleted.
- `reconcileDelete` (:477): delete owned MachineSets, wait, remove `clusterv1.MachineDeploymentFinalizer`.

## MachineHealthCheck controller

`core/reconcilers/machinehealthcheck/machinehealthcheck_controller.go` (`Reconcile` at :177), target logic in `machinehealthcheck_targets.go`.

- Watches: `MachineHealthCheck` (For); `Machine` with `machineIsChangedPredicate()` (only nodeRef name, labels, annotations, `status.conditions` changes); `Cluster`; `ClusterCache` source. `WithRateLimitInterval(15s)` is deliberate — this reconciler is hot under load.
- No finalizer, no ownerRef-driven cleanup; it only writes conditions onto Machines (and optional external remediation requests).
- Unhealthy determination — `needsRemediation` (targets.go:80), first match wins:
  1. `cluster.x-k8s.io/remediate-machine` annotation -> immediately unhealthy.
  2. Skips everything if `ClusterControlPlaneInitialized` is false (except control-plane Machines) or `ClusterInfrastructureReady` is false.
  3. `machineChecks` over `spec.checks.unhealthyMachineConditions` (condition type/status held longer than `timeoutSeconds`).
  4. `nodeChecks`: nodeRef set but Node missing -> `MachineHealthCheckNodeDeletedReason`; Node not yet created -> compare against the latest of Machine creation, `ClusterControlPlaneInitialized`, `ClusterInfrastructureReady`, `MachineInfrastructureReady` transition times against `spec.checks.nodeStartupTimeoutSeconds` (default `clusterv1.DefaultNodeStartupTimeoutSeconds`, `disabledNodeStartupTimeout` turns it off); else `spec.checks.unhealthyNodeConditions`.
  Result is written as `MachineHealthCheckSucceededCondition` (+ the v1beta1 mirror) with a multi-line "Health check failed:" message.
- Short circuit: `isAllowedRemediation` (:733) uses `spec.remediation.triggerIf.unhealthyLessThanOrEqualTo` (the v1beta2 name for `maxUnhealthy`) or `unhealthyInRange` (`[min-max]`). When not allowed it sets `status.remediationsAllowed = 0`, `MachineHealthCheckRemediationAllowedCondition=False` / `TooManyUnhealthy`, emits `EventRemediationRestricted`, patches only the health-check condition onto all targets, and returns — no remediation for anyone.
- Remediation flow: when `spec.remediation.templateRef` is set, clone it into a per-Machine external remediation request named after the Machine and set `MachineExternallyRemediatedCondition`. Otherwise set `MachineOwnerRemediatedCondition=False/WaitingForRemediation` and let the owner act — MachineSet (`reconcileUnhealthyMachines`) for workers, `controlplane/kubeadm/reconcilers/kubeadmcontrolplane/remediation.go` for KCP Machines. MHC only (re)creates the condition when it is absent or True; an in-progress remediation belongs to the owner. `shouldSkipRemediation` honours `PausedAnnotation` and `MachineSkipRemediationAnnotation`.
- Requeue: `minDuration(nextCheckTimes)` so the next condition timeout is re-evaluated exactly when it expires.

## Status machinery: phase vs conditions

- `status.phase` is a **UI-only** summary. `api/core/v1beta2/machine_phase_types.go` and `cluster_phase_types.go` both say controllers must not branch on it. Setters: `core/reconcilers/machine/machine_controller_status.go:setMachinePhaseAndLastUpdated`, `core/reconcilers/cluster/cluster_controller_status.go:setPhase`, `core/reconcilers/machinedeployment/machinedeployment_status.go:setPhase`. The `Failed` phase is deprecated and no longer set.
- Two condition sets coexist:
  - **v1beta2** (`[]metav1.Condition` in `status.conditions`) — types defined per kind in `api/core/v1beta2/<kind>_types.go`, shared names in `condition_consts.go`, helpers in `util/conditions` (`Set`, `Get`, `IsTrue`, `Delete`, `NewSummaryCondition`, `SetAggregateCondition`, `SetMirrorCondition`, custom `MergeStrategy`).
  - **v1beta1 legacy** (`[]clusterv1.Condition` under `status.deprecated.v1beta1.conditions`) — types/reasons in `api/core/v1beta2/v1beta1_condition_consts.go`, helpers in `util/conditions/deprecated/v1beta1` (imported as `v1beta1conditions`: `MarkTrue`, `MarkFalse`, `SetMirror`, `SetSummary`, `WithFallbackValue`). Still maintained in every reconciler; if you add a condition, decide explicitly whether it needs a v1beta1 twin.
- Condition computation is centralized in the `*_status.go` files (`updateStatus`), never scattered through the phases. The phases set `s.deletingReason` / `s.updatingReason` and let `setDeletingCondition` / `setUpdatingCondition` render them. Replica counters (`setReplicas`, `setControlPlaneReplicas`, `setWorkersReplicas`) are also computed there.

### v1beta2 condition types and their authors

| Object | Condition types (`api/core/v1beta2/<kind>_types.go`) | Author |
|---|---|---|
| Cluster | `Available`, `InfrastructureReady`, `ControlPlaneInitialized`, `ControlPlaneAvailable`, `ControlPlaneMachines{Ready,UpToDate}`, `WorkersAvailable`, `WorkerMachines{Ready,UpToDate}`, `RemoteConnectionProbe`, `RollingOut`, `ScalingUp`, `ScalingDown`, `Remediating`, `Deleting`, `Paused` | Cluster controller (`cluster_controller_status.go`) |
| Cluster | `TopologyReconciled` | topology controller (`core/reconcilers/topology/cluster`) |
| Machine | `Available`, `Ready`, `BootstrapConfigReady`, `InfrastructureReady`, `NodeReady`, `NodeHealthy`, `Deleting`, `Updating`, `Paused` | Machine controller (`machine_controller_status.go`) |
| Machine | `UpToDate` | Machine controller for MD-owned workers, KCP for control-plane Machines |
| Machine | `HealthCheckSucceeded` | MHC controller |
| Machine | `OwnerRemediated` | MHC creates it; MachineSet / KCP complete it |
| Machine | `ExternallyRemediated` | MHC + external remediation provider |
| MachineSet | `ScalingUp`, `ScalingDown`, `MachinesReady`, `MachinesUpToDate`, `Remediating`, `Deleting`, `Paused` | MachineSet controller (`machineset_controller_status.go`) |
| MachineDeployment | `Available`, `MachinesReady`, `MachinesUpToDate`, `RollingOut`, `ScalingUp`, `ScalingDown`, `Remediating`, `Deleting`, `Paused` | MachineDeployment controller (`machinedeployment_status.go`) |
| MachineHealthCheck | `RemediationAllowed`, `Paused` | MHC controller |

Rolled-up conditions are computed with `util/conditions.SetAggregateCondition` / `NewSummaryCondition` over child objects; both Cluster and Machine install a custom `MergeStrategy` (`clusterConditionCustomMergeStrategy`, `machineConditionCustomMergeStrategy`) to control precedence in the summary message.

### Labels and annotations you will touch

| Key | Const | Meaning |
|---|---|---|
| `cluster.x-k8s.io/cluster-name` | `clusterv1.ClusterNameLabel` | on every descendant; how `getDescendants` and all `ClusterTo*` mappers find objects |
| `cluster.x-k8s.io/set-name`, `cluster.x-k8s.io/deployment-name` | `MachineSetNameLabel`, `MachineDeploymentNameLabel` | Machine -> MS -> MD back-links (the MD link is a label, not an ownerRef) |
| `machine-template-hash` | `MachineDeploymentUniqueLabel` | per-MachineSet selector value; hash is over rollout-relevant fields only |
| `machinedeployment.clusters.x-k8s.io/revision` | `RevisionAnnotation` | MS revision; MD mirrors the current one |
| `cluster.x-k8s.io/disable-machine-create` | `DisableMachineCreateAnnotation` | blocks MS scale up (set on old MSs under OnDelete) |
| `cluster.x-k8s.io/delete-machine` | `DeleteMachineAnnotation` | user pick for next scale-down |
| `cluster.x-k8s.io/remediate-machine` / `cluster.x-k8s.io/skip-remediation` | `RemediateMachineAnnotation` / `MachineSkipRemediationAnnotation` | force / suppress MHC remediation |
| `machineset.cluster.x-k8s.io/skip-preflight-checks` | `MachineSetSkipPreflightChecksAnnotation` | per-MS preflight opt-out |
| `pre-drain.delete.hook.machine.cluster.x-k8s.io/*`, `pre-terminate.delete.hook.machine.cluster.x-k8s.io/*` | `PreDrainDeleteHookAnnotationPrefix`, `PreTerminateDeleteHookAnnotationPrefix` | external deletion gates; any key with the prefix blocks |
| `machine.cluster.x-k8s.io/exclude-node-draining`, `.../exclude-wait-for-node-volume-detach` | `ExcludeNodeDrainingAnnotation`, `ExcludeWaitForNodeVolumeDetachAnnotation` | skip those deletion stages |
| `in-place-updates.internal.cluster.x-k8s.io/{update-in-progress,pending-acknowledge-move}` | `UpdateInProgressAnnotation`, `PendingAcknowledgeMoveAnnotation` | in-place update handshake between MD, MS and Machine controllers |

### The defer-patch + WithOwnedConditions pattern
Every reconciler ends with a `patch<Kind>` helper that appends `patch.WithOwnedV1Beta1Conditions{...}` and `patch.WithOwnedConditions{...}` before calling `patchHelper.Patch`. See `core/reconcilers/cluster/cluster_controller.go:patchCluster`, `machine_controller.go:patchMachine`, `machineset_controller.go:patchMachineSet`, `machinedeployment_controller.go:patchMachineDeployment`.

Why it matters: `util/patch/patch.go:patchStatusConditions` computes a conditions diff between `before` and `after` and applies it to the **freshly read latest** object (`conditions.Patch.Apply`). For a condition type listed as owned, the apply force-overwrites; for anything else a concurrent change by another controller causes the apply to skip/conflict rather than clobber. So:

- List a condition in `WithOwnedConditions` only if this controller is its sole author.
- Deliberately omitted: `MachineOwnerRemediated` / `MachineExternallyRemediated` in MHC (owned by the remediation owner), and `MachineUpToDateCondition` in `patchMachine` even though `setUpToDateCondition` writes it — KCP writes the same type on control-plane Machines, so the Machine controller stays non-authoritative.
- Never patch the whole object in a phase with a second `patch.Helper` unless you also pass the owned-condition options (the mid-drain `patchMachine` call in `reconcileDelete` is the one sanctioned case).

## Gotchas when changing this code

- **Finalizer-add returns early.** `finalizers.EnsureFinalizer` returning `finalizerAdded=true` exits the reconcile. Do not put work before it and expect it to run in the same pass.
- **Deletion sequences are load-bearing.** Node deletion after infra deletion; drain before volume detach; pre-drain before drain; pre-terminate before infra deletion; control-plane Machines last in `filterOwnedDescendants`. Reordering silently breaks drain semantics and leaks providers' resources.
- **Requeue idioms.** Prefer no requeue when a watch already covers the trigger (comments say so explicitly). Fixed constants in use: `externalReadyWait` 30s, `deleteRequeueAfter` 5s, `drainRetryInterval`/`waitForVolumeDetachRetryInterval` 20s, MS replica refresh 15s. For "wait for our own write to land" use `r.controller.DeferNextReconcileUntilCacheUpToDate(...)` / `DeferNextReconcileForObject(...)` from `util/controller`, not `RequeueAfter`.
- **Never set `RequeueAfter` alongside an error** — `doReconcile` discards results once an error is present.
- **SSA field managers are part of the contract**: `capi-machineset`, `capi-machineset-metadata`, `capi-machinedeployment`. Writing an SSA-owned field with a different manager (or with `client.Update`) causes fights and `ssa.MitigateManagedFieldsIssue` churn.
- **Rollout-vs-in-place**: if you add a field to `MachineSpec`, you must decide whether it belongs in `mdutil.MachineTemplateDeepCopyRolloutFields` (in-place) and, if so, also propagate it in `machineset_controller.go:computeDesiredMachine` / `syncMachines` and in `internal/util/inplace/inplace.go:CleanupMachineSpecForDiff` (the comment on `MachineTemplateDeepCopyRolloutFields` says so).
- **API changes** require `make generate` (deepcopy under `api/.../zz_generated.deepcopy.go`, conversions `api/core/v1beta1/*conversion*`, CRDs under `core/config/crd/bases`, RBAC from the `+kubebuilder:rbac` markers at the top of each controller file) plus `make generate-go-conversions` and `make test-conversions` for round-trip fuzzing.
- **Tests to run**: `make test` (envtest-backed, race detector) for the whole module; targeted `go test ./core/reconcilers/machineset/... ./core/reconcilers/machinedeployment/...`. The rollout planner has golden-file tests (`core/reconcilers/machinedeployment/testdata/`) and `mdutil/rollout_planner_test.go` — rollout math changes usually mean regenerating/reviewing those. Webhook/defaulting changes touch `core/webhooks/admission`.
- **Wiring**: reconcilers are constructed in `core/main.go` (~:619-746) with per-controller `--*-concurrency` flags; new required fields must be set there and validated in `SetupWithManager`.

## Where to read more

- `docs/book/src/developer/core/controllers/cluster.md`, `machine.md`, `machine-set.md`, `machine-deployment.md`, `machine-health-check.md`, `overview.md`
- `docs/book/src/tasks/automated-machine-management/healthchecking.md`, `machine_deletions.md`, `scaling.md`
- `docs/proposals/20240916-improve-status-in-CAPI-resources.md` — the v1beta2 conditions/phase model and why `Failed` went away
- `docs/proposals/20200506-conditions.md` — original conditions + patch-helper design (`WithOwnedConditions` rationale)
- `docs/proposals/20221003-In-place-propagation-of-Kubernetes-objects-only-changes.md` — in-place propagation of metadata-only changes
- `docs/proposals/20200602-machine-deletion-phase-hooks.md` — pre-drain / pre-terminate hook annotations
- `docs/proposals/20240930-machine-drain-rules.md` — MachineDrainRule semantics used by `core/reconcilers/machine/drain`
- `docs/proposals/20191030-machine-health-checking.md`, `docs/proposals/20220927-labels-and-annotations-sync-between-machine-and-nodes.md`
- `docs/proposals/20240807-in-place-updates.md` + `20240807-in-place-updates-implementation-notes.md`, `docs/proposals/20260701-code-organization.md` (why `core/reconcilers/...`)

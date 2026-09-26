# KubeadmControlPlane and KubeadmConfig (the two in-tree providers)

Operational notes for working on `controlplane/kubeadm/` (KCP) and `bootstrap/kubeadm/` (CABPK), plus the provider contracts that core reads. Note the layout moved in the v1.12 dev cycle (see `docs/proposals/20260701-code-organization.md`): there is no longer a `controlplane/kubeadm/internal/` — reusable logic lives in `controlplane/kubeadm/pkg/`, the reconciler in `controlplane/kubeadm/reconcilers/kubeadmcontrolplane/`, webhooks in `controlplane/kubeadm/webhooks/{admission,conversion}/`, cache/client wiring in `controlplane/kubeadm/setup/`. CABPK mirrors this (`bootstrap/kubeadm/pkg/`, `bootstrap/kubeadm/reconcilers/kubeadmconfig/`). API types live outside the provider dirs, in `api/controlplane/kubeadm/v1beta2/` and `api/bootstrap/kubeadm/v1beta2/`.

## KCP file map

| File | What it owns |
| --- | --- |
| `reconcilers/kubeadmcontrolplane/kubeadmcontrolplane_controller.go` | `Reconcile`, phase ordering, cert reconcile, delete, etcd member reconcile, pre-terminate hook, adoption |
| `.../scale.go` | `initializeControlPlane`, `scaleUpControlPlane`, `scaleDownControlPlane`, `selectMachineForInPlaceUpdateOrScaleDown` |
| `.../update.go` | `updateControlPlane` (kubeadm ConfigMap mutation + RBAC), `rollingUpdate` (min/max replica math) |
| `.../preflight.go` | `preflightChecks`, `checkHealthiness`, `handlePreflightCheckResults` |
| `.../remediation.go` | `reconcileUnhealthyMachines`, retry limits, `targetEtcdClusterHealthy` |
| `.../status.go` | `updateStatus`, `updateV1Beta1Status`, all `set*Condition` funcs |
| `.../helpers.go` | `reconcileKubeconfig`, `cloneConfigsAndGenerateMachine`, `createMachine`/`updateMachine` |
| `.../inplace_canupdatemachine.go`, `.../inplace_trigger.go` | InPlaceUpdates feature gate path (Runtime SDK `CanUpdateMachine`/`UpdateMachine`) |
| `pkg/control_plane.go` | `ControlPlane` scope struct — the per-reconcile cache of Machines, InfraMachines, KubeadmConfigs, Nodes, EtcdMembers |
| `pkg/filters.go` | `UpToDate` / `matchesMachineSpec` / `PrepareKubeadmConfigsForDiff` — the rollout decision |
| `pkg/desiredstate/desired_state.go` | `ComputeDesiredMachine`, `ComputeDesiredKubeadmConfig`, `ComputeDesiredInfraMachine` |
| `pkg/cluster.go` | `ManagementCluster` iface, `Management.GetWorkloadCluster`, etcd client TLS setup |
| `pkg/workload_cluster*.go` | `WorkloadCluster` iface + impl: conditions, etcd, coredns, rbac |
| `pkg/etcd/`, `pkg/proxy/` | etcd v3 client; API-server-proxy dialer used to reach etcd static pods |

## KCP reconcile order (`kubeadmcontrolplane_controller.go:reconcile`)

Order is load-bearing; each step assumes the previous one ran.

1. `reconcileExternalReference` — put a Cluster ownerRef on the InfraMachineTemplate (only if `Kind` ends in `Template`).
2. Return early if `cluster.status.initialization.infrastructureProvisioned` is false.
3. `reconcileClusterCertificates` — see *Certificates*.
4. Return early if `cluster.spec.controlPlaneEndpoint` is not valid.
5. `reconcileKubeconfig` (`helpers.go:48`) — create or rotate the `<cluster>-kubeconfig` Secret.
6. `syncMachines` — SSA-propagate in-place-mutable fields onto Machine / InfraMachine / KubeadmConfig. Returns `stopReconcile=true` when `ssa.MitigateManagedFieldsIssue` had to act; caller requeues after 1s.
7. `reconcileControlPlaneAndMachinesConditions` — connects to the workload cluster, sets static-pod + etcd conditions on KCP *and* on each Machine.
8. `reconcileEtcdMembers` — safety net removing etcd members with no Machine/Node.
9. `reconcilePreTerminateHook` — for deleting Machines: forward etcd leadership, remove etcd member, then drop `PreTerminateHookCleanupAnnotation`.
10. `MachinesToCompleteTriggerInPlaceUpdate` — reentrancy for a half-triggered in-place update.
11. `reconcileUnhealthyMachines` — remediation. Returns non-zero result to stop the rest of the loop.
12. `MachinesToCompleteInPlaceUpdate` — block while an in-place update is running.
13. `MachinesNeedingRollout()` non-empty → `updateControlPlane` (rollout takes precedence over plain scaling).
14. Else scale: `numMachines == 0` → `initializeControlPlane`; `< desired` → `scaleUpControlPlane`; `> desired` → `scaleDownControlPlane`.
15. Only when replicas match and nothing needs rollout: `UpdateKubeProxyImageInfo`, `UpdateCoreDNS`, `reconcileCertificateExpiries`.

`Reconcile` itself: owner Cluster lookup → finalizer (`controlplanev1.KubeadmControlPlaneFinalizer`) → `paused.EnsurePausedCondition` → `initControlPlaneScope` (adopts orphan CP machines; errors out on "mixed management mode") → deferred `updateStatus` + `updateV1Beta1Status` + `patchKubeadmControlPlane`.

Requeue constants are in `reconcilers/kubeadmcontrolplane/consts.go`: `deleteRequeueAfter=30s`, `preflightFailedRequeueAfter=15s`, `dependentCertRequeueAfter=30s`. Uninitialized KCP or false `ControlPlaneComponentsHealthy` forces a 20s requeue in the deferred block.

## Rollout / up-to-date decision

`pkg/filters.go:UpToDate` is the single source of truth. A Machine is not up to date if any of:

- `collections.ShouldRolloutBefore(reconciliationTime, kcp.Spec.Rollout.Before)` — certificate expiry from the `MachineCertificatesExpiryDateAnnotation` on the KubeadmConfig.
- `collections.ShouldRolloutAfter(reconciliationTime, kcp.Spec.Rollout.After)` — deadline passed and Machine created before it.
- `matchesMachineSpec` — version, InfraMachine spec vs `ComputeDesiredInfraMachine`, KubeadmConfig spec vs `ComputeDesiredKubeadmConfig`.

`UpToDateResult.EligibleForInPlaceUpdate` is false for `rolloutBefore`/`rolloutAfter`, `DeleteMachineAnnotation`, `RemediateMachineAnnotation`, and for already-up-to-date Machines.

`pkg/filters.go:PrepareKubeadmConfigsForDiff` is where "should this field cause a rollout?" is decided. It converts the init Machine's `InitConfiguration` to a `JoinConfiguration` shape, and deliberately zeroes fields that must never trigger a rollout: `JoinConfiguration.Discovery`, `ClusterConfiguration.DNS`, `ClusterConfiguration.ControlPlaneEndpoint`, `Timeouts.ControlPlaneComponentHealthCheckSeconds`. It then applies `bootstrapadmission.ApplyPreviousKubeadmConfigDefaults` to both sides and `dropOmittableFields` to normalize nil vs empty. **Adding a KubeadmConfigSpec field almost always needs a decision recorded here.**

`rollingUpdate` (`update.go:106`) math: `maxSurge` is validated to be 0 or 1; `minReplicas = desired + maxSurge - 1`, `maxReplicas = desired + maxSurge`. Priorities: (a) if up-to-date replicas already `>= desired`, scale down; (b) if `RemediationInProgressAnnotation` is set, always scale up first; (c) get into `[min,max]`; (d) prefer in-place over scale up/down. `selectMachineForInPlaceUpdateOrScaleDown` picks the eligible subset (delete-annotation outdated → delete-annotation any → outdated-with-unhealthy-components → outdated → all) then `MachineInFailureDomainWithMostMachines`.

## Upgrade sequencing

`updateControlPlane` (`update.go:36`) runs *before* any Machine churn, against the workload cluster:

- `EnsureKubeadmPermissions` (`pkg/workload_cluster_rbac.go`) — ClusterRole/Binding new kubeadm versions need.
- `UpdateClusterConfiguration` with mutators for imageRepository, featureGates, apiServer/controllerManager/scheduler, `certificateValidityPeriodDays`, `encryptionAlgorithm`, and etcd local **or** external (mutually exclusive, cannot be switched).
- Only `controlplanev1.RollingUpdateStrategyType` is implemented; anything else logs and no-ops.

kube-proxy and CoreDNS are updated only at the end of a clean reconcile (step 15), so they lag the CP machines by design. `SkipCoreDNSAnnotation` / `SkipKubeProxyAnnotation` (on KCP) opt out.

For topology clusters, both `preflightChecks` and `reconcileUnhealthyMachines` refuse to act while `cluster.spec.topology.version != kcp.spec.version`, unless `clusterv1.ClusterTopologyUpgradeStepAnnotation` matches the KCP version (chained upgrades).

## Preflight checks

`preflight.go:preflightChecks` is the mechanism that enforces "one operation at a time". Called from `scaleUpControlPlane`, `scaleDownControlPlane`, and both in-place paths. Fails (and requeues) when:

- topology version mismatch (above),
- any Machine has a deletionTimestamp (`HasDeletingMachine`),
- `checkHealthiness` finds a Machine without `status.nodeRef`, or with a false/unknown `APIServerPodHealthy` / `ControllerManagerPodHealthy` / `SchedulerPodHealthy` (plus `EtcdPodHealthy` / `EtcdMemberHealthy` when etcd is managed).

`excludeFor ...*clusterv1.Machine` lets scale-down ignore the machine it is about to delete. When `RemediationInProgressAnnotation` is set, a looser `checkHealthinessWhileRemediationInProgress` runs instead (quorum-based, so a partially broken CP can still recover). The `CertificatesAvailable` check is deliberately outside preflight — it only applies to scale up (`scale.go:86`). Results land in `controlPlane.PreflightCheckResults` and surface via `setScalingUpCondition` / `setScalingDownCondition`.

## Etcd membership

- Client wiring: `pkg/cluster.go:GetWorkloadCluster` builds a `*tls.Config` from the `<cluster>-etcd` Secret. If the etcd CA **key** is present, KCP mints a client cert (cached 24h via `ClientCertCache`, keyed by cluster+UID+algorithm); if not, the cluster uses external etcd and KCP reuses the `<cluster>-apiserver-etcd-client` Secret. Host name verification is skipped (`InsecureSkipVerify`) because members are dialed through the API server proxy by static pod name; `verifyEtcdServerCertificate` re-adds chain verification.
- Dialing: `pkg/etcd_client_generator.go:forFirstAvailableNode` over `pkg/proxy/`.
- Normal removal path is the **pre-terminate hook**, not scale down: `scaleDownControlPlane` only forwards leadership, then `reconcilePreTerminateHook` removes the member after drain and volume detach. On KCP deletion the hook is stripped first (`removePreTerminateHookAnnotationFromMachine`) since you cannot forward leadership with no members left.
- `reconcileEtcdMembers` is the safety net for orphan members. It skips while any Machine lacks a nodeRef, and tolerates an unnamed member (a kubeadm learner) for `etcdMemberStartupGracePeriod = 20s` after a Node registers — this guards the NodeRef-before-etcd-name race. Guarded by `targetEtcdClusterHealthy` so removal cannot cause quorum loss.
- `controlPlane.NodeListError != nil` blocks all etcd operations: no node list, no port-forward.

## Remediation

`remediation.go:reconcileUnhealthyMachines` handles Machines with `MachineHealthCheckSucceeded=False` and `MachineOwnerRemediated=False` (i.e. MHC flagged it and delegated to the owner). Gates, in order:

1. Clean up stale `MachineOwnerRemediated` conditions and a stale `RemediationInProgressAnnotation`.
2. `getMachineToBeRemediated` — oldest not-yet-provisioned, else oldest; `pickMachineToBeRemediated` prefers the one with the worse etcd state when etcd is managed.
3. Skip if another remediation is in progress (annotation present).
4. Topology version mismatch → `KubeadmControlPlaneMachineRemediationDeferredReason`.
5. `checkRetryLimits` — `kcp.spec.remediation.{maxRetry,retryPeriodSeconds,minHealthyPeriodSeconds}`; state is serialized into the `RemediationInProgressAnnotation` as `RemediationData` and mirrored to `status.lastRemediation`.
6. If already initialized: refuse when `len(Machines) <= 1`, when a healthy Machine is still provisioning (`HasHealthyMachineStillProvisioning`), and when `canSafelyRemediateMachine` says etcd quorum would break.

Remediation is delete-then-recreate; the recreate is a normal scale up and `rollingUpdate` prioritizes it over rollout.

## Certificates, kubeconfig, secrets

Per-cluster Secrets are `<cluster-name>-<purpose>`, type `cluster.x-k8s.io/secret` (`clusterv1.ClusterSecretType`), keys `tls.crt`/`tls.key` (`value` for kubeconfig). Purposes are in `util/secret/consts.go`: `ca`, `etcd`, `sa`, `proxy`, `apiserver-etcd-client`, `kubeconfig`. `util/secret/secret.go:Name` / `ParseSecretName` / `HasPurposeSuffix` are the helpers.

`util/secret/certificates.go` builds the sets: `NewCertificatesForInitialControlPlane` (ca + sa + front-proxy + etcd; swaps etcd for `External` entries plus `apiserver-etcd-client` when external etcd is configured), `NewControlPlaneJoinCerts`, `NewCertificatesForWorker` (cluster CA only). `Generate` / `SaveGenerated` / `LookupOrGenerateCached` / `LookupCached` / `EnsureAllExist` / `AsFiles`.

KCP behavior (`kubeadmcontrolplane_controller.go:reconcileClusterCertificates`): before `KubeadmControlPlaneInitializedCondition` is true it will *generate* missing certs; after that it only *looks them up* and reports `CertificatesNotAvailable` — **CA regeneration is not supported**. `ensureCertificatesOwnerRef` then forces the KCP controllerRef onto each Secret.

Kubeconfig (`helpers.go:reconcileKubeconfig`): create via `kubeconfig.CreateSecretWithOwner`, requeue `dependentCertRequeueAfter` on `ErrDependentCertificateNotFound`, adopt user secrets only if `type == ClusterSecretType`, and rotate with `kubeconfig.NeedsClientCertRotation` / `RegenerateSecret` — only for KCP-owned secrets.

`reconcileCertificateExpiries` reads API server cert expiry from the workload cluster and writes `clusterv1.MachineCertificatesExpiryDateAnnotation` onto the KubeadmConfig; that annotation feeds `rolloutBefore`.

## Workload cluster plumbing and unreachability

`pkg/workload_cluster.go:WorkloadCluster` is the interface; `Workload` is the impl. Clients come from `clustercache.ClusterCache` (`GetClient`, `GetRESTConfig`); failures are wrapped in `pkg.RemoteClusterConnectionError`, which `Reconcile`'s deferred status update special-cases to a log line instead of an error. `pkg/clustercache_utils.go` defines trimmed `Node`/`Pod` types (`TransformNode`/`TransformPod`) so the remote cache stays small.

`reconcileControlPlaneAndMachinesConditions` implements the grace-period ladder using `ClusterCache.GetHealthCheckingState`:

| State | Behavior |
| --- | --- |
| Not initialized | conditions → Unknown / `*InspectionFailed`, `Overwrite: true`, return nil |
| Never probed and `< 5` consecutive failures | `*ConnectionDown`, `Overwrite: false` (keep last known), return error |
| `time.Since(max(lastProbeSuccess, controlPlaneInitialized.LastTransitionTime)) > RemoteConditionsGracePeriod` | `*ConnectionDown`, `Overwrite: true` |
| `ErrClusterNotConnected` | `*ConnectionDown`, `Overwrite: false` |

`--remote-conditions-grace-period` defaults to 5m and `SetupWithManager` rejects anything `< 2m` (must exceed the ClusterCache's worst-case 75s to drop a connection). Other KCP flags: `--etcd-dial-timeout-duration` (10s), `--etcd-call-timeout-duration`.

## Control plane provider contract (what core reads)

Helpers: `internal/contract/controlplane.go` (`contract.ControlPlane()`). Book page: `docs/book/src/developer/providers/contracts/control-plane.md`.

| Contract path | Helper | Read by |
| --- | --- | --- |
| `status.initialization.controlPlaneInitialized` (v1beta1: `status.initialized`) | `Initialized(contractVersion)` | `core/reconcilers/cluster/cluster_controller_phases.go:289`, `cluster_controller_status.go:597` |
| `spec.controlPlaneEndpoint` | `ControlPlaneEndpoint()` | `cluster_controller_phases.go:332` |
| `spec.replicas`, `status.replicas`, `status.readyReplicas`, `status.availableReplicas`, `status.upToDateReplicas` (v1beta1: `status.updatedReplicas`, `status.unavailableReplicas`) | `Replicas()`, `StatusReplicas()`, … | `cluster_controller_status.go:141-166` |
| `status.selector` | `Selector()` | scale subresource |
| `spec.version`, `status.version`, `status.versions` | `Version()`, `StatusVersion()`, `StatusVersions()` | `cluster_controller_status.go:169`, `bootstrap/kubeadm/.../kubeadmconfig_controller.go:getControlPlaneVersion` |
| `Available` condition | `AvailableConditionType()` | mirrored into `ClusterControlPlaneAvailable` |
| `spec.machineTemplate.*`, `spec.rollout.after` | `MachineTemplate()`, `RolloutAfter()` | topology reconciler |

Contract version is **not** an annotation on the object — it is a label on the provider's CRD, `cluster.x-k8s.io/<contractVersion>: <apiVersion>_<apiVersion>` (underscore-separated). `internal/contract/version.go` reads it: `GetContractVersion`, `GetGVKFromGK`, `GetLatestContractAndAPIVersionFromContract`, `GetCompatibleVersions` (v1beta2 is still compatible with v1beta1). CRD name comes from `util/contract/contract.go:CalculateCRDName`.

KCP fills its side in `status.go:updateStatus`: `Selector` from `collections.ControlPlaneSelectorForCluster`, `Versions` from `internalversion.VersionsFromMachines`, `Version` backfilled from the lowest entry, replica counters in `setReplicas` (counted from Machine `Ready`/`Available`/`UpToDate` conditions), and `setControlPlaneInitialized` which flips `Initialization.ControlPlaneInitialized` once the `kubeadm-config` ConfigMap exists in the workload cluster (never back to false).

## Bootstrap provider contract and CABPK

Contract helpers: `internal/contract/bootstrap.go` — `DataSecretCreated(contractVersion)` → `status.initialization.dataSecretCreated` (v1beta1: `status.ready`), `DataSecretName()` → `status.dataSecretName`. The Machine controller consumes them in `core/reconcilers/machine/machine_controller_phases.go:190-236`: if not created, wait; else copy into `machine.spec.bootstrap.dataSecretName` and set `machine.status.initialization.bootstrapDataSecretCreated`. An empty `status.dataSecretName` with `dataSecretCreated=true` is a hard error. Book page: `docs/book/src/developer/providers/contracts/bootstrap-config.md`.

`kubeadmconfig_controller.go:reconcile` decides what to generate, in this switch order:

1. Cluster infra not provisioned → wait.
2. `configOwner.DataSecretName() != nil` but status stale → backfill status only (pivot / restore-from-backup case).
3. `status.initialization.dataSecretCreated` already true → nothing to regenerate, but if a bootstrap token is in use: `refreshBootstrapTokenIfNeeded` when the owner has no nodeRefs, or `rotateMachinePoolBootstrapToken` for MachinePools.
4. `ClusterControlPlaneInitialized` false → `handleClusterNotInitialized` (control plane machines only; workers requeue 30s).
5. Otherwise `joinControlplane` or `joinWorker`.

`handleClusterNotInitialized` takes the init lock — `bootstrap/kubeadm/pkg/locking/control_plane_init_mutex.go`, a `<cluster>-lock` ConfigMap in the management cluster — so exactly one Machine runs `kubeadm init`; the lock is released once the cluster reports initialized. Certificates are generated here only when the Cluster has **no** `spec.controlPlaneRef` (standalone CP machines); otherwise CABPK only looks them up and defers generation to KCP.

User data: `bootstrap/kubeadm/pkg/cloudinit/` (default, `bootstrapv1.CloudConfig`) or `bootstrap/kubeadm/pkg/ignition/` + `pkg/ignition/clc/` (`bootstrapv1.Ignition`), selected by `spec.format` at each of the three call sites (`handleClusterNotInitialized`, `joinWorker`, `joinControlplane`). Both consume the same `cloudinit.BaseUserData`. Adding a user-data feature means touching both packages.

Data Secret (`storeBootstrapData`): name == KubeadmConfig name, namespace == config namespace, label `cluster.x-k8s.io/cluster-name`, controller ownerRef to the KubeadmConfig, type `clusterv1.ClusterSecretType`, `data` keys `value` (the user data) and `format`. Create-then-fallback-to-Update. `ensureBootstrapSecretOwnersRef` repairs ownership on every reconcile and errors if another kind controls the Secret.

Token handling: `reconcileDiscovery` fills `joinConfiguration.discovery.bootstrapToken` (CA hashes from the cluster CA, `apiServerEndpoint` from `cluster.spec.controlPlaneEndpoint`, and a token created in the **workload** cluster via `token.go:createToken`), patches the KubeadmConfig, then `DeferNextReconcileUntilCacheUpToDate` + 1s requeue — this stale-cache guard exists to avoid minting duplicate tokens (issue #12253). TTL from `--bootstrap-token-ttl` (`DefaultTokenTTL = 15m`). `joinWorker` always returns `RequeueAfter: tokenCheckRefreshOrRotationInterval()` so the token keeps being refreshed until consumed.

## kubeadm API vendoring

`bootstrap/kubeadm/pkg/types/` holds the upstream kubeadm types CAPI marshals to, per `docs/proposals/20210210-insulate-users-from-kubeadm-API-changes.md`. Only `upstreamv1beta4` remains; `upstreamhub/` is the conversion hub (aliasing the CAPI `bootstrapv1` types). `pkg/types/utils.go:KubeVersionToKubeadmAPIGroupVersion` maps Kubernetes >= 1.31 to `v1beta4` and **errors** below that. `Marshal{Cluster,Init,Join}ConfigurationForVersion` / `Unmarshal*` are the entry points; `upstream.AdditionalData` carries values that live in the kubeadm type but not the CAPI type.

The joining Machine's own version selects the kubeadm API version, *not* the control plane version (see the comment at `kubeadmconfig_controller.go:699`). The control plane version is only exposed as the `controlPlane.version` template variable for `spec.files` with `contentFormat: Template` (`getControlPlaneVersion` + `template.go:renderTemplates`, capped at 1 MiB).

## Gotchas when changing this code

- **Generated artifacts.** `make generate` covers: CRDs/RBAC/webhooks into `bootstrap/kubeadm/config/{crd/bases,rbac,webhook}` and `controlplane/kubeadm/config/...` (Makefile targets `generate-manifests-kubeadm-bootstrap`, `generate-manifests-kubeadm-control-plane` — note they scan `paths=./{bootstrap,controlplane}/kubeadm/reconcilers/...` and `.../webhooks/...`, so RBAC markers must stay in those trees); deepcopy for `api/{bootstrap,controlplane}/kubeadm/...` and `bootstrap/kubeadm/pkg/types/...`; conversions for `api/*/kubeadm/v1beta1` and `bootstrap/kubeadm/pkg/types/upstreamv1beta4`.
- **v1beta1 conversion must round-trip.** Every new v1beta2 field needs conversion handling in `api/{bootstrap,controlplane}/kubeadm/v1beta1/conversion.go` plus its fuzz test, and conversion webhooks in `{bootstrap,controlplane}/kubeadm/webhooks/conversion/`.
- **Two condition systems.** Nearly every code path sets both a v1beta2 condition (`util/conditions`) and a deprecated one (`util/conditions/deprecated/v1beta1`). New conditions must be added to the `patch.WithOwnedConditions` / `WithOwnedV1Beta1Conditions` lists in `patchKubeadmControlPlane` (and the CABPK equivalent in `Reconcile`), or the patch helper will drop them on conflict. Do not invent new v1beta1 reasons.
- **Rollout diffs.** Any new `KubeadmConfigSpec` or MachineTemplate field silently becomes a rollout trigger unless handled in `PrepareKubeadmConfigsForDiff` / `dropOmittableFields`. `filters_test.go` (2100+ lines) is the guard.
- **Defaulting symmetry.** `PrepareKubeadmConfigsForDiff` calls `bootstrapadmission.ApplyPreviousKubeadmConfigDefaults`; a new default that changes semantics relative to unset will cause a mass rollout on upgrade.
- **SSA field managers.** `capi-kubeadmcontrolplane` owns spec; `capi-kubeadmcontrolplane-metadata` owns labels/annotations (split introduced in v1.12). `ssa.MigrateManagedFields` / `MitigateManagedFieldsIssue` migration code in `syncMachines` is dated for removal in v1.15 — do not delete early.
- **Ordering rules that break easily.** Certificates before kubeconfig before `syncMachines`; conditions before preflight (preflight reads the conditions computed in step 7); remediation before rollout; `reconcileEtcdMembers` before `reconcilePreTerminateHook`; kube-proxy/CoreDNS/cert-expiry strictly last so they never block recovery.
- **Cache-consistency calls.** `DeferNextReconcileForObject` and `DeferNextReconcileUntilCacheUpToDate` (`util/controller`) are correctness, not optimization — dropping them reintroduces duplicate-token and double-in-place-trigger races. `ClearConsistencyStore` must be called on NotFound and on finalizer removal.
- **Test surfaces.** `kubeadmcontrolplane_controller_test.go` (~5200 lines), `remediation_test.go`, `status_test.go`, `filters_test.go`, `workload_cluster_conditions_test.go`, `bootstrap/.../kubeadmconfig_controller_test.go`. The `override*Func` fields on the KCP `Reconciler` exist purely for unit tests; keep them wired when refactoring. E2E: `test/e2e/` (`cluster_upgrade*`, `kcp_remediations*`, `self_hosted`).

## Where to read more

- `docs/proposals/20191017-kubeadm-based-control-plane.md` — the original KCP design, including the delete-and-recreate remediation contract.
- `docs/proposals/20210210-insulate-users-from-kubeadm-API-changes.md` — why `bootstrap/kubeadm/pkg/types/` exists.
- `docs/proposals/20240807-in-place-updates.md` and `-implementation-notes.md` — the `CanUpdateMachine`/`UpdateMachine` flow behind `inplace_*.go`.
- `docs/proposals/20250513-chained-and-efficient-upgrades-for-clusters-with-managed-topologies.md` — `ClusterTopologyUpgradeStepAnnotation`.
- `docs/proposals/20260701-code-organization.md` — the `internal/` → `pkg/`/`reconcilers/` move.
- `docs/book/src/developer/providers/contracts/control-plane.md`, `bootstrap-config.md` — normative contracts.
- `docs/book/src/tasks/control-plane/kubeadm-control-plane.md`, `docs/book/src/tasks/bootstrap/kubeadm-bootstrap/index.md` — user-facing behavior.
- `docs/book/src/tasks/certs/{using-custom-certificates,auto-rotate-certificates-in-kcp,generate-kubeconfig}.md` — secret conventions from the user side.
- `docs/book/src/reference/api/labels-and-annotations.md` — the annotations referenced above.

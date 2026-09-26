# Provider Contracts

How core Cluster API talks to out-of-tree providers. Core controllers never import provider types: they read and write provider objects as `unstructured.Unstructured` (or via narrow Go interfaces over them) at documented field paths, and they resolve which apiVersion to use from labels on the provider's CRD. This file covers the contract-version label scheme, the `internal/contract/` accessor package, per-resource field tables, cross-cutting labels/annotations/ownerRefs, pausing, clusterctl move requirements, and the in-tree reference provider under `test/infrastructure/docker/`.

## Why contracts exist

- Core refs are **version-less**: `clusterv1.ContractVersionedObjectReference` (`api/core/v1beta2/common_types.go:391`) carries only `apiGroup`, `kind`, `name`. The version is looked up at runtime from the CRD.
- Reads/writes go through path accessors so a field rename in core does not require recompiling providers, and so core can support two contract versions at once.
- Anything not written down as a contract rule is an internal implementation detail providers must not rely on (see the warning boxes in `docs/book/src/developer/providers/contracts/*.md`).

## Contract version discovery (labels, not annotations)

A provider declares contract support with **labels on its CRDs**: `cluster.x-k8s.io/<contract-version>: <underscore-separated list of that CRD's versions>`.

```yaml
# e.g. test/infrastructure/docker/config/crd/kustomization.yaml
labels:
- pairs:
    cluster.x-k8s.io/v1beta1: v1beta1
    cluster.x-k8s.io/v1beta2: v1beta2
```

Enforcement/read path, all in `internal/contract/version.go`:

| Symbol | Purpose |
| --- | --- |
| `Version` | contract version this CAPI build implements; equals `clusterv1.GroupVersion.Version` |
| `GetCompatibleVersions(contract)` | today `v1beta2` is also compatible with `v1beta1` |
| `getCRDLabels` | `Get`s a `PartialObjectMetadata` for the CRD named by `util/contract.CalculateCRDName(group, kind)` (`flect.Pluralize(lower(kind)) + "." + group`) |
| `GetLatestContractAndAPIVersionFromContract` | picks the newest compatible contract label, then the newest version inside it (kube-aware sort) |
| `GetGVKFromGK` / `GetContractVersion` | what controllers call: group+kind in, contract version + concrete GVK out |
| `GetContractVersionForVersion` | reverse lookup: which contract does this apiVersion belong to |
| `UpdateReferenceAPIContract` | rewrites the version in a legacy `corev1.ObjectReference` in place |

Consequences: **the CRD must be named exactly as `CalculateCRDName` computes**, or the lookup fails. `clusterctl` pre-flights this (`cmd/clusterctl/client/cluster/installer.go:validateCRDName`) and the check can be silenced with `clusterctl.cluster.x-k8s.io/skip-crd-name-preflight-check`. A missing/empty contract label produces the "cannot find any versions matching contract versions" error.

## `internal/contract/` package

Two layers:

**1. Path accessors** (`types.go`) — `Path []string` plus typed wrappers each exposing `Path()`, `Get(*unstructured.Unstructured)`, `Set(...)`: `Int64`, `Int32`, `Bool`, `String`, `StringSlice`, `Duration`, `Time`, plus `Metadata` (`metadata.go`), `V1Beta1Ref` (`references.go`), `FailureDomains` / `ControlPlaneEndpoint` (`infrastructure_cluster.go`), `StatusVersions` / `ReadinessGates` / `Taints` (`controlplane.go`), `ProvisionedBool` (`infrastructure_machinepool.go`, primary path + fallback path). `Get` returns `ErrFieldNotFound` (wrapped) when the field is absent — callers must `errors.Is` on it rather than treating it as fatal.

**2. Per-resource contract singletons**, one file each, all built with the same `sync.Once` + empty-struct pattern so call sites read as `contract.InfrastructureCluster().ControlPlaneEndpoint().Get(obj)`:

| Constructor | File |
| --- | --- |
| `contract.InfrastructureCluster()` | `internal/contract/infrastructure_cluster.go` |
| `contract.InfrastructureClusterTemplate()` | `internal/contract/infrastructure_cluster_template.go` |
| `contract.InfrastructureMachine()` | `internal/contract/infrastructure_machine.go` |
| `contract.InfrastructureMachineTemplate()` | `internal/contract/infrastructure_machine_template.go` |
| `contract.InfrastructureMachinePool()` | `internal/contract/infrastructure_machinepool.go` |
| `contract.InfrastructureMachinePoolTemplate()` | `internal/contract/infrastructure_machinepool_template.go` |
| `contract.Bootstrap()` | `internal/contract/bootstrap.go` |
| `contract.BootstrapConfigTemplate()` | `internal/contract/bootstrap_config_template.go` |
| `contract.ControlPlane()` | `internal/contract/controlplane.go` |
| `contract.ControlPlaneTemplate()` | `internal/contract/controlplane_template.go` |

Accessors that changed between contract versions take `contractVersion string` and branch, e.g. `InfrastructureCluster().Provisioned(contractVersion)` returns `status.ready` for `v1beta1` and `status.initialization.provisioned` for `v1beta2`.

**3. Typed views over unstructured** (`internal/contract/api/`) — `InfraMachine` and `BootstrapConfig` Go interfaces (`inframachine_types.go`, `bootstrapconfig_types.go`) with per-contract-version implementations in `internal/contract/api/v1beta1/` and `v1beta2/`. The Machine controller Gets a concrete typed object through the dynamic cache and casts: `s.infraMachine = obj.(contractapi.InfraMachine)` (`core/reconcilers/machine/machine_controller_phases.go:276`). `contractapi.Conditions` has a custom `UnmarshalJSON` so unknown/partial provider conditions do not fail decoding (`internal/contract/api/condition_types.go`).

### Adding a new contract field

1. Add the accessor method on the right `*Contract` struct, returning a typed accessor with the literal path; take `contractVersion string` if the path differs per contract version. Add a table-driven test next to it (`*_test.go` in the same package).
2. If the field is consumed by the typed path, add the getter to the interface in `internal/contract/api/` and implement it in **both** `v1beta1/` and `v1beta2/`, then `make generate` for deepcopy.
3. Use it from the controller and always tolerate `contract.ErrFieldNotFound` for optional fields.
4. Document the rule in `docs/book/src/developer/providers/contracts/<resource>.md` (rules table + section) and, if it is mandatory, add a migration note in `docs/book/src/developer/providers/migrations/`.
5. If it must be reconciled by topology, check `IgnorePaths` and `internal/util/ssa` / `structuredmerge` behaviour.

## InfrastructureCluster

Read by `core/reconcilers/cluster/cluster_controller_phases.go` and `..._status.go`.

| Field | Contract | Required | Notes |
| --- | --- | --- | --- |
| `spec.controlPlaneEndpoint{host,port}` | both | if not supplied elsewhere | mirrored into `Cluster.spec.controlPlaneEndpoint`; zero host/port are excluded from reconciliation via `InfrastructureClusterContract.IgnorePaths` |
| `status.initialization.provisioned` | v1beta2 | yes | `status.ready` for v1beta1 |
| `status.failureDomains` | both | no | v1beta1 = map, v1beta2 = list of `clusterv1.FailureDomain`; `FailureDomains.Get` converts and sorts |
| `status.conditions[Ready]` | both | no | mirrored to `Cluster` `InfrastructureReady`; type from `ReadyConditionType()` |
| `status.failureReason` / `status.failureMessage` | v1beta1 | deprecated | accessors marked Deprecated |
| CRD `spec.template` on `…Template` | both | for ClusterClass | `InfrastructureClusterTemplate().Template().Metadata()` |

## InfrastructureMachine

Read by `core/reconcilers/machine/machine_controller_phases.go:reconcileInfrastructure` and `machine_controller_status.go:setInfrastructureReadyCondition`, typed via `internal/contract/api/v1beta2/inframachine_types.go`.

| Field | Required | Notes |
| --- | --- | --- |
| `spec.providerID` | yes | copied to `Machine.spec.providerID`; gates NodeRef |
| `status.initialization.provisioned` | yes | `status.ready` in v1beta1 |
| `status.addresses` | no | `[]clusterv1.MachineAddress` |
| `status.failureDomain` | no | actual placement; `spec.failureDomain` is the deprecated request field |
| `status.interruptible` | no | drives the `cluster.x-k8s.io/interruptible` Node label |
| `status.conditions[Ready]` | no | mirrored into Machine `InfrastructureReady` |
| `status.deprecated.v1beta1.{conditions,failureReason,failureMessage}` | no | deprecated bucket |
| `…Template` with `spec.template.spec` | yes | must tolerate SSA dry-run for ClusterClass; autoscale-from-zero uses `capacity` annotations |

## BootstrapConfig

| Field | Required | Notes |
| --- | --- | --- |
| `status.initialization.dataSecretCreated` | yes | `status.ready` in v1beta1 |
| `status.dataSecretName` | yes | copied to `Machine.spec.bootstrap.dataSecretName`; empty value is a hard error |
| `status.conditions[Ready]` | no | mirrored to Machine `BootstrapConfigReady` |
| `status.deprecated.v1beta1.*` | no | deprecated failure/conditions |
| `…Template` with `spec.template.spec` | yes | ClusterClass |

## ControlPlane

`internal/contract/controlplane.go`; consumed by the Cluster controller and by topology (`core/reconcilers/topology/cluster/...`).

| Field | Notes |
| --- | --- |
| `spec.version`, `status.version`, `status.versions[]` | `StatusVersions.Get` validates ordering and non-negative replicas |
| `spec.replicas`, `status.{replicas,readyReplicas,availableReplicas,upToDateReplicas,selector}` | `upToDateReplicas`→`updatedReplicas`, `availableReplicas`→`unavailableReplicas` under v1beta1 |
| `status.initialization.controlPlaneInitialized` | `status.initialized` in v1beta1 |
| `spec.controlPlaneEndpoint` | same accessor type as InfraCluster; same `IgnorePaths` behaviour |
| `spec.rollout.after` | `RolloutAfter()` |
| `spec.machineTemplate.{metadata,spec.*}` | `ControlPlaneMachineTemplate`: `InfrastructureRef`, `Metadata`, `NodeDrainTimeoutSeconds`, `NodeVolumeDetachTimeoutSeconds`, `NodeDeletionTimeoutSeconds`, `ReadinessGates`, `Taints` (last two moved under `spec` in v1beta2) |
| `status.conditions[Available]` | `AvailableConditionType()`, mirrored to `ControlPlaneAvailable` |
| derived helpers | `IsProvisioning`, `IsUpgrading`, `IsScaling`, `CurrentStatusVersion` |

Also mandatory but outside this package: the kubeconfig secret `<cluster>-kubeconfig` of type `cluster.x-k8s.io/secret`.

## MachinePool / InfraMachinePool

| Field | Required | Notes |
| --- | --- | --- |
| `spec.providerIDList` | yes | `StringSlice` |
| `status.replicas` | yes | |
| `status.initialization.provisioned` | yes | `ProvisionedBool` falls back to `status.ready` even under v1beta2 because MachinePool providers lag |
| `status.infrastructureMachineKind` | no | opt-in to MachinePool Machines |

## IPAM

No `internal/contract` accessors; core types live in `api/ipam/v1beta2/` (`ipaddress_types.go`, `ipaddressclaim_types.go`). Providers reconcile `IPAddressClaim` → `IPAddress`, and pool CRDs must follow `CalculateCRDName`, expose `status.conditions[Ready]`, and carry the `cluster.x-k8s.io/cluster-name` label so pools move with the cluster.

## Cross-cutting rules

- **Labels/annotations** are constants in `api/core/v1beta2/common_types.go` and `machine_types.go`. Most relevant to providers: `ClusterNameLabel` (`cluster.x-k8s.io/cluster-name`), `ProviderNameLabel`, `WatchLabel` (`--watch-filter`), `InterruptibleLabel`, `PausedAnnotation`, `ManagedByAnnotation` (externally managed infra), `ReplicasManagedByAnnotation`, `DisableMachineCreateAnnotation`, `DeleteMachineAnnotation`, `TemplateClonedFromNameAnnotation` / `TemplateClonedFromGroupKindAnnotation`, `AutoscalerMinSizeAnnotation` / `AutoscalerMaxSizeAnnotation`, `MachineSkipRemediationAnnotation`, `RemediateMachineAnnotation`. Helpers in `util/annotations/helpers.go` (`IsPaused`, `IsExternallyManaged`, `HasPaused`, `ReplicasManagedByExternalAutoscaler`).
- **Owner references core adds**: Cluster controller sets a plain (non-controller) ownerRef on the InfraCluster/ControlPlane and sets `ClusterNameLabel` (`core/reconcilers/cluster/cluster_controller_phases.go:100-134`). Machine controller sets itself as **controller** ref on InfraMachine/BootstrapConfig, removes the transient MachineSet/control-plane "on create" owner refs, and sets `ClusterNameLabel` (`core/reconcilers/machine/machine_controller_phases.go:ensureExternalOwnershipAndWatch`). Objects cloned from templates get `ClusterNameLabel` + the two `cloned-from` annotations in `controllers/external/util.go:GenerateTemplate`; the `Template` suffix is stripped from the Kind.
- **RBAC**: if your API group is not `infrastructure.cluster.x-k8s.io`, ship a `ClusterRole` labelled `cluster.x-k8s.io/aggregate-to-manager: "true"` (see `core/config/rbac/aggregated_role.yaml`).
- **Pausing**: call `util/paused.EnsurePausedCondition(ctx, client, cluster, obj)` first thing after fetching the owner Cluster. It returns paused when `Cluster.spec.paused` is true **or** the object carries `cluster.x-k8s.io/paused`, patches the `Paused` condition with `ObservedGeneration: obj.GetGeneration()`, and returns `requeue=true` when it wrote. Skipping the requeue when only `observedGeneration` changed is deliberate (`util/paused/paused.go:71`). Reporting `Paused` matters because `clusterctl move` waits on it indirectly via `clusterctl.cluster.x-k8s.io/block-move`.
- **clusterctl move** (constants in `cmd/clusterctl/api/v1alpha3/labels.go` and `annotations.go`): every provider object must reach a `Cluster` through the ownerRef chain, or carry `clusterctl.cluster.x-k8s.io/move` / `move-hierarchy` (on the object or the CRD). Objects that cannot be paused instantly should set `clusterctl.cluster.x-k8s.io/block-move` and clear it once paused; `clusterctl.cluster.x-k8s.io/delete-for-move` is set on source objects before deletion so webhooks can allow it.

## Reference implementations

`test/infrastructure/docker/` is CAPD, now a **single provider with pluggable backends** (`docs/proposals/20250124-From CAPD(docker) to CAPD(dev) .md`). There is no `DockerCluster`/`DockerMachine` CRD any more.

| Contract obligation | Where |
| --- | --- |
| CRDs + contract labels | `test/infrastructure/docker/config/crd/kustomization.yaml`, bases `infrastructure.cluster.x-k8s.io_dev{clusters,clustertemplates,machines,machinetemplates,machinepools,machinepooltemplates}.yaml` |
| API types | `test/infrastructure/docker/api/v1beta2/devcluster_types.go`, `devmachine_types.go`, … plus `api/v1beta1/` + conversion for the older contract |
| InfraCluster loop (externally-managed check, ownerCluster lookup, finalizer, `EnsurePausedCondition`, patch-on-defer, backend dispatch) | `test/infrastructure/docker/reconcilers/devcluster_reconciler.go` |
| InfraMachine / InfraMachinePool loops | `devmachine_reconciler.go`, `devmachinepool_reconciler.go`, `devmachinetemplate_reconciler.go` |
| Backend interface | `test/infrastructure/docker/reconcilers/backends/cluster.go:DevClusterBackendReconciler` (+ `machine.go`, `machinepool.go`) |
| docker backend: sets `spec.controlPlaneEndpoint`, `status.failureDomains`, `status.initialization.provisioned`, `spec.providerID`, `status.addresses` | `reconcilers/backends/docker/dockercluster_backend.go`, `dockermachine_backend.go`, `dockermachinepool_backend.go` |
| container/LB plumbing | `test/infrastructure/docker/internal/docker/`, `internal/loadbalancer/`, `internal/provisioning/` (cloudinit + ignition) |
| webhooks, templates | `test/infrastructure/docker/webhooks/`, `test/infrastructure/docker/templates/cluster-template*.yaml` |

The **in-memory** implementation is now the second CAPD backend: `test/infrastructure/docker/reconcilers/backends/inmemory/` (see its `README.md` and `architecture.drawio.svg`), backed by the fake API server and in-memory resource-group runtime in `test/infrastructure/inmemory/pkg/{runtime,server,cloud}`. It fakes a control plane so CAPI believes a real cluster exists; it is for scale-testing CAPI itself (thousands of clusters on a laptop), never for workloads.

## If you're implementing or debugging a provider

1. CRD name is exactly `CalculateCRDName(group, kind)`; otherwise every ref lookup fails.
2. CRD carries `cluster.x-k8s.io/v1beta2: <versions>` (and `v1beta1` while supporting both); the value must list real CRD versions, and the newest is chosen.
3. `metadata.yaml`'s `releaseSeries[].contract` matches that label.
4. Resources are namespace-scoped with normal `TypeMeta`/`ObjectMeta` and a `status` subresource.
5. Set the mandatory status fields from the tables above — core logs exactly which path it is waiting on (`contract.Bootstrap().DataSecretCreated(v).Path().String()` etc.), so grep the core logs first.
6. Add a `Ready` condition; core mirrors it and its absence shows up as `Unknown` on the core object.
7. Do not fight core over fields core owns: it sets ownerRefs and `cluster.x-k8s.io/cluster-name`; don't strip them.
8. Implement pausing via `util/paused.EnsurePausedCondition` and honour `cluster.x-k8s.io/paused` + `Cluster.spec.paused`.
9. Everything must be reachable from a `Cluster` by ownerRefs, or be labelled for move.
10. Provide `…Template` types that survive SSA dry-run if ClusterClass support is wanted.
11. Debugging "field not found": confirm which contract version core resolved (`GetContractVersion`) — a v1beta1 label makes core read `status.ready` and ignore `status.initialization.provisioned`.

## Where to read more

- `docs/book/src/developer/providers/contracts/overview.md` and per-resource pages `infra-cluster.md`, `infra-machine.md`, `infra-machinepool.md`, `bootstrap-config.md`, `control-plane.md`, `ipam.md`, `clusterctl.md`
- `docs/book/src/developer/providers/getting-started/`, `best-practices.md`, `security-guidelines.md`, `migrations/`
- `docs/proposals/20240916-improve-status-in-CAPI-resources.md` (initialization/deprecated status split, conditions v2)
- `docs/proposals/20250124-From CAPD(docker) to CAPD(dev) .md` (DevCluster + backends)
- `docs/proposals/20210203-externally-managed-cluster-infrastructure.md`, `20220125-ipam-integration.md`, `20220209-machinepool-machines.md`, `20260701-code-organization.md`

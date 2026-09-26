# clusterctl

`cmd/clusterctl/` is the CLI that installs and upgrades providers in a management cluster, renders workload-cluster templates, moves clusters between management clusters, and renders the `describe cluster` tree. It is also consumed as a **library** by the E2E framework, so almost all logic lives under `client/` with the cobra layer kept thin. This file covers the layering, the main command flows, provider repository/metadata expectations, the `move` object graph (and what a new CRD must do to be movable), template processing, cert-manager, the upgrade planner, and where the tests are.

## Layering

| Layer | Path | Responsibility |
| --- | --- | --- |
| cobra commands | `cmd/clusterctl/cmd/` (`root.go`, `init.go`, `move.go`, `upgrade_plan.go`, `upgrade_apply.go`, `generate_cluster.go`, `generate_yaml.go`, `generate_provider.go`, `describe_cluster.go`, `get_kubeconfig.go`, `delete.go`, `convert.go`, `config_repositories.go`, `rollout/`) | flag parsing, output formatting, `version_checker.go` nag |
| high-level client | `cmd/clusterctl/client/` (`client.go:Client`, `init.go`, `move.go`, `upgrade.go`, `describe.go`, `delete.go`, `common.go`, `clusterclass.go`, `alias.go`) | orchestration; everything is injectable via `InjectConfig`, `InjectRepositoryFactory`, `InjectClusterClientFactory`, `InjectCurrentContractVersion` |
| config | `client/config/` | reads `clusterctl.yaml` + env (`reader_viper.go`), the built-in provider list (`providers_client.go:defaults`), variables (`variables_client.go`), image overrides (`imagemeta_client.go`), cert-manager config (`cert_manager_client.go`) |
| repository | `client/repository/` | fetches artifacts from a provider repository: `repository_github.go`, `repository_gitlab.go`, `repository_local.go`, `repository_memory.go` behind `client.go:Repository`; `components_client.go`, `template_client.go`, `clusterclass_client.go`, `metadata_client.go`, `overrides.go` |
| cluster | `client/cluster/` | everything that touches a live cluster: `proxy.go`, `inventory.go`, `installer.go`, `components.go`, `upgrader.go` + `upgrader_info.go`, `mover.go` + `objectgraph.go` + `ownergraph.go`, `cert_manager.go`, `crd_migration.go`, `template.go`, `workload_cluster.go` |
| tree | `client/tree/` | `describe cluster` object tree (`discovery.go`, `tree.go`, `annotations.go`, `node_object.go`) |
| alpha | `client/alpha/` | `clusterctl alpha rollout` (restart/pause/resume/undo for MD and KCP) |
| yaml processing | `client/yamlprocessor/` | `Processor` interface + `SimpleProcessor` (envsubst-style) |
| API types | `cmd/clusterctl/api/v1alpha3/` | `Provider` inventory CRD, `Metadata`/`ReleaseSeries`, labels, annotations; CRD YAML embedded via `cmd/clusterctl/config/embedded_manifest.go` (`manifest/clusterctl-api.yaml`) |
| support | `cmd/clusterctl/internal/{scheme,test,util}`, `cmd/clusterctl/log` | scheme, fake object graph builders, logging |

`internal/` here is only scheme/test/util — the real internals are the unexported types inside `client/*` packages.

## Main flows

### `init` — `client/init.go:Init`
1. `clusterClient.ProviderInventory().EnsureCustomResourceDefinitions` installs the embedded `Provider` CRD.
2. `CheckCAPIContract(AllowCAPINotInstalled{})` rejects a management cluster on an incompatible contract.
3. `addDefaultProviders` — on first run forces core + a bootstrap + a control-plane provider (`NoopProvider = "-"` opts out).
4. `setupInstaller` resolves each provider to a repository, version and target namespace and enqueues `repository.Components`.
5. `installer.Validate` (`client/cluster/installer.go:177`): exactly one core provider, only one instance per provider (`simulateInstall`), every provider's contract must be in `GetCompatibleVersions(coreContract)`, and every CRD name must match `util/contract.CalculateCRDName` (`validateCRDName`).
6. `certManager.EnsureInstalled`.
7. `installer.Install` applies components in provider-type order and records inventory; `waitForProvidersReady` waits on the manager Deployments when `--wait-providers`.

### `upgrade plan` / `upgrade apply` — `client/cluster/upgrader.go`
- `Plan`: the **core provider drives the whole cluster**. `upgrader_info.go` builds per-provider upgrade info from `metadata.yaml`; `getContractsForUpgrade` yields the current contract plus newer ones. For each contract an `UpgradePlan` is built with one `UpgradeItem` per provider (`getLatestNextVersion` against `getCompatibleContractVersions`). Plans that are partial (`isPartialUpgrade`) **and** require a contract change are dropped, because all providers must change contract together.
- `ApplyPlan` refuses any contract other than the clusterctl build's own `currentContractVersion`; `ApplyCustomPlan` builds a plan from explicit `--<type>-provider` flags.
- `doUpgrade`: `CheckSingleProviderInstance`; blocks skipping more than 3 minor versions for core/kubeadm-bootstrap/kubeadm-control-plane above v1.10; sorts by `ProviderType.Order()` (Core → Bootstrap → ControlPlane → Infrastructure); **scales all old provider Deployments to 0 first** (otherwise a surviving old pod serves the conversion webhook and the new pod's LISTs fail); then per provider deletes components preserving CRDs/namespace/inventory and installs the new version.
- CRD storage-version migration for provider CRDs is *not* done here — that is the in-cluster `controllers/crdmigrator`. `client/cluster/crd_migration.go:CRDMigrator` is used only for cert-manager (`cert_manager.go:304`).

### `move` — `client/move.go` → `client/cluster/mover.go`
Three modes, mutually exclusive: to another cluster (`--to-kubeconfig`), `--to-directory` (backup), `--from-directory` (restore). `--dry-run` skips all writes and the readiness gates.

### `alpha rollout` — `client/rollout.go` + `client/alpha/`
Subcommands `restart`, `pause`, `resume` (`cmd/rollout/{restart,pause,resume}.go`) against the two resource kinds declared in `client/alpha/rollout.go` (`machinedeployment`, `kubeadmcontrolplane`); the per-kind implementations live in `alpha/machinedeployment.go`, `alpha/kubeadmcontrolplane.go` and `rollout_{pauser,restarter,resumer}.go`.

### `generate cluster` / `generate yaml` / `generate provider`
- `client/cluster/template.go` + `client/repository/template_client.go`; `generate_cluster.go` resolves the template from the infra provider repository, a URL, a file, or a ConfigMap.
- `generate provider` (`client/generate_provider.go`) renders component YAML without applying it.

### `describe cluster` — `client/describe.go` → `client/tree/discovery.go:Discovery`
Walks Cluster → InfraCluster → ControlPlane (+ its machines) → MachineDeployments/MachineSets/Machines → MachinePools → ClusterResourceSets, grouping and echoing conditions; `addTemplateVirtualNode` inserts synthetic nodes for templates. Grouping/visibility is driven by annotations in `client/tree/annotations.go`.

### backup / restore
There are no separate commands; they are `move --to-directory` / `move --from-directory` (`mover.go:ToDirectory`, `FromDirectory`, `toDirectory`, `fromDirectory`, `backupTargetObject`, `restoreTargetObject`, `filesToObjs`). Each object becomes one JSON file named by `node.getFilename()`; restore rebuilds the graph with `objectGraph.addRestoredObj` and `buildOwnerChain`.

## Provider metadata and repositories

`metadata.yaml` (`cmd/clusterctl/api/v1alpha3/metadata_type.go`) maps release series to contract version and must sit at the root of every release:

```yaml
apiVersion: clusterctl.cluster.x-k8s.io/v1alpha3
kind: Metadata
releaseSeries:
- {major: 1, minor: 15, contract: v1beta2}
```

- Fetched by `client/repository/metadata_client.go:Get`, validated by `validateMetadata` (the release series for the version being installed must exist).
- `Metadata.GetReleaseSeriesForVersion` / `GetReleaseSeriesForContract` (the latter returns the newest series for a contract).
- **The contract here must match the `cluster.x-k8s.io/<contract>` labels on the provider's CRDs** — clusterctl trusts `metadata.yaml`, core controllers trust the CRD labels; a mismatch produces confusing runtime failures.

Repository layout expected per version: `infrastructure-components.yaml` (or `<type>-components.yaml`), `metadata.yaml`, `cluster-template[-<flavor>].yaml`, `clusterclass-<name>.yaml` (naming from `client/yamlprocessor/simple_processor.go:GetTemplateName` / `GetClusterClassTemplateName`). Local repositories are `<basepath>/<provider>/<version>/<file>` (`repository_local.go`).

Configuration surface:
- `clusterctl.yaml` under `$XDG_CONFIG_HOME/cluster-api/` (or legacy `$HOME/.cluster-api/`), constants `ConfigFolderXDG`, `ConfigFolder`, `ConfigName` in `client/config/reader_viper.go`; `--config` may be a URL (`DownloadConfigFile`).
- Built-in provider list in `client/config/providers_client.go:defaults()`; user `providers:` entries override/extend it, validated by `validateProvider`. Provider kinds: Core, Bootstrap, ControlPlane, Infrastructure, IPAM, RuntimeExtension, Addon (`ManifestLabel` in `api/v1alpha3/labels.go` produces e.g. `infrastructure-docker`).
- Overrides layer: `$XDG_CONFIG_HOME/cluster-api/overrides/<provider-label>/<version>/<file>` or `overridesFolder` (`client/repository/overrides.go`) — used heavily in E2E and for local dev.
- Image overrides: `client/config/imagemeta_client.go` + `util.FixImages`.
- `clusterctl config repositories` prints the effective list (`cmd/config_repositories.go`).

## `clusterctl move` internals

### Discovery — `client/cluster/objectgraph.go`
- `getDiscoveryTypes`: lists CRDs filtered by `client.HasLabels{clusterctlv1.ClusterctlLabel}` (`clusterctl.cluster.x-k8s.io`, added to every installed component by `repository/components.go:addCommonLabels`), keeps only each CRD's **storage** version, plus core `Secret` and `ConfigMap`.
  - `forceMoveHierarchy` = `Cluster`, `ClusterClass`, `ClusterResourceSet`, or CRD labelled `clusterctl.cluster.x-k8s.io/move-hierarchy`.
  - `forceMove` = `forceMoveHierarchy` or CRD labelled `clusterctl.cluster.x-k8s.io/move`.
- `Discovery`: lists all objects of each type in the namespace (all namespaces if empty); Secrets are additionally listed from every infrastructure-provider namespace; ClusterClasses referenced cross-namespace by a moved Cluster are fetched together with all their templates via `fetchRef`.
- `objToNode` / `objMetaToNode` build `node`s keyed by UID; `processOwnerReferences` creates **virtual** nodes for owners not yet seen (`checkVirtualNode` errors if any stay virtual). `captureAdditionalInformation` records `cluster.spec.topology.class`, `…classNamespace`, `clusterresourcesetbinding.spec.clustername`. The `clusterctl.cluster.x-k8s.io/block-move` annotation sets `node.blockingMove`.
- `setSoftOwnership`: ownerRef-less Secrets whose name parses as a cluster secret (`util/secret.ParseSecretName`) are soft-owned by that Cluster; Clusters are soft-owned by their ClusterClass; `ClusterResourceSetBinding` is soft-owned by its Cluster.
- `setTenants` / `setTenantHierarchy`: propagate tenancy down the owner chain from every `forceMoveHierarchy` node (cycle-guarded). `getMoveNodes` = nodes with at least one tenant, or `forceMove`.
- `setShouldNotDelete` / `setShouldNotDeleteHierarchy`: cross-namespace ClusterClasses (and their templates) are copied but not deleted.
- Global (cluster-scoped) nodes and their descendants (`isGlobal`, `isGlobalHierarchy`) are created in the target but never deleted from the source.

### Ordering and pause — `client/cluster/mover.go`
1. `getObjectGraph` → `checkProvisioningCompleted`: every Cluster must have `status.initialization.infrastructureProvisioned`, `ControlPlaneInitialized` condition true, and every Machine a `status.nodeRef`. Skipped on dry-run.
2. `checkClustersNotPaused` / `checkClusterClassesNotPaused` — move refuses to start on an already-paused Cluster/ClusterClass (it would not know whether to unpause at the end).
3. `setClusterPause(true)` on the source Clusters, `setClusterClassPause(true)` on ClusterClasses (the latter uses the `cluster.x-k8s.io/paused` annotation, `pauseClusterClass`).
4. `waitReadyForMove` polls `getMoveNodes()` for ~2 minutes until no node has `block-move`.
5. `getMoveSequence` groups nodes into waves: a node joins the current group only once **all** its `owners` and `softOwners` are already in the sequence. Create groups forward (`createGroup` → `createTargetObject`, rewriting ownerRefs to the target UIDs via `node.newUID`), then delete in reverse (`deleteGroup` → `deleteSourceObject`, which first sets `clusterctl.cluster.x-k8s.io/delete-for-move`).
6. Namespaces are created lazily in `createTargetObject`/`ensureNamespace`, not up front.
7. Resume: ClusterClasses then Clusters on the target. `patchTopologyManagedFields` re-plays topology-owned managed fields; `ResourceMutatorFunc`s let the E2E framework rewrite namespaces.

### What a new API type must do to be movable
- Be installed as part of provider components (so it carries the `clusterctl.cluster.x-k8s.io` label and is discoverable) and its **storage version** must be the one you expect to move.
- Either have an ownerRef chain that reaches a `Cluster` / `ClusterClass` / `ClusterResourceSet`, or be labelled `clusterctl.cluster.x-k8s.io/move` (object or CRD) — or `move-hierarchy` if its descendants should come too.
- Every owner must itself be discoverable, or discovery fails on a virtual node.
- Cluster-scoped types are copied, never deleted from the source — say so in your docs.
- If the object cannot be paused synchronously, set `clusterctl.cluster.x-k8s.io/block-move` and remove it once paused.
- Validating webhooks must tolerate deletion of objects annotated `clusterctl.cluster.x-k8s.io/delete-for-move`, and must tolerate creation of objects with a status already populated.
- Secrets you want moved need either an ownerRef or the `<cluster>-<purpose>` naming convention.

## Template processing and variables

- `yamlprocessor.Processor` (`processor.go`) abstracts template rendering; `SimpleProcessor` supports `${VAR}`, `${VAR:=default}`, `${VAR:-default}` via Go text/template parsing plus `convertLegacyVars`, and reports missing variables through `errMissingVariables`. `--config`/env/`clusterctl.yaml` supply values (`config/variables_client.go`).
- Component pipeline — `client/repository/components.go:NewComponents`: extract variables → substitute (unless `SkipTemplateProcess`) → parse to unstructured → apply image overrides → inspect required images → resolve target namespace (`inspectTargetNamespace`, at most one `Namespace` object) → `addNamespaceIfMissing` → `fixTargetNamespace` (including webhook/CRD-conversion/`Certificate` references: `fixMutatingWebhookNamespaceReferences`, `fixValidatingWebhookNamespaceReferences`, `fixCRDWebhookNamespaceReference`, `fixCertificate`) → `addCommonLabels` (`clusterctl.cluster.x-k8s.io: ""` + `cluster.x-k8s.io/provider: <manifest label>`) → stable sort putting `Namespace` first and `cert-manager.io` objects next so the CA injector sees Certificates before webhooks/CRDs.
- Cluster templates get variable substitution and namespace enforcement only.

## cert-manager — `client/cluster/cert_manager.go`
- Version and URL come from `config/cert_manager_client.go` (overridable in `clusterctl.yaml`); manifests resolve through the same repository machinery, so the `overrides/` folder works for cert-manager too.
- `EnsureInstalled` is a no-op if the namespace exists; `PlanUpgrade`/`EnsureLatestVersion` compare the `cert-manager.clusterctl.cluster.x-k8s.io/version` annotation and the images (`shouldUpgrade`, `certManagerImagesDiffer`), then delete + reinstall and `migrateCRDs`.
- Objects are tagged with `clusterctl.cluster.x-k8s.io/core: cert-manager` (`addCerManagerLabel`, `addCerManagerAnnotations`) so `delete`/`upgrade` can find them. `waitForAPIReady` applies `assets/cert-manager-test-resources.yaml`.
- `clusterctl upgrade plan` prints the cert-manager plan separately (`client.PlanCertManagerUpgrade`).

## Gotchas

- clusterctl only ever upgrades **to its own** contract version (`ApplyPlan` guard); mixed-contract management clusters are rejected at `installer.Validate`.
- Exactly one instance of each provider is supported (`CheckSingleProviderInstance`); multi-tenancy is via `--namespace`/`--watch-filter` on the provider, not multiple installs.
- CRD names must be `CalculateCRDName(group, kind)`; use `clusterctl.cluster.x-k8s.io/skip-crd-name-preflight-check` only for CRDs not referenced from core CAPI CRDs.
- `move` deletes from the source; `--dry-run` skips pausing and the provisioning gates, so a dry run proves much less than it looks.
- `delete` (`client/delete.go`, `client/cluster/components.go`) preserves CRDs and the namespace unless `--include-crd` / `--include-namespace`; deleting CRDs deletes the workload clusters' CRs.
- Discovery only sees the **storage** version of each CRD, and only CRDs carrying the clusterctl label — CRDs applied by hand are invisible to `move`.
- `cmd/version_checker.go` reaches out to GitHub (honouring `GOPROXY` via `goproxy.GetSchemeAndHost`) to nag about newer clusterctl releases; it caches the result under the config folder, so a stale cache can produce odd warnings.
- Adding a provider to the built-in list also needs a docs entry; see `docs/book/src/developer/providers/contracts/clusterctl.md` "Adding a provider to clusterctl".
- All cluster access funnels through `client/cluster/proxy.go:Proxy` (`GetConfig`, `NewClient`, `CurrentNamespace`, `ValidateKubernetesVersion`, `CheckClusterAvailable`, `ListResources`). `ListResources` deliberately excludes each component's own CRs so listing does not fail against a scaled-down provider — a common source of confusion when debugging `delete`/`upgrade`.
- `client/cluster/ownergraph.go:GetOwnerGraph` reuses the move discovery machinery to expose the ownerRef graph to the E2E framework (`FilterClusterObjectsWithNameFilter`); the ownerRef assertions in `test/e2e` depend on it, so changing discovery can break seemingly unrelated E2E tests.
- `clusterctl get kubeconfig` reads the `<cluster>-kubeconfig` secret through `client/cluster/workload_cluster.go:GetKubeconfig`, so it only works once the control-plane provider has created it.

## Tests

- Unit tests sit next to the code: `client/cluster/{objectgraph,mover,upgrader,installer,components,cert_manager,inventory,template}_test.go`, `client/repository/*_test.go`, `client/config/*_test.go`, `client/{init,move,upgrade,describe}_test.go`, `cmd/*_test.go`.
- Fixtures: `cmd/clusterctl/internal/test/fake_objects.go` builds synthetic Cluster/Machine/provider graphs and is the fastest way to add a `move` test case; `client/repository/repository_memory.go` and `client/config/reader_memory.go` give in-memory repositories/config.
- E2E: `test/e2e/clusterctl_upgrade.go` (+`_test.go`) exercises init/upgrade across releases, `test/e2e/self_hosted.go` exercises `move` into a self-hosted management cluster. The framework drives clusterctl as a library through `test/framework/clusterctl/`.

## Where to read more

- `docs/book/src/clusterctl/overview.md`, `configuration.md`, `developers.md`, `plugins.md`
- `docs/book/src/clusterctl/commands/` — `init.md`, `upgrade.md`, `move.md`, `generate-cluster.md`, `generate-yaml.md`, `generate-provider.md`, `describe-cluster.md`, `delete.md`, `get-kubeconfig.md`, `convert.md`, `alpha-rollout.md`, `additional-commands.md`
- `docs/book/src/developer/providers/contracts/clusterctl.md` — the normative provider-side contract (repositories, metadata, components, labels, ownerReferences chain, move rules)
- `docs/proposals/20191016-clusterctl-redesign.md`, `docs/proposals/20200511-clusterctl-extensible-template-processing.md`
- `cmd/clusterctl/README.md`

# CAPI APIs, Conversion, Webhooks and Code Generation

Operational guide to the `api/` Go module: which groups/versions exist and which is storage, how the v1beta1↔v1beta2 conversion machinery is wired (it is **not** the vanilla kubebuilder layout), where admission webhooks live after the 2026 code reorg, the metav1.Condition-based conditions API, and exactly which `make` targets regenerate which files. Read this before editing anything under `api/`, `*/webhooks/`, or any `zz_generated.*` file.

## The `api/` module

- Separate module `sigs.k8s.io/cluster-api/api` (`api/go.mod`) so providers/consumers importing CAPI types don't pull CAPI's whole dep tree. Its `require` block is deliberately tiny (k8s.io/api, apimachinery, apiextensions-apiserver, cluster-bootstrap).
- Root `go.mod:5`: `replace sigs.k8s.io/cluster-api/api => ./api`. `test/go.mod` replaces both. Adding a dep to `api/go.mod` is review-sensitive.
- It carries the strongest compatibility guarantee in the repo (K8s API deprecation policy). The root module is best-effort.

| Path | Group | Versions present |
|---|---|---|
| `api/core/` | `cluster.x-k8s.io` | `v1beta1`, `v1beta2` |
| `api/bootstrap/kubeadm/` | `bootstrap.cluster.x-k8s.io` | `v1beta1`, `v1beta2` |
| `api/controlplane/kubeadm/` | `controlplane.cluster.x-k8s.io` | `v1beta1`, `v1beta2` |
| `api/addons/` | `addons.cluster.x-k8s.io` | `v1beta1`, `v1beta2` |
| `api/ipam/` | `ipam.cluster.x-k8s.io` | `v1alpha1`, `v1beta1`, `v1beta2` |
| `api/runtime/` | `runtime.cluster.x-k8s.io` | `v1alpha1`, `v1beta2` (ExtensionConfig) |
| `api/runtime/hooks/v1alpha1` | — | Runtime SDK request/response types (not CRDs) |
| `api/runtime/catalog` | — | Runtime hook catalog (`catalog.go`, `builder.go`); KAL-exempt |
| `api/deprecated/errors` | — | legacy `*Error`/`FailureReason` constants |

Not in this module: CAPD types live in `test/infrastructure/docker/api/` (in the `test` module), clusterctl config types in `cmd/clusterctl/api/v1alpha3`.

## Version state of the world

- **v1beta2 is the hub and the storage version everywhere.** Confirm with `grep -rn "kubebuilder:storageversion" api/` — every hit is a `v1beta2` file (plus `api/runtime/v1beta2/extensionconfig_types.go`).
- **v1beta1 is deprecated but still served.** Signals:
  - `// +kubebuilder:deprecatedversion` on each v1beta1 root type (e.g. `api/core/v1beta1/machine_types.go:691`).
  - package-level `Deprecated:` note in `api/core/v1beta1/doc.go`.
  - `// +k8s:conversion-gen=sigs.k8s.io/cluster-api/api/core/v1beta2` in `api/core/v1beta1/doc.go` — this marker is what makes conversion-gen target v1beta2 as hub.
  - Served versions are visible in the CRDs, e.g. `core/config/crd/bases/cluster.x-k8s.io_machines.yaml`.
- `api/core/v1beta2/groupversion_info.go` also exports `GroupVersionInfrastructure` / `GroupVersionBootstrap` / `GroupVersionControlPlane` — the *recommended* GVs for out-of-tree providers.
- Storage-version migration of existing objects at runtime is done by `controllers/crdmigrator/crd_migrator.go`, not by the apiserver.
- Per-release API/provider changes: `docs/book/src/developer/providers/migrations/v1.14-to-v1.15.md` (and the v1.12/v1.13 siblings).

## Conversion: hub-and-spoke, two different styles

### Style A — `api/` module types (core, CABPK, KCP, addons, ipam, runtime)

Conversion logic is implemented **outside** the API types, because the `api/` module must not depend on controller-runtime. Three layers:

1. `api/<group>/<oldversion>/zz_generated.conversion.go` — generated `autoConvert_*` functions. Never hand-edit.
2. `api/<group>/<oldversion>/conversion.go` — hand-written `Convert_v1beta1_X_To_v1beta2_X` / `Convert_v1beta2_X_To_v1beta1_X` that call the `autoConvert_*` counterpart then fix up fields conversion-gen can't map. Defining a `Convert_…` with the exact expected name makes conversion-gen emit `autoConvert_…` instead and delegate to yours.
3. `<provider>/webhooks/conversion/<kind>.go` — the actual controller-runtime converter plus restore logic. E.g. `core/webhooks/conversion/cluster.go`:
   ```go
   var Cluster = conversion.NewHubSpokeConverter(&clusterv1.Cluster{},
       conversion.NewSpokeConverter(&clusterv1beta1.Cluster{}, ConvertClusterHubToV1Beta1, ConvertClusterV1Beta1ToHub))
   ```
   Registered on the webhook via `.WithConverter(conversion.MachineDeployment)` in each `SetupWebhookWithManager` (see `core/webhooks/admission/machinedeployment.go:54`).

- **Lossy-conversion annotation**: `util/conversion/conversion.go:DataAnnotation` = `cluster.x-k8s.io/conversion-data`. Hub→spoke calls `MarshalDataUnsafeNoCopy` (or `MarshalData`) with a *pruned* hub object containing only the fields that will be restored — see the comment "Only put the fields into the conversion annotation that are actually restored" in `core/webhooks/conversion/cluster.go`. Spoke→hub calls `UnmarshalData` into a `restored` object and re-applies those fields.
- **Intent-recovery helpers** in `api/core/v1beta2/conversion.go`: `Convert_bool_To_Pointer_bool`, `Convert_int32_To_Pointer_int32`, `Convert_Duration_To_Pointer_int32`, `ConvertToSeconds`, `ConvertFromSeconds`. Used because v1beta2 moved many `bool`/`Duration` fields to `*bool` / `*int32` seconds.
- **Ref conversion needs a live client**: `v1beta2` uses `ContractVersionedObjectReference` (apiGroup+kind, no version). Going hub→spoke needs the served apiVersion, supplied by `core/webhooks/conversion/common.go:SetAPIVersionGetter` (wired in `core/main.go:753` via `contract.GetGVKFromGK`; tests use `SwapAPIVersionGetter`, and `internal/test/envtest/environment.go:415,422` wires it for envtest).
- `dropEmptyString` / `dropEmptyStrings*` helpers exist purely to make round-trips exact.

### Style B — CAPD (`test/infrastructure/docker/api/`)

Classic kubebuilder: `v1beta2` types implement `Hub()` (`test/infrastructure/docker/api/v1beta2/conversion.go`) and `v1beta1` types implement `ConvertTo`/`ConvertFrom` (`test/infrastructure/docker/api/v1beta1/conversion.go`). This is the only place `make verify-conversions` runs (`hack/tools/conversion-verifier`, which checks every `+kubebuilder:storageversion` type has `Hub()` and every other version has `ConvertTo`/`ConvertFrom`). The Makefile comment says explicitly it is CAPD-only "because api/ is implementing conversion outside of the api/ package".

### Fuzz round-trip tests

- Helpers: `util/conversion/conversion.go:FuzzTestFunc` (style B) and `util/conversion/conversion.go:SpokeConverterFuzzTestFunc` (style A). Both run `spoke-hub-spoke` and `hub-spoke-hub` 10000 iterations each, compare with `apiequality.Semantic.DeepEqual`, and strip `DataAnnotation` before comparing.
- `hub-spoke-hub` additionally converts a spoke copy **with no spec** to prove `ConvertTo` can't panic on the objects the apiserver sends during managedFields conversion.
- Per-type `FuzzerFuncs` constrain the filler where exact round-trip is impossible (see `core/webhooks/conversion/conversion_test.go`).
- All fuzz test files are `//go:build !race` — they'd time out under the race detector. Hence:
  ```bash
  make test-conversions                        # runs -run "^TestFuzzyConversion$" ./...
  make test-infrastructure-conversions         # same, in test/infrastructure
  go test -run "^TestFuzzyConversion$" ./core/webhooks/conversion/...   # single package
  ```
  `make test` runs the normal suite with `-race` then calls `test-conversions` separately.
- Test locations: `core/webhooks/conversion/conversion_test.go`, `bootstrap/kubeadm/webhooks/conversion/conversion_test.go`, `controlplane/kubeadm/webhooks/conversion/conversion_test.go`, `bootstrap/kubeadm/pkg/types/upstreamv1beta4/conversion_test.go`, `test/infrastructure/docker/api/v1beta1/conversion_test.go`, `internal/topology/upgrade/test/t2/v1beta1/conversion_test.go`.

## Webhooks

`internal/webhooks/` and `*/internal/webhooks/` **no longer exist** (2026 reorg, `docs/proposals/20260701-code-organization.md`). Current layout:

| Provider | Admission (defaulting+validation) | Conversion |
|---|---|---|
| core | `core/webhooks/admission/` | `core/webhooks/conversion/` |
| CABPK | `bootstrap/kubeadm/webhooks/admission/` | `bootstrap/kubeadm/webhooks/conversion/` |
| KCP | `controlplane/kubeadm/webhooks/admission/` | `controlplane/kubeadm/webhooks/conversion/` |
| CAPD | `test/infrastructure/docker/webhooks/admission/` | (types-based, see Style B) |

Conventions, from `core/webhooks/admission/machinedeployment.go`:

- One struct per Kind, implementing `admission.Defaulter[*clusterv1.X]` and `admission.Validator[*clusterv1.X]` (typed controller-runtime generics), asserted with `var _ admission.Defaulter[...] = &X{}`.
- `SetupWebhookWithManager(mgr)` uses `ctrl.NewWebhookManagedBy(mgr, &clusterv1.X{}).WithDefaulter(webhook).WithValidator(webhook).WithConverter(conversion.X).Complete()` — defaulting, validation and conversion for a Kind are registered in **one** call.
- `+kubebuilder:webhook:...` markers sit above the struct and drive `core/config/webhook/manifests.yaml`. Paths are `/validate-cluster-x-k8s-io-v1beta2-<kind>` and `/mutate-…`, `matchPolicy=Equivalent`, `sideEffects=None`, and **`versions=v1beta2` only** (older versions reach it through conversion).
- Access to the raw AdmissionRequest (needed for `DryRun`, `req.Operation`, decoding `OldObject`) is via `admission.RequestFromContext(ctx)` + an `admission.Decoder` field lazily built in `SetupWebhookWithManager`.
- Immutability: validation compares old vs new and returns `apierrors.NewInvalid` with `field.Path` entries; grep `field.Invalid(.*immutable` for the house phrasing.
- Feature-gate-guarded validation: webhooks (not just controllers) reject objects whose feature gate is off — ClusterClass/Cluster.topology behind `feature.ClusterTopology`, MachinePool behind `feature.MachinePool`, ClusterResourceSet(Binding) behind `feature.ClusterResourceSet`, ExtensionConfig behind `feature.RuntimeSDK`. See the `NOTE:` comments in `core/main.go:751` onward and `feature/feature.go`.
- Registration is a flat list of `SetupWebhookWithManager` calls in each provider's `main.go` (`core/main.go:setupWebhooks`, `bootstrap/kubeadm/main.go:341`, `controlplane/kubeadm/main.go:454`, `test/infrastructure/docker/main.go:489`). Adding a webhook = new file + one call here.

## Conditions API

- v1beta2 conditions are plain `[]metav1.Condition` on `status.conditions`, with the house marker set (`api/core/v1beta2/machine_types.go:566`):
  ```go
  // +optional
  // +listType=map
  // +listMapKey=type
  // +kubebuilder:validation:MaxItems=32
  Conditions []metav1.Condition `json:"conditions,omitempty"`   // machine_types.go:576
  ```
- Type/reason constants: `api/core/v1beta2/condition_consts.go` (shared: `AvailableCondition`, `ReadyCondition`, `UpToDateCondition`, `RollingOutCondition`, `ScalingUp/DownCondition`, `RemediatingCondition`, `DeletingCondition`, `PausedCondition`, …) plus per-kind files. Same pattern under `api/bootstrap/kubeadm/v1beta2/`, `api/controlplane/kubeadm/v1beta2/`, `api/addons/v1beta2/`.
- The **old** `clusterv1.Condition` type (with `Severity`) still exists in `api/core/v1beta2/condition_types.go` but every symbol is marked `Deprecated:`; it is only used inside `status.deprecated.v1beta1.conditions`. Old constants were renamed with a `V1Beta1` infix and live in `api/core/v1beta2/v1beta1_condition_consts.go` (`ReadyV1Beta1Condition`, `InfrastructureReadyV1Beta1Condition`, …).
- Each type therefore has four accessors: `GetConditions`/`SetConditions` (metav1) and `GetV1Beta1Conditions`/`SetV1Beta1Conditions` (legacy, lazily allocating `Status.Deprecated.V1Beta1`) — see `api/core/v1beta2/machine_types.go:842-868`.
- Helper packages: **new** `util/conditions/` (`Get`, `Set`, `IsTrue/False/Unknown`, `NewSummaryCondition`/`SetSummaryCondition`, `NewMirrorCondition`/`SetMirrorCondition`, `NewPatch`/`Patch.Apply`, `UnstructuredGet`, `merge_strategies.go`, `aggregate.go`). **Old** `util/deprecated/v1beta1/conditions/` for the legacy type. Don't mix them.
- Background: `docs/proposals/20240916-improve-status-in-CAPI-resources.md` and `docs/proposals/20200506-conditions.md`.

## Code generation

| Target | Generates | Tool |
|---|---|---|
| `make generate` | everything below (`generate-modules generate-manifests generate-go-deepcopy generate-go-conversions generate-go-openapi generate-crd-docs`) | — |
| `make generate-manifests` | CRDs, RBAC, webhook manifests into `<provider>/config/...` | controller-gen |
| `make generate-go-deepcopy` | `zz_generated.deepcopy.go` | controller-gen |
| `make generate-go-conversions` | `zz_generated.conversion.go` | conversion-gen |
| `make generate-go-openapi` | `hack/tools/runtime-openapi-gen/**/zz_generated.openapi.go` for `api/core/v1beta2` + `api/runtime/hooks/v1alpha1` | openapi-gen |
| `make generate-crd-docs` | `docs/book/src/reference/api/crd-api-reference.md` (+ `-v1beta1.md`) | crd-ref-docs |
| `make generate-modules` | `go mod tidy` in root, `api`, `hack/tools`, `test` | go |

- Each family fans out over `ALL_GENERATE_MODULES = core kubeadm-bootstrap kubeadm-control-plane docker-infrastructure test-extension`, so `make generate-manifests-core`, `make generate-go-deepcopy-kubeadm-bootstrap`, `make generate-go-conversions-core-api`, etc. exist for fast iteration.
- Generators **delete before regenerating** (`clean-generated-yaml`, `clean-generated-deepcopy`, `clean-generated-conversions`, `clean-generated-openapi-definitions`), so a stale file won't survive a rename.
- Never hand-edit: `zz_generated.deepcopy.go`, `zz_generated.conversion.go`, `zz_generated.openapi.go`, anything in `*/config/crd/bases/`, `*/config/rbac/role.yaml`, `*/config/webhook/manifests.yaml`, `docs/book/src/reference/api/crd-api-reference*.md`.
- `make generate-go-conversions-core-api` runs conversion-gen from *inside* `api/` (`cd api; $(CONVERSION_GEN) … ./core/v1beta1`) — running it from the repo root will not work.

### verify targets that gate CI

`ALL_VERIFY_CHECKS = licenses boilerplate shellcheck tiltfile modules gen crd-docs conversions doctoc capi-book-summary diagrams go-directive`; `make verify` runs all of them plus `lint-dockerfiles`. The ones that bite:

- `make verify-gen` — runs `make generate` then fails on any `git diff`.
- `make verify-modules` — `go mod tidy` across all four modules; also rejects `client-go …+incompatible`.
- `make verify-crd-docs`, `make verify-conversions` (CAPD only), `make verify-boilerplate` (new files need the Apache header from `hack/scripts/verify/boilerplate/`).
- `make apidiff` (`APIDIFF_OLD_COMMIT ?= origin/main`) to see what you broke.

### Linting: `.golangci.yml` + `.golangci-kal.yml`

- `make lint` runs golangci-lint in root, `api`, `test`, `hack/tools`, then the KAL (kube-api-linter) custom build over `api/` only. `make lint-api` / `make lint-api-fix` run just KAL; `make lint-fix` adds `--fix`. CI: `.github/workflows/pr-golangci-lint.yaml` has a `lint-api` job plus a `lint` matrix over those four working directories.
- `.golangci.yml` notables: `importas` pins aliases — `clusterv1` = `api/core/v1beta2`, `clusterv1beta1` = `api/core/v1beta1`, `bootstrapv1`/`controlplanev1`/`addonsv1`/`ipamv1`/`runtimev1` = the v1beta2 packages, `coreadmission`/`bootstrapadmission`/`controlplaneadmission` for the webhook packages, `pkgerrors` for `github.com/pkg/errors`, `ctrl`, `ctrlcache`, `kerrors`, `logsv1`. Also `gci`/`goimports` import grouping, `godot`, `ginkgolinter`, `revive`, `nolintlint`.
- `.golangci-kal.yml` enables the `kubeapilinter` module on `api/**` (excluding `api/runtime/catalog`, `api/deprecated`, all `zz_generated.*`, all `_test.go`). Enabled rules: `commentstart`, `conditions`, `conflictingmarkers`, `duplicatemarkers`, `forbiddenmarkers`, `integers`, `jsontags`, `maxlength`, `nobools`, `nodurations`, `nofloats`, `nomaps`, `nonullable`, `notimestamp`, `optionalfields`, `optionalorrequired`, `requiredfields`, `ssatags`, `statusoptional`, `statussubresource`, `uniquemarkers`. Practical consequences for new fields:
  - Every field needs `+optional` or `+required`; comment must start with the JSON field name.
  - `kubebuilder:default` / `default` are **forbidden markers** — do defaulting in the webhook, not the schema.
  - No `bool` (use an enum), no `metav1.Duration` (use `…Seconds *int32`), no floats, no maps, no `*Timestamp` names, only `int32`/`int64`.
  - Strings/arrays need `MaxLength`/`MaxItems`; array fields need `+listType`.
  - Optional fields: pointer only `WhenRequired`, plus `omitempty`/`omitzero`; required fields must not be pointers and must not have `omitempty`.
  - `conditions` rule: `Conditions` must be first in status, no `patchStrategy`, no protobuf tags.
  - All the existing carve-outs are enumerated in `.golangci-kal.yml` `exclusions.rules` — read them before assuming a violation is acceptable; v1beta1 packages are fully excluded because they are frozen.
  - `nophase` is intentionally left disabled (phase fields are kept).

## Recipe: add a field to an API type

1. Add it to the **v1beta2** struct only, e.g. `api/core/v1beta2/machine_types.go`. v1beta1 is frozen.
2. Write the markers: doc comment starting with the json name, `+optional`/`+required`, `+kubebuilder:validation:MaxLength|MaxItems|Minimum|Enum`, `+listType`/`+listMapKey` for lists, `omitempty` (+ `omitzero` for structs). No `kubebuilder:default`.
3. Obey KAL types: seconds-as-`*int32` instead of `Duration`, enum string instead of `bool`, list-with-name instead of map.
4. If the field must survive a v1beta1 round trip, decide the story:
   - Mappable to an existing v1beta1 field → nothing extra, conversion-gen handles it.
   - Not representable in v1beta1 → add it to the pruned hub object marshalled in `<provider>/webhooks/conversion/<kind>.go` (`MarshalDataUnsafeNoCopy`) and restore it from `UnmarshalData` in the spoke→hub direction.
   - Shape changed (bool→*bool, Duration→seconds) → add a hand-written `Convert_…` in `api/core/v1beta1/conversion.go` and use the helpers in `api/core/v1beta2/conversion.go`.
5. `make generate-go-deepcopy-core && make generate-go-conversions-core-api` (or just `make generate`).
6. Defaulting/validation → `core/webhooks/admission/<kind>.go` (`Default`/`ValidateCreate`/`ValidateUpdate`), including immutability and any feature-gate guard; add table tests in `<kind>_test.go`.
7. `make generate-manifests-core` to refresh CRD/RBAC/webhook YAML; `make generate-crd-docs` for the book.
8. Tests: extend the per-kind `FuzzerFuncs` in `core/webhooks/conversion/conversion_test.go` if the field can't round-trip verbatim, then `make test-conversions`.
9. Builders: if tests construct the object, add a `With…` to `util/test/builder/builders.go`.
10. `make lint-api` (KAL) then `make lint`, `make test`, `make verify-gen verify-modules verify-crd-docs`.
11. If a controller must react to it, also touch `core/reconcilers/<kind>/` and, for topology-managed fields, the ClusterClass/desired-state code.

## Where to read more

- `docs/book/src/developer/core/API.md` — API guidelines and CAPI-specific considerations.
- `docs/book/src/developer/providers/getting-started/implement-api-types.md`, `.../webhooks.md` — the marker/webhook conventions as taught to provider authors.
- `docs/book/src/developer/providers/migrations/v1.14-to-v1.15.md` (+ `v1.13-to-v1.14.md`, `v1.12-to-v1.13.md`, `migrations/overview.md`) — what changed per release, including the v1beta1→v1beta2 API churn.
- `docs/book/src/developer/core/repository-layout.md` — stale on directory names; prefer `docs/proposals/20260701-code-organization.md`.
- `docs/proposals/20240916-improve-status-in-CAPI-resources.md`, `docs/proposals/20200506-conditions.md` — conditions/status design.
- `docs/book/src/reference/api/crd-api-reference.md` — generated field reference.

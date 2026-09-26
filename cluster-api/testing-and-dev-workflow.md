# CAPI Testing Layers and the Local Dev Loop

How tests are organised in this repo (fake-client unit tests, envtest integration tests, Ginkgo e2e against real kind clusters), the exact `make` targets and env vars that drive them, the helper packages you are expected to use instead of hand-building objects, and the Tilt-based inner loop. Ends with the minimal pre-push checklist and how it maps to CI. Read this before writing or running tests here.

## The three layers

| Layer | Where | Needs | Build tag |
|---|---|---|---|
| Unit | `*_test.go` next to the code; `client/fake` from controller-runtime | nothing | none |
| Integration (envtest) | `*_test.go` in packages that have a `suite_test.go` with `TestMain` | kube-apiserver + etcd binaries | none |
| Fuzz round-trip | `*/webhooks/conversion/conversion_test.go`, `test/infrastructure/docker/api/v1beta1/conversion_test.go` | nothing | `//go:build !race` |
| E2E (Ginkgo) | `test/e2e/` (in the `test` module) | docker, kind, built images | `//go:build e2e` |

Four Go modules each have their own test target: root, `api/`, `test/` (subdivided into `test/infrastructure`, `test/extension`, `test/framework`), `hack/tools/`.

## Running tests

```bash
make test                 # root + api module, -race, then make test-conversions
make test-no-race         # same without -race (much faster; fuzz tests included)
make test-verbose         # make test with -v
make test-cover           # coverage.out/.txt/.html for root and api
make test-conversions     # only -run "^TestFuzzyConversion$" ./...
make test-infrastructure  # test/infrastructure (CAPD, in-memory), -race + conversions
make test-test-extension  # test/extension
make test-junit           # what CI runs: -json + gotestsum junit into $(ARTIFACTS)
make setup-envtest        # download kubebuilder assets, print KUBEBUILDER_ASSETS
```

- `-race` is the default for `make test`. Fuzz conversion files carry `//go:build !race` because they time out under the race detector, which is why `make test` invokes `test-conversions` as a separate pass.
- `TEST_ARGS` is threaded into every `go test` invocation — this is how you pass `-run`, `-v`, `-count=1`:
  ```bash
  make test TEST_ARGS="-run TestMachineDeploymentDefault -v"
  make test-no-race TEST_ARGS="-count=1"
  ```
- For a single package, skip make and set the assets yourself:
  ```bash
  export KUBEBUILDER_ASSETS=$(make --no-print-directory setup-envtest | sed 's/KUBEBUILDER_ASSETS=//')
  go test ./core/reconcilers/machineset/... -run TestMachineSetReconciler -v
  go test -run "^TestFuzzyConversion$" ./core/webhooks/conversion/...
  ```
  `KUBEBUILDER_ASSETS ?= $(shell $(SETUP_ENVTEST) use --use-env -p path $(KUBEBUILDER_ENVTEST_KUBERNETES_VERSION))`; the version is pinned at `Makefile:47` (`KUBEBUILDER_ENVTEST_KUBERNETES_VERSION ?= 1.37.0`).
- The `api/` module is a separate `go test` run (`cd api; go test ./...`). Tests there never need envtest.

### envtest specifics

- Shared harness: `internal/test/envtest/environment.go`. Every integration suite has a `suite_test.go` like `controllers/crdmigrator/suite_test.go` or `core/reconcilers/machine/suite_test.go`:
  ```go
  var (
      env *envtest.Environment
      ctx = ctrl.SetupSignalHandler()
  )

  func TestMain(m *testing.M) {
      os.Exit(envtest.Run(ctx, envtest.RunInput{
          M:                m,
          SetupReconcilers: func(ctx context.Context, mgr ctrl.Manager) { /* register controllers */ },
          SetupIndexes:     setupIndexes,   // local func, see core/reconcilers/machine/suite_test.go:122
          SetupEnv:         func(e *envtest.Environment) { env = e },
      }))
  }
  ```
  `envtest.RunInput` fields: `M`, `SetupManagerCacheOptions`, `ManagerClientOptions`, `SetupIndexes`, `SetupReconcilers`, `SetupEnv`, `MinK8sVersion`, `AdditionalSchemeBuilder`, `AdditionalCRDDirectoryPaths`.
- `envtest.Run` installs all CAPI CRDs, starts the webhooks (`internal/test/envtest/webhooks.go`), waits for them, and wires `SetAPIVersionGetter` for the conversion webhooks. It also runs `verifyPanicMetrics` at the end, so a panicking reconciler fails the suite.
- Feature gates default off. Suites that need them flip them in `TestMain` — see `internal/test/envtest/example/suite_test.go`, which is also the documented way to attach a debugger to the embedded apiserver.
- `Environment` helpers to prefer over raw client calls: `CreateNamespace`, `CreateAndWait`, `PatchAndWait`, `DeleteAndWait`, `Cleanup`, `CleanupAndWait`, `CreateKubeconfigSecret`, `ApplyCRDs`. They handle cache-sync races.
- Env vars that change behaviour:

| Var | Effect |
|---|---|
| `CAPI_DISABLE_TEST_ENV` | skip starting the test env entirely (fake-client-only tests) |
| `CAPI_TEST_ENV_KUBECONFIG` | write a kubeconfig for the running test env to a path |
| `CAPI_TEST_ENV_SKIP_STOP` | leave the test env running after the suite |
| `CAPI_TEST_ENV_LOG_LEVEL` | klog verbosity |
| `USE_EXISTING_CLUSTER=true` | run against your current kubectl context instead of a local apiserver |
| `ARTIFACTS` | if set, apiserver audit logs are written there |

  For a faster loop use a long-lived kind cluster instead of starting an apiserver per package: `./hack/scripts/dev/kind-create-for-envtest.sh`, then `USE_EXISTING_CLUSTER=true`.

## Test helper conventions

- **Object builders: `util/test/builder/`** (there is no `internal/test/builder`). Fluent `Cluster(ns, name).WithTopology(...).Build()`, plus `ClusterClass`, `MachineDeployment`, `MachineSet`, `MachinePool`, `ClusterTopology`, `MachineDeploymentTopology` and the generic infrastructure/bootstrap/controlplane unstructured builders in `infrastructure.go`, `bootstrap.go`, `controlplane.go`, `remediation.go`. `crds.go` provides matching test CRDs (`GenericInfrastructureClusterCRD`, …) to install into envtest. `v1beta2_transition.go` has `Phase0Obj`/`Phase1Obj`/… synthetic types used to test the conditions migration.
- `util/deprecated/v1beta1/test/builder/` mirrors it for the legacy API; `util/deprecated/v1beta1/conditions` has matchers for old-style conditions.
- Condition assertions on the new API: `util/conditions/matcher.go` (`conditions.MatchCondition`, `MatchConditions`).
- **E2E framework: `test/framework/`** — `ClusterProxy` (`cluster_proxy.go`) is the handle for a management or workload cluster; `bootstrap/` creates the kind bootstrap cluster; `clusterctl/` wraps clusterctl (`e2e_config.go` parses the config file, `repository.go` builds the local clusterctl repository, `clusterctl_helpers.go` has `ApplyClusterTemplateAndWait` etc.); the `*_helpers.go` files hold the `WaitFor…`/`Discover…` assertions; `ownerreference_helpers.go` holds the owner-reference assertion tables used by the quick-start spec; `ginkgoextensions/` and `internal/log/` handle logging.
- **Shared e2e specs**: each `test/e2e/<name>.go` exports `XxxSpec(ctx, inputGetter func() XxxSpecInput)` and the matching `<name>_test.go` is a thin `Describe` that fills the input. Examples: `test/e2e/quick_start.go:QuickStartSpec`, `test/e2e/clusterctl_upgrade.go:ClusterctlUpgradeSpec` (+ `ClusterctlUpgradeSpecInput`, `ClusterctlUpgradeSpecInputUpgrade`), `cluster_upgrade.go`, `self_hosted.go`, `md_scale.go`, `scale.go`. Providers out of tree import these specs from the `test` module — so changing a `*SpecInput` struct is a breaking change for them.
- **Config and templates**: `test/e2e/config/docker.yaml` declares providers, images, `variables:` (all overridable by env var) and `intervals:` (the `["5m","10s"]` timeout/poll pairs consumed by `e2eConfig.GetIntervals`). Cluster templates are kustomize overlays under `test/e2e/data/infrastructure-docker/{main,v1.12,v1.13,v1.14}/` plus `test/e2e/data/{shared,cni,autoscaler,kubetest}`. Regenerate the flattened YAML with:
  ```bash
  make generate-e2e-templates      # kustomize build -> cluster-template*.yaml
  ```
  `make test-e2e` depends on it, so it runs automatically.

## Gomega / Ginkgo conventions

- Unit and envtest tests are **plain `go test` + Gomega**, not Ginkgo. Always `g := NewWithT(t)` — including inside each table-test subtest, never a package-level `Expect`:
  ```go
  for _, tt := range tests {
      t.Run(tt.name, func(t *testing.T) {
          g := NewWithT(t)
          g.Expect(webhook.Default(ctx, tt.md)).To(Succeed())
      })
  }
  ```
  When a closure needs its own Gomega (inside `Eventually`), take `g Gomega` as the parameter.
- `Eventually`/`Consistently` in envtest suites use package-level constants defined in `suite_test.go` — `timeout = time.Second * 30` (15s in `core/reconcilers/clusterresourceset/`) with the interval left to the Gomega default. Follow the local file's constants rather than inventing new durations.
- `ginkgolinter` is enabled in `.golangci.yml`, so `Expect(err).ToNot(HaveOccurred())`, `Expect(x).To(BeEmpty())` style is enforced; it will reject `Expect(len(x)).To(Equal(0))` and friends. Dot-imports of ginkgo/gomega are allowed only in test files and test utils (`.golangci.yml` exclusions).
- E2E specs: `Describe("When ...", Label("PR-Blocking", "ClusterClass"), func(){...})`. The bracketed tags in the description are duplicated as real Ginkgo `Label`s — keep both in sync. `test/e2e/e2e_suite_test.go` owns the `SynchronizedBeforeSuite` that creates the bootstrap cluster, builds the local clusterctl repository, and `clusterctl init`s providers; flags are `--e2e.config`, `--e2e.artifacts-folder`, `--e2e.skip-resource-cleanup`, `--e2e.use-existing-cluster`, `--e2e.clusterctl-config`, `--e2e.also-log-to-file`.

## Running e2e

```bash
make docker-build-e2e                 # build all manager images with e2e tags
make tilt-e2e-prerequisites           # build kindest/node images + generate-e2e-templates
make test-e2e                         # full suite (GINKGO_TIMEOUT default 3h)
GINKGO_LABEL_FILTER="PR-Blocking" make test-e2e
GINKGO_LABEL_FILTER="!K8s-Upgrade" make test-e2e
GINKGO_FOCUS="When upgrading a workload cluster" make test-e2e
GINKGO_SKIP="Conformance" GINKGO_NODES=3 make test-e2e
SKIP_RESOURCE_CLEANUP=true make test-e2e     # leave clusters around to debug
make e2e-framework                    # just compile test/framework
```

Knobs (all `?=` in the Makefile, so env-settable): `GINKGO_FOCUS`, `GINKGO_SKIP` (each word becomes a `-skip=`), `GINKGO_LABEL_FILTER`, `GINKGO_NODES` (default 1), `GINKGO_TIMEOUT` (3h), `GINKGO_ARGS`, `GINKGO_NOCOLOR`, `GINKGO_POLL_PROGRESS_AFTER`/`_INTERVAL`, `E2E_CONF_FILE` (default `test/e2e/config/docker.yaml`), `ARTIFACTS` (default `_artifacts`), `SKIP_RESOURCE_CLEANUP`, `USE_EXISTING_CLUSTER`. Anything in the `variables:` block of the config file can also be overridden by env var. Prefer `GINKGO_LABEL_FILTER` over `GINKGO_FOCUS`: labels in use include `PR-Blocking`, `K8s-Upgrade`, `Conformance`, `ClusterClass`, `IPv6`.

`hack/scripts/ci/ci-e2e.sh` is what Prow runs; reading it is the fastest way to reproduce a failing CI e2e job locally.

## Local dev loop (Tilt)

```bash
make kind-cluster        # hack/scripts/dev/kind-create-for-capd.sh: kind + local registry + docker socket mount
make tilt-up             # kind-cluster, then `tilt up`
make kind-cluster-kubevirt   # variant with KubeVirt for CAPK work
```

- `Tiltfile` reads `./tilt-settings.yaml` (preferred) or `./tilt-settings.json` from the repo root — not committed, you create it. Minimum useful content:
  ```yaml
  default_registry: gcr.io/your-project      # required unless the kind cluster has a local registry
  enable_providers:
  - docker
  - kubeadm-bootstrap
  - kubeadm-control-plane
  ```
  Other fields: `allowed_contexts`, `kind_cluster_name` (default `capi-test`, also `CAPI_KIND_CLUSTER_NAME`), `provider_repos` (paths to out-of-tree providers that ship a `tilt-provider.yaml`), `kustomize_substitutions`, `template_dirs`, `trigger_mode: manual`, `build_engine` (auto-detects podman), and `debug`.
- `core` is always enabled; `enable_providers` adds `kubeadm-bootstrap`, `kubeadm-control-plane`, `docker`, `test-extension`, or your own repos. Each entry in the `providers` map in `Tiltfile` declares `context`, `image`, `live_reload_deps` and `label` (CAPI / CABPK / KCP / CAPD).
- **Iterating on one controller**: edit anything under that provider's `live_reload_deps` (e.g. for core: `core/reconcilers`, `core/webhooks`, `core/main.go`, `../api`, `../internal`, `../util`) and Tilt rebuilds just that binary and live-updates the pod. Changing `api/` reloads core + whichever provider lists that API dir.
- Per-provider `debug` settings in `tilt-settings.yaml` give you a delve port, `metrics_port`, `profiler_port` (exposed as a `/debug/pprof` link in the Tilt UI) and `race_detector: true`. Ports are forwarded by `get_port_forwards` in the `Tiltfile`.
- CRDs/manifests are applied by `hack/tools/internal/tilt-prepare` (invoked from the `Tiltfile`), so a CRD change needs `make generate-manifests` before Tilt will pick it up.
- `make tiltfile-fix` / `make verify-tiltfile` format and check the Starlark.
- Book preview while editing docs: `make serve-book`.

## Pre-submit checklist

Minimum before pushing:

```bash
make generate            # or the narrow generate-* targets you need
make lint                # + make lint-api if you touched api/
make test                # add make test-infrastructure if you touched test/infrastructure
make verify              # licenses boilerplate shellcheck tiltfile modules gen crd-docs
                         # conversions doctoc capi-book-summary diagrams go-directive + lint-dockerfiles
```

Faster partial runs: `make verify-gen verify-modules` catches the two most common CI failures; `make lint-fix` / `make lint-api-fix` auto-fix what they can.

| Local command | CI job |
|---|---|
| `make lint`, `make lint-api` | `.github/workflows/pr-golangci-lint.yaml` (`lint` matrix over ``, `api`, `test`, `hack/tools`; separate `lint-api` job) |
| `make verify*` | Prow `pull-cluster-api-verify` via `hack/scripts/ci/ci-verify.sh` |
| `make test-junit`, `make test-infrastructure-junit`, `make test-test-extension-junit`, `make test-framework-junit` | Prow test jobs via `hack/scripts/ci/ci-test.sh` |
| `make test-e2e` with `GINKGO_LABEL_FILTER` | Prow e2e jobs via `hack/scripts/ci/ci-e2e.sh` |
| `make apidiff` | `hack/scripts/ci/ci-apidiff.sh` |
| PR title convention | `.github/workflows/pr-verify.yaml` → `hack/scripts/verify/verify-pr-title.sh` |

Other GitHub workflows (`pr-md-link-check.yaml`, `pr-dependabot.yaml`, `release.yaml`, `weekly-*.yaml`) are not affected by ordinary code changes. Prow job definitions live in kubernetes/test-infra and are generated here by `make generate-test-infra-prowjobs`.

## Where to read more

- `docs/book/src/developer/core/testing.md` — the canonical guide; has a long "Quick reference" for `envtest`, `fakeclient`, `ginkgo`, `gomega`, `go test`, plus IDE (IntelliJ/VS Code) run configurations and e2e log analysis.
- `docs/book/src/developer/core/e2e.md` — writing e2e specs, the e2e config file, namespaces/objects/exec helpers, portable tests, conformance.
- `docs/book/src/developer/core/tilt.md` — full `tilt-settings` field reference, `tilt-provider` configuration, podman/Lima notes, troubleshooting.
- `docs/book/src/developer/providers/getting-started/building-running-and-testing.md` — the same loop from a provider author's angle.
- `test/framework/README.md`, `util/test/builder/README.md`, `test/e2e/data/infrastructure-docker/README.md`.
- `CONTRIBUTING.md` — PR expectations, including "run generators and make sure `make test`/`make lint` pass".

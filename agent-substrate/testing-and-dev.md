# Testing, Code Style, and the Dev Loop

The project will not merge untested code (`AGENTS.md`: *"Write tests for all new code. We will not merge code that lacks tests."*). This page covers the test layers, the style rules that are specific to this repo, the verifiers, and the CI matrix.

## The two commands

```sh
go test -race ./...     # unit + functional
hack/verify-all.sh      # formatting, lint, boilerplate, licenses, modules, codegen, metrics
```

Run both before asking for review. `hack/verify-all.sh` catches the things that are easy to forget — a missing copyright header, a stale generated file, an unregistered metric.

## Test layers

| Layer | Where | Needs | Run with |
|---|---|---|---|
| **Unit** | `*_test.go` beside the code | nothing | `go test -race ./...` |
| **Store contract** | `cmd/ateapi/internal/store/storecontract/` | **Docker** (a PostgreSQL testcontainer) | same; skips locally without Docker, **fails in CI** |
| **Functional** | `cmd/ateapi/internal/controlapi/functionaltest/` | a real store | same. The fastest way to see an end-to-end control-plane path **without a cluster.** |
| **envtest** | `internal/testenv` consumers | the kubebuilder control plane | same |
| **Root-gated** | anything calling `roottest.Require` | **root** | `hack/run-root-tests.sh` |
| **E2E** | `internal/e2e/suites/<suite>/` | **a cluster with Substrate installed** | `hack/run-e2e.sh`, `hack/run-e2e-kind.sh` |
| **Benchmarks** | `benchmarking/` | a cluster | `benchmarking/deploy_locust.sh` |

The scale is substantial: ~370k lines of Go, and test files frequently exceed the code they test (`lifecycle_test.go` is 70 KB against `main.go`'s 86 KB; `actor_test.go` 41 KB; `crash_test.go` 34 KB; `xds_test.go` 65 KB; `outbox_test.go` 42 KB). That ratio is the norm here, not an outlier.

---

## The fail-closed rule — the single most important testing convention

`docs/dev/best-practices/ci-fail-closed.md`. **A skip and a pass are the same exit code.**

| Strategy | Locally | In CI |
|---|---|---|
| hard-fail everywhere | unrunnable without setup | correct |
| skip everywhere | runnable | **green having run nothing** |
| **skip locally, fail in CI** | runnable | correct |

The motivating incident is stated plainly:

> *"~280 PostgreSQL-backed tests once skipped when their testcontainer failed to start, including a 2,800-line store contract, and the job stayed green."*

### The convention

Resolve strictness **in one named function**. Never an inline `os.Getenv("CI")`.

Reference implementation — `cmd/ateapi/internal/store/dockerenv.Required()`:

```go
func Required() bool {
    return os.Getenv("CI") == "true" || os.Getenv("REQUIRE_DOCKER") == "true"
}
```

Callers branch on it:

```go
if containerErr != nil {
    if dockerenv.Required() {
        t.Fatalf("PostgreSQL testcontainer unavailable and required "+
            "(CI or REQUIRE_DOCKER is set): %v", containerErr)
    }
    t.Skipf("PostgreSQL testcontainer unavailable (requires Docker): %v", containerErr)
}
```

**Both messages name the cause; the fatal one also names what made it fatal.** The shell form is the same shape:

```bash
if [[ -z "${E2E_JUNIT_FILE:-}" && "${CI:-}" == "true" ]]; then
    echo "run-e2e.sh: E2E_JUNIT_FILE must be set when CI=true." >&2
    exit 1
fi
```

### And the rule that gives it teeth

> **"Prove the strict branch red before merging: a guard only ever observed passing is not known to guard anything."**

`AGENTS.md` repeats this as instruction #5 in Testing Instructions. Treat it as mandatory.

---

## Root-gated tests

Tests needing root (overlay mounts, `mknod`, `trusted.*` xattrs) call `roottest.Require(t, "<what>")` **as their first statement**. They skip in a plain `go test ./...`; **CI reruns every package whose tests import `internal/roottest` under `sudo`.**

```sh
hack/run-root-tests.sh
```

> **New privileged tests only need the `roottest.Require` call — no CI changes.** Importing the package is itself the marker.

`internal/roottest`'s own test is a nice touch: it asserts that the gated code ran **iff** `os.Geteuid() == 0`.

---

## Go style rules specific to this repo

`docs/code-style-guide.md` is short and records only project-specific decisions. The baseline is Effective Go + the Google Go style guide + `gofmt`.

### Proto field access: check presence, don't default it

This is the rule most likely to come up in review.

> *"Generated getters (e.g. `req.GetActor().GetName()`) return zero values when any message in the chain is nil. That is convenient and dangerous: a required message the caller forgot to set silently becomes `""`, and the bug surfaces far from its cause."*

- **A getter chain must never stand in for a presence check.** Where a field is required at that point, check it explicitly and fail loudly: `INVALID_ARGUMENT` at the API boundary, a real error (not a guessed zero value) elsewhere.
- Once the boundary has validated presence, downstream getters are fine, as is the guarded form:

```go
if wass := worker.Assignment; wass != nil {
    // Fields of wass were validated on write; use them directly.
}
```

- **Absence of a group of fields set and cleared together is a nil message** (`worker.Assignment == nil`), **never** probing a scalar inside it for its zero value.

### Testing style

- **Standard library `testing` only — no assertion or mocking frameworks.**
- **Table-driven with `t.Run` subtests is the default shape.**
- **Prefer a real test implementation when one exists**: the PostgreSQL fixture for the store, `envtest` for the Kubernetes API. Release resources with `t.Cleanup`.
- Tests using `preview.InitForTest` **cannot run in parallel** (the gates are process-wide).

### TODOs

`TODO(<issue-number>): ...`, **placed where the decision will eventually have to be made** — not in a list somewhere. This is why TODOs in this codebase are useful: each one is at the call site that will have to change.

### Comments

From `AGENTS.md`:

> *"Keep them brief and to the point. **Comment the final state of the code, not the path taken to it** — a problem that only existed partway through writing the change is noise to the next reader, as is a pointer to a scratch or planning file that isn't in the repository."*

### Spelling

**American English.** `golangci-lint` runs `misspell` with `locale: US`, so British spellings fail lint.

---

## Verifiers

`hack/verify-all.sh` runs everything in `hack/verify/`:

| Script | Checks |
|---|---|
| `gofmt.sh` | formatting (fix with `hack/update/gofmt.sh`) |
| `golangci-lint.sh` | `.golangci.yaml` — includes `misspell` with `locale: US` |
| `kube-api-linter.sh` | `.golangci-kal.yaml` over the CRD types in `pkg/api/v1alpha1` |
| `boilerplate.sh` | copyright headers (templates in `hack/boilerplate/`) |
| `licenses.sh`, `python-licenses.sh` | dependency licenses (`_LICENSES/`) |
| `go-modules.sh` | `go.mod`/`go.sum` tidy across every module |
| `codegen.sh` | **fails if generated code is stale.** Fix with `hack/update/codegen.sh`. |
| `proto-fmt.sh` | proto formatting |
| `metrics.sh` | the OTel Weaver registry (`weaver registry check`) |
| `postgresql-migrations.sh` | **migrations are immutable** — see `docs/dev/postgresql-schema-evolution.md` |
| `ateapi-deps.sh` | guards what `cmd/ateapi` may import |
| `threats.sh` | `docs/threats.json` ↔ `docs/threat-model.md` consistency |
| `python-protos.sh`, `benchmarking-python.sh` | the benchmarking harness |
| `shellcheck.sh` | shell scripts |

`hack/update/` has the matching fixers: `gofmt.sh`, `codegen.sh`, `licenses.sh`, `proto-fmt.sh`, `go-modules.sh`. `hack/update-all.sh` runs them all.

---

## E2E tests

`internal/e2e/README.md`. **Go `test` is the harness — no Ginkgo, no custom runner.**

### Principles

- Suites live in `internal/e2e/suites/<suite>`; helper libraries in `internal/e2e`.
- **Each suite implements `TestMain` using `e2e.RunTestMain()`**, so e2e tests are **skipped by a plain `go test ./...`** unless `--e2e` is passed.
- Setup and teardown are **per component**, and the component author's responsibility.

```go
func run(m *testing.M) int {
	Setup()
	defer Teardown()
	return e2e.RunTestMain(m)   // return, so the deferred Teardown runs
}
func TestMain(m *testing.M) { os.Exit(run(m)) }
```

Copy `testmain_test.go` from `internal/e2e/suites/example` to start a new suite.

### The 17 suites

`authz`, `capabilities`, `combinedvolumes`, `demo`, `egressauthz`, `egresscredinject`, `egressmitm`, `example`, `identity`, `metrics`, `multiactor`, `networking`, `networkpolicy`, `parking`, `sandboxconfig`, `security`, `sizing`.

### Running them

```sh
hack/run-e2e-kind.sh -v -args --no-color             # kind, gVisor (the default)
E2E_SANDBOX_CLASS=microvm hack/run-e2e-kind.sh -v -args --no-color
hack/run-e2e.sh ./internal/e2e/suites/networking -run TestFoo -v -args --preview=ALL
```

Flags before `-args` go to `go test`; flags after go to the test binary. E2E flags: `--e2e`, `--no-color`, `--kube-config`, `--kube-context`, `--storage-class` (default `csi-nfs-sc`), `--preview`. Defaults: `-p $E2E_PARALLELISM` (4), `-parallel $E2E_TEST_PARALLELISM` (8), `-timeout $E2E_TIMEOUT` (30m).

### Runtime-agnostic by design

**The same suites run against gVisor and micro-VM.** `E2E_SANDBOX_CLASS` repoints every fixture at its variant (`e2e.CounterFixture`, `e2e.EgressFixture`, `e2e.RenderFixtureManifest` in `sandbox.go`). Unset means gVisor.

The micro-VM lane needs its fixtures installed first, and a node with `/dev/kvm`:

```sh
hack/run-microvm-demo-kind.sh                          # counter-microvm + assets
hack/install-ate-kind.sh --deploy-demo-egress-microvm  # egress-microvm
```

Overrides for a cluster that installs fixtures elsewhere: `E2E_SUBSTRATE_TEMPLATE_ATESPACE`, `E2E_SUBSTRATE_TEMPLATE_NAME`, `E2E_SUBSTRATE_POOL_NAMESPACE`, `E2E_SUBSTRATE_POOL_NAME`, and `E2E_TEMPLATE_READY_TIMEOUT` (the golden-snapshot budget: **90 s on gVisor, 10 min on micro-VM**, where the golden is a Cloud Hypervisor cold boot plus a checkpoint).

### Preview features in e2e

```go
func TestMyFeature(t *testing.T) {
    e2e.RequirePreview(t, preview.GateMyFeature)   // skips unless --preview names it
    ...
}
```

The `--preview` value **must match what the cluster was installed with.** **CI must pass it, even as empty, so a lane that forgets cannot silently skip every preview test** (`internal/e2e/preview.go`). Unset or empty means no gates.

### After a failure

**A suite deletes the namespaces it created only when it passed.** Deliberately:

> *"A failed run keeps them, because the failure is usually explained inside a worker pod (the worker logs, and for a micro-VM worker the guest's console tail), and deleting the namespace takes those pods with it."*

```sh
kubectl logs -n <kept-namespace> <worker-pod>
hack/cleanup-e2e.sh   # deletes every namespace labeled ate.dev/e2e — when you're done reading
```

**Nothing reclaims them afterwards, and each holds a WorkerPool's worth of running pods.**

### E2E helpers worth knowing

`internal/e2e/`: `fixture.go`, `template.go`, `sandbox.go`, `serverpod.go`, `router_client.go`, `probe.go`, `resume.go`, `namespace.go`, `manifest.go`, `trustbundle.go`, `collector_metrics.go` (asserting on telemetry), `statusz.go`, `credprovider.go`, `egresspolicy.go`, `atenet_dataplane.go`, `preflight.go`, `pretty.go`.

### An e2e-only install setting

| Key | Env | Flag | Default |
|---|---|---|---|
| `e2e.envoyConcurrency` | `E2E_ATE_ENVOY_CONCURRENCY` | `--e2e-envoy-concurrency` | `0` |

Sets `--concurrency` (worker threads) on the Envoy container in `atenet-egress`; `0` leaves the manifest unchanged. The nighthawk egress benchmark sets it to `1` to measure single-CPU Envoy performance.

---

## CI (`.github/workflows/pr-workflow.yaml`)

### The unit job

build for arm64 · unit tests (`-race`) · apitool tests · junittool tests · the `gcp-secret-manager` plugin's own tests and verify (it is its own Go module) · dynamic-module tests (with a cargo cache, for the Rust `envoy-dataplane`) · **root-gated tests** · **"Require tests to have run"** · "Summarize test failures" (into the job summary and annotations) · upload `test-results-unit` · `verify` · **"Verify immutable PostgreSQL migrations"**.

Note the explicit **"Require tests to have run"** step — the fail-closed philosophy applied to the job itself.

### The e2e matrix

Each entry is **an independent job on a fresh cluster**, `timeout-minutes: 120` (*"generous, but far short of the six-hour platform default a hung lane would otherwise consume"*).

| Axis | Values | Notes |
|---|---|---|
| `dataplane` | `envoy`, **`agentgateway`** | agentgateway runs **pre-submit only on `release-*` branches**; elsewhere post-merge and on the weekly schedule. `continue-on-error` for agentgateway until it stabilizes. |
| `preview` | `off`, `on` | **`off` is the default install and proves the suites pass without preview features; `on` enables every one.** |

The `envoy` + `preview: off` entry reports the **required** `e2e-test` check.

Job-wide env, each with a stated reason:

- `ATE_CREDENTIAL_PROVIDER='{"name":"k8s.io"}'` — *"the installer requires a credential provider selection… Job-wide because every `ate-setup` invocation reads it, including the control plane redeploy in `run-microvm-demo-kind.sh`."*
- `ATE_PREVIEW` = `ALL` or `''` — *"**Empty is an explicit 'no preview features', not an unset value.** The e2e steps pass it to the suites as `--preview`, so tests of a disabled gate skip themselves."*
- `LANE_SUFFIX` — appended to lane and artifact names (`e2e-gvisor` vs `e2e-gvisor-preview`).
- **`E2E_JUNIT_FILE` is set per step, not job-wide**, *"the lanes below share this job, so one job-level path would have each overwrite the last."* And `run-e2e.sh` **fails closed** when `CI=true` and it is unset.
- `DOCKER_BUILD_FLAGS` caches the `envoy-dataplane` Rust build in the GHA cache — **only `main` writes it**, so PRs share one copy instead of each storing their own.

Two build-cache notes worth copying: `setup-go`'s `cache: false` because its key is shared with the unit job, *"whose unit-test builds don't match ko's flags (`CGO_ENABLED=0`, `-trimpath`), so ko got no cache hits from it"* — a separate `e2e-go-` cache is restored instead. And buildx uses the `docker-container` driver with `network=host`, *"the docker-container driver is what can use the GitHub Actions cache. Host networking and plain HTTP let it push to the kind registry."*

Other workflows: `govulncheck.yaml`, `release.yaml`.

---

## The dev loop

### kind

```sh
hack/create-kind-cluster.sh
hack/install-ate-kind.sh --deploy-ate-system --credential-provider='{"name":"k8s.io"}'
hack/install-ate-kind.sh --deploy-demo-counter
go install ./cmd/kubectl-ate
```

Rebuild one component:

```sh
hack/install-ate.sh --deploy-ate-apiserver
hack/install-ate.sh --deploy-atelet
hack/install-ate.sh --deploy-atenet
hack/install-ate.sh --help            # every option
hack/install-ate.sh --delete-all
hack/delete-kind-cluster.sh
```

Everything is built from the checkout with `ko` and pushed to the local registry — **no manual image step.**

Full walkthrough: [end-to-end-on-kind.md](end-to-end-on-kind.md).

### GKE

```sh
cp hack/ate-dev-env.sh.example .ate-dev-env.sh    # edit, then:
source .ate-dev-env.sh
gcloud auth application-default login --project=${PROJECT_ID}
go run ./tools/setup-gcp bootstrap                # GKE + GCS + IAM
hack/install-ate.sh --deploy-ate-system --credential-provider='{"name":"k8s.io"}'
hack/teardown.sh --all
```

> **The required Kubernetes beta APIs can only be enabled *at cluster creation*.** If you bring your own cluster, read the "Create Cluster" warning in `tools/setup-gcp/README.md` first. `bootstrap` on a fresh project also creates the atelet Workload Identity IAM grants that snapshots depend on — `tools/setup-gcp/README.md#what-create-iam-actually-grants` to audit or apply them by hand.

`tools/setup-gcp/cloud-sql.md` covers running the store on Cloud SQL with IAM database authentication and no passwords.

### micro-VM locally

`docs/dev/microvm-local.md`. Needs `/dev/kvm`, or Lima nested virtualization on Apple Silicon. `hack/create-kind-cluster.sh` probes for it and only wires it up when present; **without KVM the cluster still works for gVisor.**

```sh
hack/install-microvm-deps.sh
hack/run-microvm-demo-kind.sh
```

### Benchmarking

```sh
./benchmarking/deploy_locust.sh --deploy --sandbox-class microvm --worker-count 3 --actor-memory 1536Mi
kubectl port-forward svc/locust -n benchmarking 8089:8089    # pick GluttonUser
./benchmarking/deploy_locust.sh --delete
```

`benchmarking/README.md` is **44 KB** — the most detailed operational doc in the repo. `cmd/benchmarking/glutton` is the synthetic workload that consumes RAM, disk and file descriptors on demand; `benchmarking/locust/common/memload_config.py` has its flags. `benchmarking/analysis/` turns the phase logs into percentiles and waterfalls; `benchmarking/automation/orchestrator.py` drives whole runs; `benchmarking/nighthawk-{ingress,egress}/` are the gateway benchmarks.

---

## Debugging quick reference

```sh
kubectl -n ate-system get pods
kubectl -n ate-system logs deploy/ate-api-server -f
kubectl -n ate-system logs ds/atelet -f
kubectl -n ate-system logs deploy/atenet-router -c atenet-router -f
kubectl -n <pool-ns> logs <worker-pod> -f        # includes the actor's own output, identity-tagged

kubectl ate get actor <name> -a <atespace> -o yaml   # status.snapshots is the ground truth
kubectl ate get workers
kubectl ate get actor-templates -a <atespace>        # the golden column

kubectl -n ate-system port-forward deploy/atenet-router 4040:4040   # /statusz: last 100 queries
kubectl port-forward -n otel-system svc/jaeger 16686:16686
kubectl port-forward -n otel-system svc/prometheus 9090:9090
kubectl ate get actor <name> -a <atespace> --trace

kubectl logs -n ate-system ds/atelet | grep "timing breakdown"
docker exec -it kind-control-plane ls -la /var/lib/ate
```

**Reading a trace:** `step.skipped=true` + `step.skip_reason` on a span means a re-entered workflow fast-forwarded past completed work — that is normal and by design. Look for **the first span that is not skipped** (where this attempt started doing work) and any error, whose message is always `workflow failed at step <Name>: …`.

---

## Contributing

`CONTRIBUTING.md` + `AGENTS.md`.

### Process

- **CLA required** (`cla.developers.google.com`).
- **Discuss first.** *"Before you start a new contribution, please discuss it with us first (if there is an issue open, comment there and if not, open one)… PRs which are not aligned with our near-term goals may be closed without extensive review."* The near-term goals are the six priorities in `docs/roadmap.md`.
- Building something that runs *on* Substrate rather than changing Substrate → `docs/integration-repos.md`.

### PR sizing

> *"We optimize PRs for easy review — large PRs get broken down, small PRs get merged."*

| Size | Lines changed |
|---|---|
| S | < 30 |
| M | < 100 |
| L | < 500 |
| XL | < 1000 |

**Most PRs should be L or smaller; XL and above are candidates for breaking down.**

- **Large:** split into a series, each a logically distinct feature. When the intermediate steps are not useful alone, keep it as one PR split into commits at logical break points **and preserve those commits on merge.**
- **Small and bulk:** *"if you find a typo, review the whole file and fix everything in one pass rather than sending the single edit. Group related typo, doc, and single-line cleanup fixes into one PR."* Maintainers may ask you to consolidate fragmented PRs or close them in favour of a combined one.

### Commit messages

- **Describe the change and why it was needed** — *"The message is read on `main` long after the PR branch is gone, so it should stand on its own."*
- **No issue or PR references.** No `#1234`, no `Fixes #1234`, no GitHub URLs. *"GitHub renders them as cross-references on the linked thread, and rebases or force-pushes repeat them. Put that context in the pull request description instead, where it belongs to the review rather than to the permanent history."*

### PR titles are release notes

> **"Each pull request title becomes its line in the release notes."** (`docs/dev/release-notes.md`)

### No backward compatibility before v1.0.0

`AGENTS.md`, and it shapes every review:

> *"Do not add code to stay compatible with older clients, binaries, protos, flags, or config. Remove or rename fields, RPCs, flags, and types outright and update every caller in the same change. Do not add `reserved` statements to protos, deprecated aliases, fallbacks for old formats, or version negotiation."*

### Other gates

- **Copyright headers on every source file** — `hack/boilerplate/`.
- **PostgreSQL schema changes follow `docs/dev/postgresql-schema-evolution.md`.** Every migration must keep working for the previous `ate-api-server` binary; migrations are never rolled back.
- `go mod tidy` if dependencies changed.
- American English.

---

## Where to read more

- `AGENTS.md` — the single best orientation file; written for coding agents but correct for humans.
- `CONTRIBUTING.md`, `COLLABORATING.md`, `GOVERNANCE.md`, `MAINTAINERS.md`, `CODE_OF_CONDUCT.md`.
- `docs/code-style-guide.md` — short, and all of it specific to this repo.
- `docs/dev/best-practices/ci-fail-closed.md` — read before writing a test with a precondition.
- `internal/e2e/README.md` — the e2e contract.
- `docs/dev/postgresql-schema-evolution.md`, `docs/dev/release-notes.md`, `docs/dev/microvm-local.md`, `docs/dev/mitm-egress-test-runbook.md`, `docs/dev/suspend-resume-phase-breakdown.md`.
- `benchmarking/README.md` — exhaustive.

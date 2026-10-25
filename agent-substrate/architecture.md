# Agent Substrate Architecture

The 10,000-foot map of `agent-substrate/substrate`: the eight binaries, who talks to whom, the two API surfaces, and the cross-cutting machinery (preview gates, pod certificates, OTel, serverboot) you hit in the first hour. Read this before touching code you haven't worked in.

**One-line thesis.** Substrate maps many *actors* (stateful, mostly-idle, untrusted agent-like workloads) onto few *workers* (pre-started Kubernetes Pods), by snapshotting idle actors out of memory and restoring them on demand. Kubernetes provisions capacity; a purpose-built control plane does the high-QPS per-actor scheduling that etcd and kube-scheduler cannot.

Targets from `docs/architecture.md`: 100 ms p95 activation latency, 1B actors per cluster, 1000 wakeups/s. Current README claims: <400 ms resume, >1500 activations/s.

## The two API surfaces

This is the single most important structural fact about the project.

| Surface | Where defined | Served by | Stored in | Holds |
|---|---|---|---|---|
| **gRPC control API** | `pkg/proto/ateapipb/ateapi.proto` (3050 lines) | `ate-api-server` | PostgreSQL | Atespace, ActorTemplate, Actor, Tag, Worker, ActorAssignment, EgressPolicy, AccessPolicy |
| **Kubernetes CRDs** | `pkg/api/v1alpha1/` | kube-apiserver | etcd | `WorkerPool` (namespaced), `SandboxConfig` (cluster), `CSIDriverConfig` (cluster) |

The split is deliberate (`docs/architecture.md` "Architectural Rationale"): **infrastructure** is Kubernetes-shaped so platform teams get RBAC/audit/policy for free; **workload instances** are database-shaped because millions of records changing many times per second would melt etcd, and because a 100 ms activation cannot afford informer-convergence latency.

Consequence you will hit constantly: an `ActorTemplate` is **not** a Kubernetes object. `kubectl get actortemplate` does nothing. Use `kubectl ate get actor-templates`.

See `../agent-substrate/resource-model.md` → `docs/resource-model.md` for the hierarchy, and [api-and-validation.md](api-and-validation.md) for how the proto is validated.

## Binaries

Every binary is `cmd/<name>/main.go`, thin, with its private packages in `cmd/<name>/internal/`.

| Binary | Shape | LOC | Role |
|---|---|---|---|
| `cmd/ateapi` (`ate-api-server`) | Deployment, gRPC :443 | ~80k | The control plane. Owns actor lifecycle, scheduling, workflows, the store, authn/authz, actor identity minting. |
| `cmd/atelet` | **DaemonSet** | ~36k | Node supervisor. Pulls images, assembles OCI bundles, stages sandbox assets, drives `ateworker`, moves snapshots, serves `WorkerSupport` to in-pod atunnel. |
| `cmd/ateworker-gvisor` | container **inside** each worker pod | ~8k | Runs actors under `runsc`; checkpoint/restore via gVisor. Hosts `atunnel`. |
| `cmd/ateworker-microvm` | container inside each worker pod | ~20k | Runs actors as Cloud Hypervisor micro-VMs via the Kata guest model; CH snapshot/restore with `userfaultfd` demand paging. |
| `cmd/atecontroller` (`ate-controller`) | Deployment, controller-runtime | ~10k | Reconciles `WorkerPool`→Deployment, `SandboxConfig` protection, NetworkPolicy, egress MITM trust. Also runs `workersync`, which mints `Worker` records from worker Pods. |
| `cmd/atenet` | Deployment ×2 (`atenet-router`, `atenet-egress`) | ~20k | One binary, `--mode=ingress|egress`. Envoy xDS control plane + `ext_proc` server. |
| `cmd/podcertcontroller` | Deployment | ~2.6k | Polyfill for upstream Kubernetes PodCertificate signers. Three signers; the root of all component mTLS. |
| `cmd/ate-idp-server` | Deployment | ~0.5k | Serves OIDC discovery + JWKS so relying parties can verify actor JWTs. |
| `cmd/kubectl-ate` | CLI (`kubectl` plugin) | ~10k | Talks the gRPC API. Auto-port-forwards to `ate-api-server` by default. |
| `cmd/ate-setup` | CLI | ~26k | **The installer.** `hack/install-ate.sh` is a thin shim over it. |
| `cmd/objectstore-plugin` | **sidecar** | ~0.3k | Implements `objectstore.v1` over a unix socket, backed by GCS or S3. |
| `cmd/credential-provider` | sidecar/Deployment | ~1.8k | Reference credential provider (`k8s.io`, reads Kubernetes Secrets) for egress credential injection. |
| `cmd/benchmarking` | — | ~4.9k | Synthetic workloads, notably `glutton` (burns RAM/disk/FDs on demand). |
| `cmd/dataplane` | (no Go) | 0 | Holds the docker-built `envoy-dataplane` image build context. |

`tools/setup-gcp` is a **separate Go module** (`go run ./tools/setup-gcp`) that provisions GKE/GCS/IAM.

## Who calls whom

```
kubectl-ate ──gRPC/mTLS or JWT──► ate-api-server ──┬──► PostgreSQL (store + embedded OpenFGA)
                                      │            └──► objectstore-plugin sidecar (ControlProvider: copy/cleanup snapshots)
                                      │
atenet-router ──ResumeActor──────────►│
atenet-egress ──GetActor / Mint*─────►│
ate-controller ──Create/Update Worker►│
                                      │
                                      └──Atelet RPC (mTLS, k8s:/// resolver)──► atelet (DaemonSet, per node)
                                                                                   │
atelet ──unix socket──► objectstore-plugin sidecar (NodeProvider + AssetProvider)   │
atelet ──unix socket /var/lib/ate/workers/<podUID>/worker.sock──► ateworker (in worker pod)
ateworker/atunnel ──unix /var/lib/ate/worker-support.sock──► atelet (WorkerSupport) ──► ate-api-server (WorkerService)
```

Three gRPC services, three audiences:

| Service | Proto | Callers |
|---|---|---|
| `Control` | `pkg/proto/ateapipb` | clients (`kubectl-ate`), `atenet-router`, `atenet-egress`, `ate-controller` |
| `WorkerService` | same file, `ateapi.proto:2546` | **only `atelet`**, and only for Workers on its own node |
| `Atelet` + `WorkerSupport` | `internal/proto/ateletpb` | `ate-api-server` → `Atelet`; in-pod `atunnel` → `WorkerSupport` |
| `Worker` | `internal/proto/ateworkerpb` | `atelet` → `ateworker` |
| `NodeProvider`/`ControlProvider`/`AssetProvider` | `pkg/proto/objectstorepb/v1` | `atelet`/`ate-api-server` → their own sidecars |
| `CredentialProvider` | `pkg/proto/credproviderpb` | `atenet-egress` → credential provider |

`Control` and `WorkerService` live in the *same* proto file but are split as services because they have different callers and different authorization — `WorkerService` bypasses the authz interceptor entirely and each handler checks `atelet`'s pod certificate by hand.

## Repository layout

Authority: `docs/dev/code-layout.md` and the table in `AGENTS.md`.

| Path | What's in it |
|---|---|
| `cmd/<binary>/` | One per binary. `main.go` thin; `cmd/<binary>/internal/<pkg>` for single-binary packages. |
| `internal/<pkg>` | Shared across binaries, not externally importable. ~60 packages — see the table below. |
| `pkg/<pkg>` | Deliberately public API. **Use sparingly** — post-GA it carries a compat guarantee. Today: `api/v1alpha1` (CRDs), `client/` (generated clientset/informers/listers), `proto/`, `objectstorage/`, `sandboxclass/`, `postgressetup/`, `assetfetch/`, `v1alpha1/`. |
| `internal/plugins/<name>` | Self-contained plugins, **each its own `go.mod`**, importing only `pkg/`. Nothing in the repo may depend on them. |
| `docs/` | Design docs, dev guides, and `docs/metrics/registry/` (an OTel Weaver registry). |
| `hack/` | Shell for humans and CI. `hack/verify-all.sh`, `hack/update/*`, `hack/verify/*`. |
| `manifests/ate-install/` | The install bundle: one YAML per component plus kustomize overlays (`kind/`, `postgres/`, `cloudsql/`, `agentgateway*/`, `components/`). |
| `demos/` | 8 self-contained demos. `tools/` | Standalone Go tools, each its own `go.mod`. |
| `benchmarking/` | Locust load tests, nighthawk benchmarks, `analysis/` (phase-breakdown reports), `monitoring.yaml`. |

### Where to put new Go code

| Situation | Location |
|---|---|
| Only used by one binary | `cmd/<binary>/internal/<pkg>` |
| Shared across binaries, not public | `internal/<pkg>` |
| Public API for external consumers | `pkg/<pkg>` |
| Public proto (control-plane gRPC) | `pkg/proto/<name>` |
| Internal proto (atelet/ateworker) | `internal/proto/<name>` |
| Dev/CI script | `hack/` |
| Standalone Go dev/CI tool | `tools/<name>` with its own `go.mod` |

## The `internal/` packages worth knowing by name

Grouped by what they're for. Each has a real package doc — read it rather than guessing.

**Identity and crypto**
- `substratex509` — creates/parses the Substrate X.509 extensions carrying workload identity. The `ActorIdentity` extension OID is `1.3.6.1.4.1.11129.2.12.2` (Google's enterprise number; **will change after CNCF donation**).
- `localca` — a CA whose state lives in a file or a Kubernetes Secret. Backs all three podcert signers and the actor-identity CA.
- `localjwtauthority` — a minimal JWT "CA". `oidcjwt` verifies JWTs via OIDC discovery. **Both built on stdlib crypto only — `AGENTS.md` forbids adding go-jose or any JOSE library.**
- `credbundle` — reads Kubernetes PodCertificate credential bundles (one PEM file: private key first, then chain).
- `clustertrustbundle` — handles both stable and beta Kubernetes trust-bundle APIs.
- `actoridjwt`, `principal`, `ateapiauth`, `pemutil`, `testcert`.

**Networking**
- `atunnel/` — the in-worker tunnel: `ingress/`, `egress/`, and a full per-actor `dns/` relay (see [networking-and-egress.md](networking-and-egress.md)).
- `atenet` — the shared contract: `ate-target-actor` header parsing lives here (`internal/atenet/headers.go`).
- `netns` — creates Linux netns and runs code/dials inside them; locks an OS thread for the duration.
- `egresspolicy` — evaluates an `EgressPolicy` against a destination. **ateapi validates patterns with the same parser the gateway matches with, so they cannot drift.**
- `k8sresolver` — the `k8s:///service.ns:port` gRPC resolver. `portforward` — tunnels to pods behind a Service (used by `kubectl-ate`).

**Node / sandbox**
- `nodepath` — the shared node paths. `BasePath = /var/lib/ate`, mounted at the same path in atelet and every worker pod. Memorize: `actors/<uid>/sandbox-assets`, `workers/<podUID>/worker.sock`, `workers/<podUID>/sandbox-runtimes.json`, `worker-support.sock`, `atelet-otlp.sock`.
- `imagecache` — node-local content-addressed unpacked-layer pool + the per-bundle overlay spec.
- `ocispec` — runtime-neutral OCI spec for actor bundles; each worker shapes it for its runtime.
- `sizing` — turns the actor's declared limits into a sandbox size (cgroup quota + Sentry vCPUs for gVisor; CH vCPUs/RAM for micro-VM).
- `hardware` — probes host compatibility attributes and decides whether a worker can restore a given snapshot.
- `sandboxruntimes` — reads/writes `sandbox-runtimes.json`. `deviceplugin` — advertises `/dev/kvm` etc. to kubelet. `cdi` — Container Device Interface specs. `linuxsys`, `volume`, `volumepath`, `tarutil`, `snapshotcache`, `objectstore`, `objectstoreplugin`, `wakeupprobe`, `workercapacity`, `workerstats`.

**Cross-cutting**
- `preview` — the preview-gate mechanism. Process-wide; `Init` once at startup. See [preview-gates-and-compat.md](preview-gates-and-compat.md).
- `serverboot` — the startup boilerplate every long-running server shares: slog, OTel tracer+meter, Prometheus endpoint, health server.
- `ateattr` — **the single source of truth for `ate.*` telemetry attributes.** Identity keys for spans/logs, bounded value sets for metric labels. Never invent an attribute key elsewhere.
- `actorevent` — emits actor lifecycle events and usage samples, as OTLP events or stdout, from one call so the two cannot drift.
- `apierror` — the errors a server *deliberately* returns. Deliberately **not** a gRPC status (no `GRPCStatus` method) so an accidental error can't masquerade as an intended one.
- `protoredact` — masks `[debug_redact = true]` proto fields so messages can be logged.
- `resources` — the `Ref` types (`ActorRef`, `TagRef`, `ActorTemplateRef`), `StoragePrefix`, `SnapshotURI`, `ToAPIError`.
- `actorlock` (serializes lifecycle RPCs per actor, in-process), `actorlog`, `contextlogging`, `pluginsocket`, `otlprelay`, `installdefaults`, `version`, `roottest`, `testenv`, `e2e`.

## Cross-cutting machinery you hit immediately

### Pod certificates are the root of trust
`podcertcontroller` implements three Kubernetes PodCertificate signers, all backed by `internal/localca`:

| Signer | Issues |
|---|---|
| `servicedns.podcert.ate.dev/identity` | Certs whose DNS SANs are derived from the **Services that select the pod**. A workload must be selected by a Service to get one. |
| `podidentity.podcert.ate.dev/identity` | Certs equivalent to a Kubernetes ServiceAccount token — SPIFFE ID `spiffe://cluster.local/ns/<ns>/sa/<sa>`. This is the component-mTLS identity. |
| `postgres.podcert.ate.dev/identity` | PostgreSQL login certs for `ate-api-server`. CN = the authorized login, selected by the `postgres.podcert.ate.dev/username` annotation. |

Each publishes a `ClusterTrustBundle` (`<signer>:identity:primary-bundle`, labeled `podcert.ate.dev/canarying=live`). Every component mounts a `credential-bundle.pem` + `trust-bundle.pem` pair. There are no static TLS Secrets.

### Preview gates
Two independent switches: server-side `--preview=<gates>|ALL` on *every* binary, and client-side `Ate-Preview-Allowed: true` metadata (`kubectl ate --preview`). Gates declared in `internal/preview/preview.go`; API fields gated with `+k8s:ifDisabled(<Gate>)=+k8s:forbidden`. Full rules in [preview-gates-and-compat.md](preview-gates-and-compat.md).

### `serverboot` + OTel everywhere
Every server calls `serverboot.InitLogger`/`SetLogLevel`, gets a tracer + meter provider, a Prometheus `--metrics-listen-addr` (`:9090`) and a `/healthz`+`/readyz` server (`--health-listen-addr`, `:9091`). Worker pods have **no network path** to the collector — they export OTLP through `internal/otlprelay` over atelet's unix socket. Metric instruments are declared in an **OTel Weaver registry** at `docs/metrics/registry/metrics.yaml`; `hack/verify/metrics.sh` gates CI. See [observability.md](observability.md).

### Graceful drain is a first-class concern
Every long-running server has `--drain-delay` (keep accepting after SIGTERM, so Service endpoints propagate) and `--drain-timeout`. `atenet-router`'s is the most elaborate: readiness flip → endpoint propagation → Envoy admin-API drain → ext_proc drain → write a drain-complete marker on a shared emptyDir that releases the Envoy container's `preStop` hook. The whole sequence must fit inside `terminationGracePeriodSeconds`.

### Control-plane isolation
When the installer is told to, control-plane workloads select `ate.dev/workloadType=ate-control-plane` and tolerate the matching `NoSchedule` taint, with postgres on its own one-node pool (`ate.dev/workloadType=ate-postgres`), so a burst of worker load cannot starve the control plane.

## No backward compatibility before v1.0.0

`AGENTS.md` is explicit and this shapes every review: **do not add code to stay compatible with older clients, binaries, protos, flags, or config.** Remove or rename fields/RPCs/flags/types outright and update every caller in the same change. No proto `reserved`, no deprecated aliases, no old-format fallbacks, no version negotiation. (The version-skew policy in [preview-gates-and-compat.md](preview-gates-and-compat.md) applies *from* 1.0.0.)

## Operational quick reference

```sh
# local cluster + registry (IPv4; IP_FAMILY=dual|ipv6 overrides)
hack/create-kind-cluster.sh
hack/install-ate-kind.sh --deploy-ate-system --credential-provider='{"name":"k8s.io"}'
hack/install-ate-kind.sh --deploy-demo-counter
go install ./cmd/kubectl-ate

kubectl ate create actor my-counter-1 -a ate-demo-counter --template counter
kubectl port-forward -n ate-system svc/atenet-router 8000:80
curl -X POST -H "ate-target-actor: ate-demo-counter/my-counter-1" -i http://localhost:8000/
```

- `--credential-provider` is **required** on install. `{"name":"k8s.io"}` deploys the bundled provider with an empty, **default-deny** namespace policy — it resolves no Secrets until you grant each atespace its namespaces.
- A Linux host running >~1000 sandboxes needs a bigger kernel neighbor table (`net.ipv4.neigh.default.gc_thresh1/2/3` = 4096/16384/32768). `hack/create-kind-cluster.sh` warns at the default of 1024.
- Everything: `go test -race ./...` and `hack/verify-all.sh` before asking for review.

## Where to read more

- `docs/architecture.md` — the authoritative design doc, with the mermaid lifecycle/state/resource diagrams. **Its header warns much of it is aspirational.**
- `docs/glossary.md` — read this *first* if the vocabulary is new. Precise definitions of actor/atespace/worker/fidelity/pause-vs-suspend.
- `docs/resource-model.md` — scope, parent, subresource, the full hierarchy.
- `docs/security.md` + `docs/threat-model.md` + `docs/threats.json` — security principles with honest Met/In-Progress/Future/Out-of-Scope status per principle.
- `docs/roadmap.md` — the six stated priorities and the "coming soon" list. Read before proposing anything large.
- `docs/dev/code-layout.md`, `AGENTS.md` — where code goes, how to build and verify.
- `CONTRIBUTING.md` — CLA, PR sizing (S<30 / M<100 / L<500 / XL<1000 lines; most PRs should be L or smaller), root-gated tests, and the rule that **PR titles become release-note lines**.

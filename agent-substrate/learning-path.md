# Learning path: from Kubernetes engineer to Agent Substrate contributor

A staged reading and practice order for someone who knows Kubernetes well but is new to Agent Substrate. Each stage says what to read, what to do with your hands, and what "done" looks like.

**Budget:** stages 0–2 are a focused week. Stage 3 is your first PRs. Stages 4–5 are where the real design work is.

**Before anything else, internalize two facts.** They cause more confusion than everything else combined:

1. **An `ActorTemplate` is not a Kubernetes object.** `kubectl get actortemplate` does nothing. Only `WorkerPool`, `SandboxConfig` and `CSIDriverConfig` are CRDs; Atespace, ActorTemplate, Actor, Tag, Worker, ActorAssignment, EgressPolicy and AccessPolicy live in PostgreSQL behind a gRPC API.
2. **An atespace is not a Kubernetes namespace.** They are orthogonal axes. Actors and templates live in atespaces; WorkerPools live in Kubernetes namespaces.

---

## Stage 0 — The mental model (an afternoon)

Read, in this order:

1. **`docs/glossary.md`** — short, precise, and the vocabulary is non-obvious. Especially: actor vs worker, atespace, the three snapshot fidelities, and **pause vs suspend** (they are not the same thing and both end in `SUSPENDED`).
2. **`docs/architecture.md`** — the design doc, with the mermaid lifecycle, state-machine and resource diagrams. **Its header warns that much of it is aspirational;** believe the warning, and treat `docs/security.md`'s Met/In-Progress/Future columns as the corrective.
3. **[architecture.md](architecture.md)** in this knowledge base — the component map, the two API surfaces, the `internal/` package index.
4. **[resource-model.md](resource-model.md)** — the nouns, properly.
5. `README.md` in the repo, for the pitch and the quickstart.

**Hands-on:** bring up a kind cluster and run the counter demo end to end.

```sh
hack/create-kind-cluster.sh
hack/install-ate-kind.sh --deploy-ate-system --credential-provider='{"name":"k8s.io"}'
hack/install-ate-kind.sh --deploy-demo-counter
go install ./cmd/kubectl-ate

kubectl ate create actor my-counter-1 -a ate-demo-counter --template counter
kubectl port-forward -n ate-system svc/atenet-router 8000:80
# in another terminal:
curl -X POST -H "ate-target-actor: ate-demo-counter/my-counter-1" -i http://localhost:8000/
curl -X POST -H "ate-target-actor: ate-demo-counter/my-counter-1" -i http://localhost:8000/   # counters increment
kubectl ate suspend actor my-counter-1 -a ate-demo-counter
kubectl ate get actor my-counter-1 -a ate-demo-counter -o yaml                                 # read status.snapshots
curl -X POST -H "ate-target-actor: ate-demo-counter/my-counter-1" -i http://localhost:8000/   # BOTH counters continue
```

Then poke at it:

```sh
kubectl -n ate-system get pods
kubectl ate get workers
kubectl -n ate-demo-counter get pods -l ate.dev/worker-pool
docker exec -it kind-control-plane ls -la /var/lib/ate/actors
kubectl -n ate-demo-counter exec <worker-pod> -- ps aux | grep runsc
```

**Done when:** you can say, without looking, which objects exist for one running actor, which store each lives in, and which component wrote each one. And you can explain why `runsc` is running *inside* an ordinary runc pod rather than being a `RuntimeClass`.

---

## Stage 1 — The activation path (a day or two)

This is the single most important thing to understand in the project, and it's the thing every design discussion comes back to.

Read **[end-to-end-on-kind.md](end-to-end-on-kind.md)** with the code open beside it, in this order:

1. `cmd/atenet/internal/router/ingress/ingress.go` → `HandleRequestHeaders`
2. `cmd/ateapi/internal/controlapi/workflow.go` → **read the `stepSpan` doc comment first; it is the design brief for the whole workflow engine**
3. `cmd/ateapi/internal/controlapi/workflow_resume.go` → `ResumeActor`, then `ensureWorkerAssigned`
4. `cmd/ateapi/internal/scheduling/scheduling.go` — small, read it whole
5. `cmd/atelet/main.go` → `Restore` (the errgroup with two legs)
6. `cmd/ateworker-gvisor/runsc.go` → `buildArgs`, `cmdRestore`

**Hands-on:** trace one activation through the logs and a trace.

```sh
kubectl port-forward -n otel-system svc/jaeger 16686:16686
kubectl ate get actor my-counter-1 -a ate-demo-counter --trace
# open http://localhost:16686, find the trace, read the step.* spans

kubectl logs -n ate-system ds/atelet | grep "timing breakdown"
kubectl logs -n ate-demo-counter -l ate.dev/worker-pool | grep "timing breakdown"
```

**Done when:** you can draw the activation path from `curl` to the actor process from memory, naming every process and socket; and you can look at a trace and say which workflow steps did real work versus fast-forwarded.

---

## Stage 2 — The three contracts

Read, in this order, each with the code open:

### 2a. The workflow / store contract
[control-plane.md](control-plane.md) → then `cmd/ateapi/internal/store/store.go`, `store/atepg/atepg.go`, and **`store/atepg/outbox.go`'s header comment** (the transactional outbox in four paragraphs — ordering, retention, durability, loss detection). Then `migrations/000001_initial.sql`, which is short and tells you most of the data model.

The two ideas to take away: **the ensure pattern** (every step derives whether its work is done from persisted state alone, so a re-entered workflow fast-forwards) and **the `uid`+`version` precondition pair** (version guards concurrent modification; uid guards name reuse across lifecycles — neither substitutes for the other).

### 2b. The node contract
[node-and-sandbox.md](node-and-sandbox.md) → then `internal/nodepath/nodepath.go` (memorize the paths), `internal/imagecache/imagecache.go`'s header (the atelet/ateworker privilege split), and `cmd/atelet/sandbox_assets.go` (the manifest, and the asset trust chain).

The idea: **`/var/lib/ate` mounted at the same path in both is the entire atelet↔ateworker data plane**, and the privilege split — atelet has `DAC_OVERRIDE` and nothing else; ateworker owns every mount — explains why the code is divided the way it is.

### 2c. The network contract
[networking-and-egress.md](networking-and-egress.md) → then **`cmd/atenet/internal/router/README.md`**, which is the best single document in the repo on any subject. Then `internal/ateworker/actornet/net.go` + `sandbox.go`, and `internal/atunnel/dns/README.md`.

The idea: **the actor's only interface is a `/30` in its own netns**, and ingress and egress apply **opposite trust models** — which is why they are separate packages that cannot import each other.

**Hands-on:** add a log line in each of the three layers (router ext_proc, a workflow step, atelet's Restore), rebuild just that component, and watch all three appear for one request.

```sh
hack/install-ate.sh --deploy-atenet
hack/install-ate.sh --deploy-ate-apiserver
hack/install-ate.sh --deploy-atelet
```

**Done when:** you could add a field that travels from a `kubectl ate` flag, through the proto, through a workflow, into an atelet RPC, into an ateworker RPC — and you know which file to touch at each hop and which codegen to re-run.

---

## Stage 3 — The rules before your first PR

These are process gates, and ignoring one will cost you a review round.

Read **all** of these before opening a PR. They are short.

1. **`AGENTS.md`** — the best single orientation file in the repo. Build commands, where code goes, and the backward-compatibility rule.
2. **`CONTRIBUTING.md`** — CLA, **discuss first**, PR sizing (S<30/M<100/L<500/XL<1000; most PRs should be L or smaller), and the commit-message rules (**describe the change and why; no `#1234` or `Fixes #1234`** — that context goes in the PR description).
3. **`docs/code-style-guide.md`** — short and entirely project-specific. Especially **"check presence, don't default it"**: a getter chain must never stand in for a presence check.
4. **`docs/dev/best-practices/ci-fail-closed.md`** — if your test has a precondition, read this. **A skip and a pass are the same exit code**, and *"a guard only ever observed passing is not known to guard anything."*
5. **[api-and-validation.md](api-and-validation.md)** if you are touching the proto — and then `docs/api-style-guide.md` and `docs/api-validation.md`, which are normative.
6. **[testing-and-dev.md](testing-and-dev.md)** for the verifiers and the e2e contract.

Four things that will be enforced:

- **A PR title becomes a release-note line.**
- **No backward compatibility before v1.0.0.** Rename and remove outright; update every caller in the same change. No proto `reserved`, no deprecated aliases.
- **`go test -race ./...` and `hack/verify-all.sh`** before asking for review.
- **A new RPC without a `defaultRPCPermissions` entry fails a test and is denied at runtime.**

### Good first-contribution shapes in this repo

- **Validation gaps.** `docs/api-validation.md` says *"all fields of all APIs must be validated"* and *"be strict"*. Finding a field with a weak bound and tightening it is a real, reviewable contribution — and the reason it matters is stated plainly: once bad data is in the database post-1.0, you cannot fix it.
- **Missing or inaccurate crash/error messages.** `controlapi/crash.go` uses fixed strings precisely so they are greppable and stable.
- **A test for a path with none**, especially a fail-closed guard whose strict branch has never been proven red.
- **Documentation gaps you hit in stages 0–2.** The docs are good and the project clearly values them; a correction is welcome.
- **Metric-registry drift.** The registry check does **not** read Go, so an instrument in the code and not in `metrics.yaml` is invisible to CI. Finding one is useful.

**Done when:** 2–3 merged PRs, and you have debugged an e2e failure from a kept namespace's worker-pod logs by yourself.

---

## Stage 4 — The hard subsystems

These are where current design work happens. Read the doc **before** the code in each case — the docs explain constraints the code only implies.

| Subsystem | Read |
|---|---|
| **The workflow engine and store** | [control-plane.md](control-plane.md) → `workflow_{resume,suspend,delete,revert,tag}.go` → `store/atepg/{outbox,lease}.go` → `docs/dev/postgresql-schema-evolution.md` |
| **Snapshots and fidelity** | [resource-model.md](resource-model.md) → `docs/api-guide.md#snapshot-storage-layout` → `cmd/atelet/{sandbox_assets,snapshot_transfer}.go`, `internal/snapshottracker` → `docs/dev/suspend-resume-phase-breakdown.md` |
| **The micro-VM runtime** | [node-and-sandbox.md](node-and-sandbox.md) → `cmd/ateworker-microvm/rootfsupper.go`'s header → `internal/ch/prefault.go` (a masterclass in encoding an upstream regression) → `internal/kata/` → `docs/dev/microvm-local.md` |
| **Egress policy and the gateway** | [networking-and-egress.md](networking-and-egress.md) → `docs/network-egress.md` (normative MUST/MUST NOT) → `cmd/atenet/internal/router/README.md` → `cmd/atenet/internal/router/egress/` → `cmd/atenet/internal/router/xds.go` |
| **Identity** | [security.md](security.md) → `docs/authentication.md`, `docs/authorization.md` → `internal/{substratex509,localca,localjwtauthority,oidcjwt,credbundle}` → `cmd/podcertcontroller/` |
| **Scheduling and hardware compatibility** | `cmd/ateapi/internal/scheduling/scheduling.go` → `internal/hardware` → the `SandboxRuntime`/`VersionedSandboxCompat` section of [resource-model.md](resource-model.md) |
| **API design** | [api-and-validation.md](api-and-validation.md) → `docs/api-style-guide.md` + `docs/api-validation.md` in full. **API mistakes are the most expensive kind here**, because bad data in the database is not fixable after 1.0. |
| **Version skew** | [install-and-operations.md](install-and-operations.md) → `docs/version-skew-policy.md` + `docs/upgrade.md`, read together |

### The code to read for pleasure and education

Not required, but each is unusually well-reasoned and will teach you how this team thinks:

- `cmd/ateapi/internal/store/atepg/outbox.go` — four mechanisms, each with its reason.
- `cmd/ateworker-microvm/internal/ch/prefault.go` — how to encode an upstream behavior change you cannot fix, including which way to be wrong.
- `cmd/ateworker-microvm/rootfsupper.go` — the host-overlay design, **plus the two retired alternatives and why they failed.**
- `internal/ateworker/pid1/` — a four-line explanation of a subtle reaping race.
- `internal/atunnel/dns/README.md` — why `NXDOMAIN` and not `REFUSED` (glibc and musl abort on `REFUSED`).
- `cmd/atenet/internal/router/README.md`'s "adding a dataplane attribute" and its naming rules: *"The prefix is a claim about who may rename the key."*
- `docs/request-parking.md` — what a budget does and does not bound, and why an in-flight attempt is never cancelled.
- `cmd/atecontroller/internal/workersync/syncer.go`'s `workerKey` comment — why the UID belongs in a workqueue key.

---

## Stage 5 — Becoming useful to the project

- **Pick one subsystem from stage 4 and go deep.** This is a small, fast-moving project; depth in one area is worth more than breadth.
- **The roadmap is the filter.** `docs/roadmap.md` lists six priorities, and `CONTRIBUTING.md` says plainly that *"PRs which are not aligned with our near-term goals may be closed without extensive review."* Read the priorities before proposing anything large:
  1. pinning down architectural decisions that influence everything else
  2. performance and reliability of wakeups, suspend/resume, snapshot management
  3. a fast, scalable control plane — storage, reliability, security
  4. useful identity, such that policies can be written in terms of identity
  5. useful policy — ingress, egress, peer networking
  6. runtime modularity across sandbox technologies
- **Show up.** The weekly community meeting is Thursdays 10:00–11:00 PST ([meeting notes](https://docs.google.com/document/d/1obSIvfcafLNniLYTQCcT2eCgxHqa2AQ3Ga7YTsju49s)); the [`ate-dev`](https://groups.google.com/g/ate-dev) Google Group; `#substrate-users` and `#substrate-dev` in the CNCF Slack.
- **Read `GOVERNANCE.md`, `MAINTAINERS.md` and `COLLABORATING.md`** for how decisions are actually made.
- **Consider extracting security-review skills from the threat model.** `docs/threat-model.md` states this as an intended outcome and nobody has done it yet.
- **Building *on* Substrate rather than changing it?** `docs/integration-repos.md` says where that code should live and how fixes flow back to core.

---

## Standing habits

- **Before changing code in an area, read the corresponding guide here and the repo doc it points at.** If a guide turns out to be stale, fix it in the same session — the source of truth is the code plus `docs/`.
- **`git log --oneline -- <path>`** on the file you are about to change tells you who to ask and what recently broke there.
- **Read the doc comment before the function.** This codebase puts the reasoning in comments at an unusually high rate, and most "why is it like this?" questions are answered in the file you already have open.
- **When a comment explains a failure mode, believe it.** The ordering constraints in `atelet`'s `Checkpoint`, the parking lot's admission point, and the `prefault.go` fallback direction are all there because something broke.
- **Check `docs/metrics/substrate.yaml`'s `blind_spots` before attributing a fault to a component.** The store has no instruments, and every lifecycle operation uses it.
- **Treat `docs/security.md`'s status columns as the truth** about what exists, not the README's pitch.

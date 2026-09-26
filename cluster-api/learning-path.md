# Learning path: from Kubernetes engineer to Cluster API maintainer

A staged reading and practice order for someone who knows Kubernetes well but is new to Cluster API. Each stage lists what to read (in this repo), what to do hands-on, and what "done" looks like. The concept guides referenced here are in `.claude/concepts/`.

## Stage 0 — Get the mental model (a few hours)

Read, in order:

1. `docs/book/src/user/concepts.md` — the vocabulary: management cluster vs workload cluster, the four provider types, Cluster/Machine/MachineSet/MachineDeployment.
2. `docs/book/src/user/manifesto.md` — the project's values. Worth reading properly; it explains *why* reviews look the way they do and is referenced in `CONTRIBUTING.md`.
3. `.claude/concepts/architecture.md` — the repo map and the manager binaries.
4. `docs/book/src/developer/core/overview.md`. Skip `repository-layout.md` — it predates the restructure in `docs/proposals/20260701-code-organization.md`; read that proposal instead for the current layout and its rationale.

Hands-on: run the quick start with the Docker provider (`docs/book/src/user/quick-start.md`) so you have a real management cluster and a real workload cluster to poke at. Then `clusterctl describe cluster <name>` and read the object graph you just created with `kubectl`.

Done when: you can explain, without looking, what objects exist for one workload cluster and which controller owns each.

## Stage 1 — The core reconcile loops (the heart of the project)

Read `.claude/concepts/core-objects.md` alongside the code. Then read the official per-controller pages, which are short and good:

- `docs/book/src/developer/core/controllers/cluster.md`
- `docs/book/src/developer/core/controllers/machine.md`
- `docs/book/src/developer/core/controllers/machine-set.md`
- `docs/book/src/developer/core/controllers/machine-deployment.md`
- `docs/book/src/developer/core/controllers/machine-health-check.md`

Hands-on: get the local dev loop working — `make kind-cluster`, then `make tilt-up` (see `.claude/concepts/testing-and-dev-workflow.md`). Add a log line in the Machine reconciler, watch it reload, then delete a Machine and follow the deletion sequence in the logs.

Done when: you can trace a `kubectl apply` of a Cluster through to a running Node by naming the controllers and the fields they set, in order.

## Stage 2 — Contracts and providers

Read `.claude/concepts/provider-contracts.md`, then `docs/book/src/developer/providers/contracts/` (one page per contract), then `.claude/concepts/control-plane-and-bootstrap.md`.

Hands-on: read CAPD (`test/infrastructure/docker/`) end to end — it is small and implements the full infrastructure contract. Note the current shape: the types are `DevCluster`/`DevMachine` (`test/infrastructure/docker/api/v1beta2/`) with pluggable backends under `reconcilers/backends/`, and the in-memory provider is one of those backends rather than a standalone provider. Then read one KCP scale-up and one KCP upgrade path in `controlplane/kubeadm/reconcilers/kubeadmcontrolplane/` and `controlplane/kubeadm/pkg/`.

Done when: you could review a PR to an out-of-tree provider and say whether it satisfies the contract.

## Stage 3 — First contributions

Target `good-first-issue` items (`help wanted` ones assume deeper knowledge — see `.claude/concepts/contributing-and-review.md`). Good first-contribution shapes in this repo:

- Condition/message accuracy fixes in a reconciler.
- Missing or wrong validation in an admission webhook (`core/webhooks/admission/`, or the provider equivalents).
- Flaky or missing unit tests, especially around the `util/test/builder` object builders.
- Book documentation gaps you hit during stages 0–2 — label 📖; these are real contributions and get you known.

Process gates to internalize before your first PR: the `triage/accepted` requirement, the PR-title emoji prefix, and the AI guidance (write review replies yourself). All in `.claude/concepts/contributing-and-review.md`.

Done when: 2–3 merged PRs, and you have triaged an e2e failure from Prow artifacts by yourself.

## Stage 4 — The hard subsystems

These are where most of the current design work happens, and where reviewer bandwidth is scarcest — becoming useful here is the fastest route up the ladder.

- `.claude/concepts/clusterclass-topology.md` + `docs/book/src/developer/core/controllers/cluster-topology.md` + the CAEP `docs/proposals/20210526-cluster-class-and-managed-topologies.md`.
- `.claude/concepts/runtime-sdk.md` + `docs/proposals/20220221-runtime-SDK.md` and `docs/proposals/20220330-topology-mutation-hook.md`.
- `.claude/concepts/api-and-codegen.md` + `docs/book/src/developer/core/API.md` — API versioning and conversion is the area where mistakes are most expensive.
- `.claude/concepts/clusterctl.md` — `clusterctl move` in particular; it breaks whenever new API types forget the move contract.

Read the matching CAEP in `docs/proposals/` *before* the code for each of these. The proposals explain constraints that the code only implies.

## Stage 5 — Toward reviewer / maintainer

- Pick one sub-area from the OWNERS table in `.claude/concepts/contributing-and-review.md` and go deep. Ownership in this project is per-sub-area, not global.
- Start reviewing other people's PRs in that area even before you have `lgtm` rights; leave substantive, specific review comments (write them yourself — see the AI guidance).
- Help with issue triage: add `kind/*` and `priority/*` reasoning, reproduce bugs, close duplicates. `CONTRIBUTING.md:162` says community help here is explicitly needed.
- Show up at office hours (Wednesdays 10:00 PT) and say what you are working on.
- Consider joining a release team (`docs/release/role-handbooks/`) — it is a well-trodden path to visibility.
- Then: OWNERS-file PR, announce at the community meeting, one week of lazy consensus.

## Standing habits

- Before changing code in an area, read the corresponding `.claude/concepts/` guide; if the guide turns out to be wrong or stale, fix the guide in the same session.
- Before proposing a design, search `docs/proposals/` — the question has usually been argued before.
- `git log --oneline -- <path>` on the file you are about to change tells you who to ask and what recently broke there.

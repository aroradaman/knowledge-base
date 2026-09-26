# Contributing, review, and release process

How work actually gets merged in kubernetes-sigs/cluster-api: the AI guidance that constrains how this repo may be used with assistants, issue triage gates, PR conventions enforced by CI, Prow commands, backports, the CAEP process, and the OWNERS/sub-area structure that defines the path to reviewer and maintainer. Source of truth is `CONTRIBUTING.md`; this file is the operational summary.

## AI guidance (read this first)

`CONTRIBUTING.md:19` ("AI Guidance") aligns the project with the [Kubernetes AI guidance](https://www.kubernetes.dev/docs/guide/pull-requests/#ai-guidance). It is explicit and it applies to this repo specifically:

- Maintainers want to engage with the contributor, not a bot.
- **Concise PR descriptions are preferred over lengthy AI-generated ones.**
- **Responses to review comments must be written by the contributor without relying on AI tools.**
- AI-generated issue/PR comments are assumed to be ignored; repeatedly posting them is reportable.

Practical rules when assisting in this repo:

- Never draft replies to reviewer comments for the contributor to paste. Help them understand the review instead — explain what the reviewer means, find the relevant code, propose a code change. The prose reply is theirs.
- Keep PR descriptions short: what changed, why, `Fixes #NNNN`. No generated multi-section essays, no bullet-point walls, no test-plan boilerplate that duplicates CI.
- Do not post issue comments, triage comments, or review comments on the contributor's behalf.
- Code assistance is fine and expected ("use AI wisely to amplify your impact"); the constraint is on community interaction, not on writing code.
- Attribution/co-author trailers on commits pushed to this upstream repo are a judgement call for the contributor — ask before adding them.

## Before you write code: the issue gate

- Non-trivial changes require an issue, and the issue must carry the **`triage/accepted`** label. A PR for an untriaged issue is likely to be closed immediately (`CONTRIBUTING.md:75`).
- Triage labels to look for: `needs-triage` (auto-applied) -> one of `triage/accepted`, `triage/needs-information`, `triage/not-reproducible`, `triage/duplicate`. Initial triage also requires `priority/*` and `kind/*` labels.
- `good-first-issue` = no deep system knowledge needed. `help wanted` = deeper knowledge required, explicitly *not* recommended for newcomers.
- Assign yourself to signal you're working on it. Non-members of the kubernetes-sigs org cannot self-assign; ask a maintainer to `/assign <github-id>`.
- If the issue says "needs research", share the research outcome and get consensus on the issue *before* opening a PR.
- Big features, API changes, or contract changes need a CAEP (see below), not just an issue.

## PR conventions

PR **title** format is enforced in CI by `hack/scripts/verify/verify-pr-title.sh` (workflow `.github/workflows/pr-verify.yaml`). Rules:

| Prefix | `:emoji:` form | Use for |
| --- | --- | --- |
| ⚠ | `:warning:` | breaking / major change |
| ✨ | `:sparkles:` | non-breaking feature |
| 🐛 | `:bug:` | patch / bugfix |
| 📖 | `:book:` | docs or proposals |
| 🚀 | `:rocket:` | release |
| 🌱 | `:seedling:` | infra, tests, chores, other minor |

- The prefix is mandatory and must come first. A leading `WIP` marker or a `[tag]` is stripped before the check, so `[WIP] 🐛 fix ...` and `WIP 🐛 fix ...` both pass.
- The title must **not** contain an issue or PR number (`#123`). Issue references go in the body: `Fixes #123` or `Related to #123`.
- Unfinished work gets the `[WIP]` prefix.
- Sign the [Kubernetes CLA](https://github.com/kubernetes/community/blob/main/CLA.md) once, before your first PR.
- Docs changes: source lives in `docs/book/src/`, preview with `make serve-book` (fetches mdBook into `hack/tools/bin/`, serves on `http://localhost:3000`), label the PR 📖.

## Prow commands and CI signal

CI is Prow-driven (not GitHub Actions for the heavy jobs; GH Actions only runs lint, PR-title verify, dependabot, and link checks — see `.github/workflows/`). Useful commands as PR/issue comments:

- `/retest`, `/test <job-name>` — rerun failed jobs.
- `/lgtm`, `/approve` — review gates; both are needed (`lgtm` from a reviewer, `approve` from an OWNERS approver for each touched path).
- `/hold`, `/hold cancel` — block merge.
- `/assign <id>`, `/cc <id>`, `/kind bug`, `/area clusterctl`, `/priority important-soon`.
- `/cherry-pick release-1.x` — open a backport PR after merge (see below).

Triaging failures (`CONTRIBUTING.md:218`):

- Lint/unit failures reproduce locally with the make target named in the Prow log — usually `make lint`, `make test`, or a `make verify-*`.
- E2E failures: open the **Artifacts** link at the top of the Prow log. It contains the generated clusterctl local repository (components YAML + cluster templates), controller logs and a dump of Cluster API resources for management clusters, and machine logs for workload clusters.
- Periodic job results for all supported branches are on [testgrid](https://testgrid.k8s.io/sig-cluster-lifecycle-cluster-api).

## Branches and backports

- `main` is where development happens, including breaking changes.
- `release-X.Y` branches are stable and backwards compatible; minor/patch releases are tagged from them.
- Backport by commenting `/cherry-pick release-X.Y` on the merged main PR. Only backward-compatible changes are eligible; maintainers police this strictly to protect release-branch stability.
- Opening a fix directly against a release branch is possible but should be rare.
- Release cycle, supported versions, and the release team process live under `docs/release/` (`release-cycle.md`, `releases/`, `role-handbooks/`).

## CAEP (Cluster API Enhancement Proposal)

Required for new features, API changes, contract changes between components, and CLI interface changes.

- Template and accepted proposals live in `docs/proposals/` (filenames are `YYYYMMDD-topic.md`). Reading these is the fastest way to understand *why* a subsystem looks the way it does — e.g. `20210526-cluster-class-and-managed-topologies.md`, `20220221-runtime-SDK.md`, `20191017-kubeadm-based-control-plane.md`, `20200506-conditions.md`.
- Every proposal must be tied to an issue, should be discussed at the weekly community meeting or on the SIG Cluster Lifecycle mailing list, starts as a collaborative doc (Google Docs preferred), then becomes a PR, and must be merged in `implementable` state to count for a release.

## OWNERS and the contributor ladder

Ownership is split into sub-areas, each with its own OWNERS file. This is the mechanism for stepping up: pick a sub-area, build a review track record there, then (1) open a PR adding yourself to that OWNERS file, (2) announce it at the community meeting, (3) collect +1s and wait one week of lazy consensus (`CONTRIBUTING.md:243`).

Current sub-areas with their own OWNERS:

| Sub-area | Path |
| --- | --- |
| clusterctl | `cmd/clusterctl/` |
| kubeadm Bootstrap Provider (CABPK) | `bootstrap/kubeadm/` |
| kubeadm Control Plane Provider (KCP) | `controlplane/kubeadm/` |
| ClusterClass / managed topologies | `core/reconcilers/topology/` |
| Docker infra provider (CAPD) | `test/infrastructure/docker/` |
| in-memory infra provider | `test/infrastructure/inmemory/` |
| Test / test framework | `test/`, `test/framework/` |
| Docs | `docs/` |
| MachinePools | `core/reconcilers/machinepool/` |
| Ignition support | `bootstrap/kubeadm/pkg/ignition/`, `test/infrastructure/docker/internal/provisioning/ignition/` |

Root `OWNERS` delegates to the `cluster-api-maintainers` / `cluster-api-reviewers` / `cluster-api-admins` aliases defined in `OWNERS_ALIASES`. Use `git log --format='%an' -- <path>` on a sub-area to see who is active there before asking for review.

## Community

- Slack `#cluster-api` on kubernetes.slack.com.
- Office hours weekly, Wednesdays 10:00 PT (Zoom link and agenda in `CONTRIBUTING.md:41`).
- Subscribe to the SIG Cluster Lifecycle Google Group for docs and calendar access.

## Where to read more

- `CONTRIBUTING.md` — authoritative; re-read the AI guidance and triage tables rather than trusting this summary if they conflict.
- `docs/book/src/developer/getting-started.md`, `docs/book/src/developer/overview.md` — developer onboarding.
- `docs/book/src/developer/core/repository-layout.md` — official layout description.
- `docs/release/release-cycle.md` — release cadence and branch support policy.
- `docs/proposals/` — the design history of every major subsystem.

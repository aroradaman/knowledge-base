# Cluster API Knowledge Base

Comprehensive learning and reference guides for Kubernetes Cluster API (CAPI), built for contributors moving toward maintainership of `kubernetes-sigs/cluster-api`.

## Quick Start

If you're new to CAPI, start with **[learning-path.md](learning-path.md)** — it's a staged roadmap from quick-start to reviewer/maintainer status, with hands-on milestones at each stage.

## Concept Guides

Read the relevant guide before working in an area. They assume you know Kubernetes deeply (controllers, CRDs, informers, webhooks, kubeadm) and explain only what is CAPI-specific.

| When you're working on | Read |
| --- | --- |
| **Anything new** | Start with [architecture.md](architecture.md) for the 10k-ft map |
| Core controllers and object lifecycle | [core-objects.md](core-objects.md) — Cluster/Machine/MachineSet/MachineDeployment/MHC reconcilers, conditions, deletion ordering |
| KCP, CABPK, or bootstrap/control-plane logic | [control-plane-and-bootstrap.md](control-plane-and-bootstrap.md) — etcd, certs, workload-cluster access, remediation |
| ClusterClass, managed topologies, desired-state | [clusterclass-topology.md](clusterclass-topology.md) — patch engine, SSA ownership, rollouts |
| Lifecycle hooks, topology mutation hooks, ExtensionConfig | [runtime-sdk.md](runtime-sdk.md) — hook catalog, discovery, failure policies |
| Provider contracts, unstructured object access | [provider-contracts.md](provider-contracts.md) — contract fields, labels/annotations, CAPD reference impl |
| clusterctl, move, init, upgrade | [clusterctl.md](clusterctl.md) — object graphs, template processing, what makes a CRD movable |
| API types, conversions, webhooks, code generation | [api-and-codegen.md](api-and-codegen.md) — hub/spoke, `make generate` targets, add-a-field recipe |
| Tests, envtest, e2e, Tilt, pre-submit | [testing-and-dev-workflow.md](testing-and-dev-workflow.md) — test layers, GINKGO_*, local loop |
| Issues, PRs, triage, backports, CAEPs, OWNERS | [contributing-and-review.md](contributing-and-review.md) — AI guidance, Prow commands, the contributor ladder |

## Key Assumptions

- **v1beta2 is the storage/served version.** `api/core/v1beta2` is imported everywhere as `clusterv1`; `v1beta1` exists for conversion only.
- **Layout is post-2026-reorg.** Every component has the shape `{core,bootstrap,controlplane}/{{reconcilers,webhooks,setup,config,pkg}/`. No `internal/controllers/`, no root `main.go`. If docs say otherwise, check `docs/proposals/20260701-code-organization.md` — it's the authority.
- **CONTRIBUTING.md AI guidance applies.** Keep PR descriptions short; draft review replies yourself (not AI-written); AI-generated comments get ignored.
- **Non-trivial PRs need a `triage/accepted` issue first.** PR titles must start with one of ⚠ ✨ 🐛 📖 🚀 🌱.

## Index

- **[architecture.md](architecture.md)** (126 lines) — provider flavors, manager binaries, modules, cross-cutting machinery
- **[core-objects.md](core-objects.md)** (203 lines) — the five core reconcilers and the object graph
- **[control-plane-and-bootstrap.md](control-plane-and-bootstrap.md)** (198 lines) — KCP/CABPK specifics and contracts
- **[clusterclass-topology.md](clusterclass-topology.md)** (151 lines) — desired-state computation, patch engine, SSA
- **[runtime-sdk.md](runtime-sdk.md)** (154 lines) — hooks, discovery, failure policies
- **[provider-contracts.md](provider-contracts.md)** (183 lines) — contract fields, labels, move rules, CAPD
- **[clusterctl.md](clusterctl.md)** (148 lines) — commands, object graphs, movability
- **[api-and-codegen.md](api-and-codegen.md)** (176 lines) — versions, conversions, webhooks, generators
- **[testing-and-dev-workflow.md](testing-and-dev-workflow.md)** (183 lines) — test layers, Tilt, pre-submit
- **[contributing-and-review.md](contributing-and-review.md)** (114 lines) — triage, Prow, CAEPs, maintainership path
- **[learning-path.md](learning-path.md)** (77 lines) — staged milestones and reading order

## Built for

An experienced Kubernetes engineer who is new to Cluster API and aiming to become a maintainer of `kubernetes-sigs/cluster-api`. The guides skip generic Kubernetes explanation and focus on CAPI-specific concepts, contracts, and design decisions.

## How to use

- **First time:** Read [learning-path.md](learning-path.md) for the staged roadmap.
- **Before working in an area:** Skim the relevant concept guide from the table above.
- **When a guide is stale:** Fix it (specs change, repos reorganize). The source of truth is the code and `docs/proposals/` in the CAPI repo.

## Feedback

If you find errors or stale information, open an issue or PR in the CAPI repo itself and update the guides in `.claude/concepts/` there — they are part of the project's knowledge infrastructure.

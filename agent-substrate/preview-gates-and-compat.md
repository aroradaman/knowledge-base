# Preview Gates and Compatibility

Two related mechanisms with opposite timelines. **Preview gates** let a feature ship while off by default, and they work today. **The version-skew policy** says which releases may run together, and it applies **from 1.0.0** — before then, `AGENTS.md` forbids compatibility code entirely.

Authorities: `docs/dev/preview-gates.md`, `docs/version-skew-policy.md`, `AGENTS.md`, `internal/preview/`.

---

# Part 1 — Preview gates

> *"A preview gate lets a new feature ship in a release while staying off by default. This gives users an opportunity to try things but protects them from accidentally using things that we are not confident in."*

**There are exactly two states: preview (gated, off unless enabled) or GA (always on). There is no alpha or beta.** And explicitly: **gates are coarse switches, not access control.**

## Two independent switches, both required

| | Switch |
|---|---|
| **Server side** | `--preview=<gates>` on **every** Substrate binary — a comma-separated list of gate names, or `ALL`. Install-wide via ate-setup's `preview` setting (`ATE_PREVIEW`, `--preview=<gates>`), which passes the same value to **every** Substrate container. **`atecontroller` passes its own gates on to the `ateworker` containers of each WorkerPool.** |
| **Client side** | a gRPC request to ate-api must send `Ate-Preview-Allowed: true` (`preview.Header`). `kubectl ate --preview` sends it. **A request without it is treated as if every gate were off.** |

Substrate's own components (`atelet`, `atenet`, `atecontroller`) dial ate-api with `preview.DialOptionsWithOptIn()`, so they are **never blocked by the header check**. `ate-setup` does the same when run with preview gates.

## Declaring a gate

`internal/preview/preview.go`:

```go
const (
	GateExternalVolumes Gate = "ExternalVolumes"
	GateMyFeature       Gate = "MyFeature"
)

var realKnownGates = []Gate{
	GateExternalVolumes,
	GateMyFeature,
}
```

**Gate names are always UpperCamelCase.** The gates are **process-wide**: a binary calls `preview.Init` exactly once, before use.

## Gating an API field

Gate the fields a **user** sets with a declarative-validation tag in the `.proto`, then regenerate (`hack/update/codegen.sh`):

```proto
  // +ate:preview=MyFeature
  // +k8s:ifDisabled(MyFeature)=+k8s:forbidden
  // +k8s:optional
  MyFeatureConfig my_feature = 7;
```

- **`+ate:preview=<Gate>` enforces nothing** — it marks the field as part of a preview feature, for readers and docs.
- **`+k8s:ifDisabled(<Gate>)=+k8s:forbidden`** is what actually rejects the field when the gate is off for this request.

### The trap that makes gating silently not work

The gate state reaches the validators through `apivalidation.MakeCreateOp(ctx)` and `apivalidation.MakeUpdateOp(ctx)`:

> *"Every `Validate*` wrapper **must** build its `operation.Operation` with one of these, not with a bare `operation.Operation{...}` literal; **otherwise the field is never gated.**"*

### Two more rules

- **Unchanged values are not re-validated on update**, so an object created while a gate was on **stays usable after the gate is turned off.**
- **Do not gate fields the server writes**, such as `status`:

> *"The server writes them while doing system work, such as deleting an actor, and gating them can leave objects that cannot be cleaned up after a gate is turned off. This is subtle and will require careful thinking when implementing a gated feature."*

## Gating behavior in code

For a check made **while handling a user request**, use the context-aware form:

```go
if !preview.IsEnabledFor(ctx, preview.GateMyFeature) {
	return nil, resources.ToAPIError(field.ErrorList{field.Forbidden(
		path, "requires the MyFeature preview gate and a client that opts in to "+
			"preview features (kubectl ate --preview, or the "+preview.Header+" header)")})
}
```

Use **`preview.IsEnabled(gate)` only for checks about the process, not the request** — e.g. whether a component should start a feature at all.

> *"Be deliberate about what happens when a gate is turned **off** while the feature is in use. **Prefer to keep cleanup paths (delete, suspend, detach) working regardless of the gate**, so that turning it off cannot strand resources."*

## Request context and workflows

ate-api's gRPC interceptor marks a request's context as **opted out** unless it sent the header. Work that a request starts but that runs **on the system's behalf** must not inherit that restriction, so every exported `ActorWorkflow` and `WorkerWorkflow` method begins with:

```go
ctx = preview.ContextWithOptIn(ctx)
```

**`TestWorkflowsOptInToPreview` enforces this.** Contexts that never went through the interceptor — background reconcilers — are not opted out.

## Testing a gated feature

| Layer | How |
|---|---|
| Unit | `preview.InitForTest(t, "MyFeature")`, or with no names for all-off. It restores the previous state at test end. **Tests that use it cannot run in parallel** (gates are process-wide). |
| The gate mechanism itself | `preview.InitForTestFake` with **fake** gate names, so those tests don't depend on real gates |
| `apivalidation` tables | `assertValidate` takes `want` (gates off) **and** `wantWithPreview`, keyed by a `--preview` value such as `"ALL"` |
| Functional (`controlapi/functionaltest`) | `setupPreviewTest` gives a client that sends the header; `tc.noOptInClient` one that does not. **Neither enables a gate** — call `preview.InitForTest` for that. |
| E2E | `e2e.RequirePreview(t, preview.GateMyFeature)` skips unless `--preview` names that gate, **which must match what the cluster was installed with** |

```shell
hack/run-e2e-kind.sh -v -args --preview=ALL
```

**CI runs every e2e lane with preview gates both off and on, and requires `--preview` to be passed (even as empty) so a lane cannot silently skip every preview test.**

**Always test gated code both with and without the gate.**

## Graduating or removing a gate

> *"When the feature goes GA, or is removed, **keep the gate name in `realKnownGates` for a while** so existing `--preview` values still parse, but make the feature always on (GA) or always off (removed). Then delete the `ifDisabled` tags, `IsEnabled`/`IsEnabledFor` checks, and `RequirePreview` calls."*

Open TODOs in the doc: how to gate a new RPC method or service, and how to gate a new CLI feature.

---

# Part 2 — Before 1.0.0: no compatibility code at all

`AGENTS.md`, and it shapes every review:

> *"Agent Substrate makes no compatibility guarantee before v1.0.0. **Do not add code to stay compatible with older clients, binaries, protos, flags, or config.** Remove or rename fields, RPCs, flags, and types outright and update every caller in the same change. Do not add `reserved` statements to protos, deprecated aliases, fallbacks for old formats, or version negotiation."*

This is why `ActorState` has gaps at 5 and 6, and why you will not find a single `Deprecated:` alias in the tree. (The section carries `<!-- TODO #2004: Remove this section at the v1.0.0 release. -->`.)

Upgrade reality today, from `docs/upgrade.md`:

| From → to | Supported |
|---|---|
| a minor before 1.0, e.g. 0.3.x → 0.4.y | **No. Reinstall.** *"Before 1.0, breakage still exists for minor versions."* |

---

# Part 3 — From 1.0.0: the version-skew policy

`docs/version-skew-policy.md`. It says **which releases may run together** and **in what order to upgrade** so only supported combinations ever exist. `docs/upgrade.md` implements it and may add restrictions of its own.

Scope: moving an installation between two **compatible** releases (every change between them compatible under the compatibility policy), and the mixture that exists while that happens. **Between incompatible releases, reinstall.** From 1.0.0, the patch and minor releases of one major are compatible. All components of a release are built from one commit and carry one version string.

**Patch releases add no API fields, public or internal, and no schema migrations.**

## The component order

```
podcertificate-controller → atelet → workers → ate-api-server → ate-controller → atenet → clients
```

- **A component may be one release newer than any component *after* it in this order.**
- **A component must not be newer than any component *before* it.**
- While a component rolls, its own replicas/nodes/pods are at N and N+1.

So during an N→N+1 upgrade these combinations are supported: a new `atelet` driving old `ateworker` processes on its node; a new `atelet` calling an old `ate-api-server`; an old `ate-controller` rendering Deployments for new workers; old `ate-controller` and clients talking to a new `ate-api-server`.

**The reverse pairs are not supported.** In particular: **a worker must not be newer than the atelet on its node, and `ate-api-server` must not be newer than any worker.** The stated reason:

> *"`ate-api-server` reaches the workers through `atelet`, and a new `ate-api-server` may rely on new worker behavior behind existing fields, so the workers go first."*

## The two schemas behave differently during a roll

| | Shared? | Consequence |
|---|---|---|
| **API schema** | **not shared** | Until every `ate-api-server` replica is at N+1, a request can reach an N replica, which **rejects fields new in N+1**. So **clients must not use new fields until `ate-api-server` finishes rolling.** |
| **Database schema** | **shared** | The **first N+1 replica migrates it**, so replicas at N and N+1 both run against the **N+1 schema** while `ate-api-server` rolls. |

## What each release must keep true

The skew rules only hold if every release maintains all of the following.

### Worker pod template
`ate-controller` upgrades **after** the workers, so workers start the N+1 image with the Deployment **N** rendered. **Within a major, `ate-controller` renders the same worker pod template for the same WorkerPool.** New `ateworker` behavior comes from the image or a new WorkerPool field, **not from the template** — so `ateworker` accepts the flags and environment the template passes for the whole major, and **no WorkerPool rolls at `ate-controller`'s step**, in an upgrade or a rollback.

### API fields — what each component does with a field it does not know

**Servers:**

| Component | Behavior |
|---|---|
| `ate-api-server`, on a **request** | **rejects** with `INVALID_ARGUMENT` |
| `ate-api-server`, on a **stored object** | **drops the unknown fields and serves the reduced object.** *"Such a field was written by a newer replica, during ate-api-server's roll or before a rollback, and this release cannot validate it. Rejecting it would make the object unreadable. Keeping it would serve a field this release cannot validate."* |
| `atelet` | **rejects.** Its callers (`ate-api-server`, `ateworker`) are never newer in the supported order, so an unknown field means **the upgrade order was not followed.** |
| `ateworker` | **rejects.** Its caller `atelet` *can* be one release newer, so **atelet sends a new field only on releases after `ateworker` learns it** — a two-step release. |

**Clients:**

- A client that reads an object, changes it and writes it back **keeps the unknown fields.** *"An older client must not strip fields that a newer server still understands."*
- `kubectl-ate` **rejects a manifest file with fields it does not know**, as the server does for a create.

> **Wherever a caller can be newer than its server, the caller sends a new field only one release after the server learns it (two-step releases). Today that is `atelet` calling `ate-api-server`, and `atelet` calling `ateworker`.**

This is also why `docs/api-style-guide.md` tells clients to **modify the message they read rather than reconstruct it**, and never to round-trip a resource through JSON: `protojson` cannot represent unknown fields and **silently drops them**.

### CRDs
The N+1 CRDs are applied with `ate-controller` N+1. Components that upgrade before it read the N CRDs, **so users must not set new CRD fields until the upgrade completes.** Between compatible releases a CRD change only **adds optional fields**. The CRDs serve one version with no conversion — **so a rollback that re-applies the N CRDs removes every field N does not know from every object.**

### SandboxConfig
Each release ships its SandboxConfig under its own name and **keeps the earlier ones.** The upgrade adds the N+1 one, and WorkerPools without a `configRef` switch to it. **That does not change the pod template, so workers do not roll.**

**An actor resolves its pool's SandboxConfig each time it starts.** *"If the runtime its snapshot needs is gone, it cold boots, keeping its files and losing its memory."* Rollback deletes the N+1 SandboxConfig.

### Database schema
Updated with `ate-api-server`, following `docs/dev/postgresql-schema-evolution.md`: **every migration keeps working for the previous `ate-api-server` binary**, and **migrations are never rolled back.** Gated by `hack/verify/postgresql-migrations.sh` and a dedicated CI step.

### Data on nodes and in snapshots

Substrate's own data only — *"The sandbox runtime's checkpoint files follow the runtime's own rules: a restore across runtime versions keeps only the file systems."*

**Node layout** (the directory structure and files `atelet` keeps under `/var/lib/ate`):
- **What N writes, N+1 can read. What N+1 writes, N can read.**
- **So a layout change takes two releases**: N+1 reads the old and the new layout but still **writes the old one**; N+2 writes the new one. *"This keeps a rollback possible."*
- **`atelet` passes the per-actor directories to `ateworker` in the RPC, so only `atelet` knows the layout.** The sockets they find each other by do not change within a major.

**Snapshot `manifest.json`** (written and read by `atelet`):
- Within a major, a release reads the snapshot data **any earlier release of that major** wrote.
- **A new field must be safe to ignore.** A release ignores manifest fields it does not know, **so a snapshot taken at N+1 still restores after a rollback to N.**

### The system bundle
Applied **last**, and **a rollback does not undo it.** So a component at N+1 must work with these resources at N, **and** a component at N must work with them at N+1.

> *"Changes that only take something away, such as a NetworkPolicy that denies traffic, can go in this step. **Anything a new component needs to run, such as a new Secret or a rule that allows its traffic, ships in that component's own step.**"*

## The upgrade order

```
1. podcertificate-controller
2. atelet, node by node
3. every WorkerPool's spec.workerImage   (this rolls its workers)
4. ate-api-server                        (the first N+1 replica migrates the schema)
5. the CRDs, then ate-controller
6. atenet
7. the SandboxConfig admission policy + the default SandboxConfig
8. the rest of the Substrate system bundle
9. clients
```

**Rollback is the same list in reverse with release N, except the schema migration and step 8 are not undone.** Clients stop sending N+1 fields before `ate-api-server` goes back, and **`atelet` goes back only after every worker is back at N.** Rolling back further than N is not supported.

### Skipping releases

| | |
|---|---|
| **Patch** releases | **can be skipped** — `x.y.1` → `x.y.5` directly |
| **Minor** releases | **cannot** — to go `x.1` → `x.3`, upgrade to `x.2` first |

> *"The two-release steps and the database schema evolution both assume each upgrade moves one minor release."*

## Outside this policy

The Kubernetes version, the database engine, the node OS, the CSI drivers and the actor images are **not changed by a Substrate upgrade.**

## Testing

> *"Before a release is cut, an installation at the previous compatible release is upgraded to it **with actors running** and **rolled back**, in the order above."*

---

## Checklist: adding a preview-gated feature

1. Declare the gate in `internal/preview/preview.go` (UpperCamelCase) and add it to `realKnownGates`.
2. Gate user-set API fields with **both** `+ate:preview=<Gate>` and `+k8s:ifDisabled(<Gate>)=+k8s:forbidden`. **Never gate `status` or anything the server writes.**
3. Make sure every `Validate*` wrapper builds its operation with `apivalidation.MakeCreateOp/MakeUpdateOp` — **a bare `operation.Operation{}` literal silently disables gating.**
4. Use `preview.IsEnabledFor(ctx, …)` for request-scoped checks, `preview.IsEnabled(…)` only for process-scoped ones.
5. **Keep cleanup paths working regardless of the gate**, so turning it off cannot strand resources.
6. Any new workflow method starts with `preview.ContextWithOptIn(ctx)`.
7. Test with the gate **on and off**, at unit, validation-table, functional and e2e layers. Mark e2e tests with `e2e.RequirePreview`.
8. `hack/update/codegen.sh`, then `go test -race ./...` and `hack/verify-all.sh`.

## Where to read more

- `docs/dev/preview-gates.md` — the authority; short and complete.
- `docs/version-skew-policy.md` — the authority; read alongside `docs/upgrade.md`.
- `docs/dev/postgresql-schema-evolution.md` — the migration rules the policy depends on.
- `internal/preview/preview.go` — the gate list and the helpers.
- [api-and-validation.md](api-and-validation.md) — how the DV tags work.
- [install-and-operations.md](install-and-operations.md) — the upgrade runbook, step by step.

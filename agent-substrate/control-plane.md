# The Control Plane (`ate-api-server`)

`cmd/ateapi` — ~80k lines, the largest and most important component. It owns the Actor lifecycle, schedules Actors onto Workers, coordinates snapshots, mints actor identity, and enforces authn/authz. Everything else in the system either calls it or is called by it.

## Three layers, and the rule about each

`cmd/ateapi/internal/controlapi/service.go` defines the layering, and the doc comments state the rule explicitly:

| Layer | Type | Rule |
|---|---|---|
| **RPC** | `RPCService` (implements `ateapipb.ControlServer`) | "as light as possible" — validate, convert, delegate |
| **Middleware** | `ServiceImpl` (implements `store.Interface`) | "most of the logic" — invariants, validation, cross-resource rules |
| **Store** | `store.Interface` → `atepg.Persistence` | PostgreSQL, preconditions, leases, the outbox |

Above those sit the **workflows** (`ActorWorkflow`, `WorkerWorkflow`) for anything multi-step, and the **reconcilers** (`ActorTemplateReconciler`, `WorkerAssignmentReconciler`) for background convergence.

A recurring idiom: each workflow and reconciler declares a **narrow interface listing the exact store methods it needs and nothing more** (`actorWorkflowStore`, `workerWorkflowStore`, `templateReconcilerStore`, `serviceStore`). When you add a store call to a workflow you must widen that interface deliberately. `ServiceImpl.store` is named explicitly rather than embedded "to prevent accidentally satisfying methods we need to trap."

## The workflow engine

`cmd/ateapi/internal/controlapi/workflow.go` + `workflow_{resume,suspend,delete,revert,tag,template_delete,worker_delete,reconcile_assignments}.go`.

### The ensure pattern — the single most important idea in this component

Every workflow step:

1. **Derives whether its work is already done from persisted state alone** (and calls `markSkipped(ctx, reason)` if so).
2. **Validates the state-machine edge** it is about to take.
3. **Persists what it changed** before returning.

So a re-entered workflow **fast-forwards** to wherever the previous attempt stopped. There is no durable workflow log — progress is reconstructed from the Actor record. Idempotency is not a bonus here; it is the mechanism.

Each step opens a trace span named `step.<Name>` via `stepSpan(ctx, name)`, which on error records it on the span and wraps it: `workflow failed at step <name>: %w`. A skipped step is annotated `step.skipped=true` + `step.skip_reason`, **so a trace shows exactly which steps fast-forwarded and where real work restarted.** This is your primary debugging tool for a stuck lifecycle operation.

### Step names, by workflow

| Workflow | Steps |
|---|---|
| **Resume** | `LoadActorForResume` → `CreateVolumes` → `AssignWorker` → `AttachVolumes` → `CallAteletRestore` → `FinalizeRunning` |
| **Suspend / Pause** | `LoadActorForSuspend` → `MarkSuspending` → `CallAteletSuspend` (or `PromoteSnapshot` for a paused-origin suspend) → `FinalizeSuspended` → `ReleaseReplacedSnapshot` |
| **Revert** | `LoadActorForRevert` → `MarkReverting` → `DiscardWorker` → `DiscardInProgressSnapshot` → `FinalizeReverted` |
| **Delete** | `LoadActorForDelete` → `CallAteletTerminate` → `DetachVolumesForDelete` → `ReleaseWorker` → `MarkDeleting` → `DeleteVolumes` → `ReleaseExternalSnapshots` → `FinalizeDeleted` |
| **CreateTag** | `LoadActorForTag` → `ReserveTag` → `RecordTagSnapshot` → `CopyTagSnapshot` → `FinalizeTag` |
| **DeleteTag** | `LoadTagForDelete` → `ReleaseTagSnapshot` → `FinalizeTagDeleted` |
| **DeleteTemplate** | `LoadTemplateForDelete` → `DeleteGoldenActor` → `DeleteGoldenTag` → `FinalizeTemplateDeleted` |
| **DeleteWorker** | `LoadWorkerForDelete` → `DrainWorkerForDelete` → `ReleaseBoundActors` → `FinalizeDeleted` |
| **ReconcileAssignments** | single step, same name |

### Workflows act for the system, not the caller

Every exported workflow method begins with:

```go
ctx = preview.ContextWithOptIn(ctx)
```

The request was already validated; what the workflow writes is the **system's own state**. Without this, a client that didn't send `Ate-Preview-Allowed: true` would have the system's own status writes rejected by preview gating — and could strand resources that cleanup paths can no longer touch. `TestWorkflowsOptInToPreview` enforces that every method does this.

### Distributed leases serialize per-resource work

`acquireLease(ctx, holder, key, subject)` returns a **context that is cancelled if the lease is lost**, so a workflow cannot keep writing after another replica has taken over. Keys:

- `lease:actor:<atespace>:<name>`
- `lease:tag:<atespace>:<name>`
- per-Worker leases in the assignment reconciler

A lost race surfaces as `apierror.Aborted("another operation is in progress for this <subject>")`.

Implementation (`store/atepg/lease.go`): a `leases` table row with a `token` (UUID) and `expires_at`; default TTL **30 s**, renewed by a background goroutine; release is a `DELETE` guarded by the token, with a 5 s timeout on a detached context so shutdown still releases. A failed release falls back to the TTL. Acquisition logs its own duration deliberately, because it happens **before any step span opens** and would otherwise be an unattributable gap in the trace. Expired-row cleanup deletes in batches of 1000 so a backlog drains over several short statements instead of one long one holding many row locks.

### Resume, in detail (`workflow_resume.go`)

```go
func (w *ActorWorkflow) ResumeActor(ctx, actorRef) (*Actor, resumed bool, err error)
```

Order of operations and the reasoning behind each:

1. **Read the actor *before* taking the lease.** The router calls `ResumeActor` on **every routed request**, even when the actor is already running. Reading first means the hot path does not upsert-and-delete a PostgreSQL lease row. If already RUNNING: return `(actor, resumed=false, nil)` immediately.
2. Take the actor lease. Re-read under it (`loadActorForResume`), which also resolves the **boot source** (`resumeSnapshotSource`, resolved once and passed **by value** to the restore step — never mutated after).
3. Check RUNNING again (it may have changed while waiting for the lease).
4. `ensureVolumesCreated` — provisions any CSI volume with no `storage_volume_id` yet, **persisting the result even when creation partially failed** so progress is not lost. An actor being deleted gets no new volumes.
5. `ensureWorkerAssigned` — see below.
6. `ensureVolumesAttached` → returns the per-volume publish contexts.
7. `ensureAteletRestored` — the atelet `Restore` RPC.
8. `finalizeRunning` — commit RUNNING.

**Metrics subtlety worth copying:** the lifecycle-op metric is recorded in a `defer` placed **before** the lease acquisition, so lease contention still counts as an attempt — but clean already-running no-ops are **skipped**, because the router resumes per routed request and recording those would sample at router QPS and bury cold-resume latency.

#### `ensureWorkerAssigned`

Two paths:

- **Actor already RESUMING** (a previous attempt assigned it): its persisted assignment is **revalidated** — worker still exists, not draining, still owned by this actor UID, still eligible for the actor's constraints. A stale assignment **crashes the actor**, because a RESUMING actor whose worker vanished/drained/was reassigned/became ineligible can never make progress on its own. A worker that is merely `UNAVAILABLE` is reported as such for the caller to retry.
- **Actor SUSPENDED**: goes through scheduling — **pinned to `status.assignedNode` when it holds a node-local snapshot** — claims the worker, and persists RESUMING with the assignment.

A version conflict in the assignment write is retried under a bounded backoff, but **only after re-reading the actor and revalidating that it can still be resumed** — the conflicting writer may have crashed, drained, or deleted it. `assignmentBackoff` reaches its cap after 5 attempts so retries end before `Steps` runs out: the loop takes roughly **0.5–1 s**.

`errWorkerFilledUp` is the retryable case where the scheduler picked a worker that the store then refused under its row lock (`BindActorToWorker`'s `admit` callback) — the next attempt re-runs scheduling. `schedulerRecordable(err)` excludes the two transparently-retried errors from the scheduler metric, so retries don't inflate the error rate or double-count the eventual success; `recordExhaustedAssignment` writes the one record for a loop that never succeeded (and names no pool, because no worker was assigned).

`validateGoldenSnapshotFidelity` rejects a golden snapshot that doesn't carry guest state. Golden actors always commit MEMORY, so this only trips on a drifted record — and it's there to surface a clear error instead of shipping a restore request atelet would reject or that would boot an empty guest.

### Suspend and Pause are one function

`suspendActor(ctx, actorRef, mode)` with `mode ∈ {suspend, pause}`. Both take the same steps; **the mode only decides the shape of the resulting snapshot** (`mode.uploadMode()` → `ateletpb.UploadMode`).

- `ensureMarkedSuspending` mints the in-progress snapshot and commits SUSPENDING. It **skips when a previous attempt of the same mode already marked the actor.**
- `preferredFidelity(atespace, tmpl)` — **golden actors always commit MEMORY** regardless of the template's setting.
- `isPausedOriginSuspend(actor)` distinguishes a suspend of an already-paused actor, which takes the `PromoteSnapshot` path (upload the node-local files) instead of `CallAteletSuspend`. Note the comment: *a local snapshot alone does not mean paused-origin* — resume restores from it too. There is a `TODO(paused-suspend)` to remove this branch once the paused→suspended transition lands.
- `ensureSuspendedFinalized` re-reads the actor first (so an out-of-band transition — e.g. the syncer crashing it — is seen), then commits SUSPENDED **with the assignment cleared in a single update** and releases the worker. **When the latest snapshot cannot be recorded it commits CRASHED instead.**
- `releaseDroppedSnapshots` / `releaseReplacedSnapshot` do the object-storage GC: an actor owns one snapshot at a time, so the previous one is deleted. **An actor that borrowed its snapshot from a tag releases nothing** — the tag owns it (`actorSnapshotOwner`).
- `ensureVolumesDetached` carries a `TODO`: it re-executes rather than checking the volumes' attach state.

Step `CallAteletSuspend` is documented as **the atelet reentrancy seam (#372)** — the point where a retried workflow may call atelet again for work atelet may already have done.

### Snapshot object-storage operations go through a plugin, always

`ActorWorkflow` holds an `objectstorev1.ControlProviderClient` and does exactly two things with it:

```go
func (w *ActorWorkflow) cleanupSnapshot(ctx, prefix resources.StoragePrefix) error  // CleanupSnapshot
func (w *ActorWorkflow) copySnapshot(ctx, src, dst resources.StoragePrefix) error   // CopySnapshot
```

`ate-api-server` itself has **no object-storage SDK linked and no storage credentials.** The plugin is a sidecar on a unix socket (`--objectstore-plugin-socket`, default `/run/objectstore-plugin/control.sock`). Errors are normalized through `pluginsocket.CallError`. Passing `nil` is allowed but leaves snapshots in place — "only tests that never reach those steps pass nil; ate-api always builds one."

## The scheduler

`cmd/ateapi/internal/scheduling/scheduling.go` — small (~300 lines) and worth reading whole.

```go
type Scheduler interface {
    Schedule(ctx, constraints) (*Worker, error)   // ErrNoCapacity if nothing eligible has room
    Applies(worker, constraints) bool              // non-capacity constraints only
    HasRoom(worker, constraints) bool              // capacity only, independent of Applies
}
```

`Applies` deliberately **excludes capacity** so that an existing assignment does not make its own Worker look ineligible.

### `Constraints`

| Field | Semantics |
|---|---|
| `SandboxClass` | the class of the snapshot being restored. The worker's **must equal** it, at **every** fidelity, **never relaxed** — a snapshot's files, volumes included, are written by one class's worker and read back by the same. Empty for an actor with no snapshot. |
| `RequiredRuntime` (`hardware.Requirement`) | restricts to workers whose `DefaultRuntime` or `RestorableRuntimes` can restore a snapshot stamped with it. **Prepared once per `Schedule`**, since `Applies` asks it of every candidate. |
| `TemplateSelector` + `ActorSelector` | **both** must match the worker's labels (the AND of template and actor selectors). |
| `RequiredNodes` | non-empty restricts to those nodes — used when the latest snapshot is node-local. |
| `Limits` | the actor's declared limits, **named the way a Worker names reported capacity, so the two subtract.** |

### The algorithm: power of two choices

Filter the fleet for eligible candidates with room → **sample two at random** → return the less-loaded one, where load is `max(actor-slot utilization, compute-resource utilization)`. Spreading across the warm pool avoids hotspots until autoscaling reclaims idle workers. `WithIntn` injects the random source for tests (`intn(n)` then `intn(n-1)`).

### The worker cache

`cmd/ateapi/internal/workercache` — an in-memory view of **all** workers, kept current via `store.Interface.WatchWorkers`, exposing `Workers()` for O(1) scheduling reads. It is the `WorkerSource`.

- `Start` does an initial `relist`, then `watchEvents`; on a lost/trimmed watch it `resync`s (relist + new watch).
- `AddHandler` lets other components (notably `WorkerAssignmentReconciler`) observe every Worker change.
- `relistInterval` forces a periodic full relist as a safety net.

Its package doc flags it as "the natural home for future indices (by node, by label, …)" — today scheduling scans the whole fleet.

## The store

`cmd/ateapi/internal/store/store.go` defines `Interface`; `store/atepg/` implements it on PostgreSQL via `pgx/v5`. `store/storecontract/` holds the shared conformance tests, `store/storetest/` the helpers, `store/dockerenv/` the **reference implementation of a fail-closed test precondition** (see [testing-and-dev.md](testing-and-dev.md)).

### Schema (`store/atepg/migrations/000001_initial.sql`)

Every resource table is the same shape: **identity columns + `version bigint` + `proto bytea`.** The proto is the source of truth; the columns exist for keys, indexes and preconditions.

```sql
atespaces(name PK, uid, version, proto)
actors(atespace→atespaces ON DELETE RESTRICT, name, uid, version, proto, PK(atespace,name))
actor_egress_policies(atespace, actor_name, ... , PK(atespace,actor_name), FK→actors ON DELETE CASCADE)
actor_templates(atespace→atespaces ON DELETE RESTRICT, name, ..., PK(atespace,name))
tags(atespace→atespaces ON DELETE RESTRICT, name, ..., PK(atespace,name))
workers(name PK, uid UNIQUE, version, proto)              -- named by pod UID
worker_assignments(actor_uid PK, worker_name→workers ON DELETE CASCADE, proto)
  + INDEX on (worker_name)
leases(key PK, token, expires_at) + INDEX(expires_at)
```

Two design choices to internalize:

- **`ON DELETE RESTRICT` on the atespace FKs** is what makes "an atespace can only be deleted once empty" a database invariant rather than application logic. Egress policies cascade with their actor; assignments cascade with their worker.
- **`worker_assignments` is keyed by `actor_uid`, not by worker.** One row per Actor, because an Actor has at most one Worker. Kept out of the `workers` table so **Worker reads, writes and watch events do not grow with occupancy** — a worker hosting 1000 actors is still one small row. The PK finds an Actor's Worker; the `worker_name` index lists a Worker's Actors.

Migrations use **goose** (`-- +goose Up`). `000002_openfga.sql` adds the embedded OpenFGA tables; `000003_access_policies.sql` the access policies. `hack/verify/postgresql-migrations.sh` gates them, and the rules are in `docs/dev/postgresql-schema-evolution.md`: **every migration must keep working for the previous `ate-api-server` binary**, and migrations are never rolled back.

### The transactional outbox — how `WatchWorkers` works

`store/atepg/outbox.go` (~22k, plus a 42k test file — read the header comment).

```sql
CREATE TABLE worker_outbox (
    xid        xid8 NOT NULL DEFAULT pg_current_xact_id(),
    created_at timestamptz NOT NULL DEFAULT clock_timestamp(),
    payload    bytea NOT NULL
) PARTITION BY RANGE (created_at);
CREATE UNLOGGED TABLE worker_outbox_default PARTITION OF worker_outbox DEFAULT
    WITH (autovacuum_enabled = off);
CREATE TABLE worker_outbox_trim (id boolean PK DEFAULT true CHECK(id), xid xid8);
```

Four mechanisms, each with a comment explaining why:

1. **Ordering (`xid`).** `writeAndAppendEvent` guarantees **exactly one row per transaction**, so xids are distinct and a polling batch never splits a transaction. Watchers poll with an **xmin-fenced xid cursor**.
2. **Retention (`created_at` range partitions).** `outboxMaintenance` pre-creates partitions and retires old ones by **dropping** them — avoiding the VACUUM I/O debt of mass deletes. A `DEFAULT` partition catches overflow.
3. **Durability (`UNLOGGED`).** Skips WAL overhead. A crash recovery just triggers watchers to rebuild from the primary `workers` table. **`worker_outbox_trim` stays LOGGED** so the high-water mark survives restarts.
4. **Loss detection.** `worker_outbox_trim` is a single-row high-water mark: the greatest xid ever discarded (with an expired partition, or on truncating DEFAULT). A watcher compares it against its cursor to detect **exactly that unconsumed rows were discarded out from under it** — and resyncs.

Payload format: **one event-type byte followed by the binary Worker proto.** The tag byte is read by other replicas during rolling deploys, so `store.WorkerEventType` values must stay **append-only stable and fit in a byte**. `unmarshalWorkerEvent` asserts invariants at the boundary: an empty payload, an unknown type byte, or a worker with no name is an **error that triggers a loud resync**, rather than falling through downstream as a silent no-op.

`!! The MUST comment in the migration:` `created_at` uses `clock_timestamp()`, not `now()`. `now()` freezes at transaction start, so a slow transaction would route its row into an already-expired partition.

### Connection pools and PostgreSQL identity

Four pools with different roles (`docs/postgres.md`):

| Pool | Role | Max conns |
|---|---|---|
| read/write (DML) | `--postgres-read-write-role` (`substrate_readwrite`) | `--postgres-pool-max-conns` |
| owner (DDL: migrations, partition maintenance) | `--postgres-owner-role` (`substrate_owner`) | 2 |
| watch | read/write | 3 |
| readiness probe | read/write | 1 |

Roles are `NOLOGIN`; `ateapi` runs `SET ROLE` after connecting, and `--postgres-schema` (default `substrate`) **overrides any `search_path` in the DSN**. Login users come from the provider (ordinary users, IAM identities, …) — Substrate neither creates them nor requires specific names.

The bundled dev database uses **no passwords.** `ate-api-server` presents two projected Kubernetes Pod Certificates from `postgres.podcert.ate.dev/identity`, one per pool, with `userAnnotations.postgres.podcert.ate.dev/username` set to `substrate_owner_user` or `substrate_readwrite_user`; the signer sets the cert CN to that login and **rejects missing annotations, unknown keys, and every other username.** The server's `hostssl ... cert` rule requires CN == requested username; `ateapi` uses `sslmode=verify-full`. The real authorization boundary is **the namespace + service account** (`--postgres-client-namespace`, default `ate-system`), because anyone who can create pods as `ate-api-server` there can request either login.

## Reconcilers

### `ActorTemplateReconciler` — builds golden snapshots

`controlapi/template_reconciler.go` (~19k code, ~36k test). Driven by a workqueue with **5 goroutines**, resynced every `--template-resync-interval` (default 20 s, minimum enforced), listing 100 templates per page.

It drives golden actors through the **in-process Control service** via a narrow `goldenActorControl` interface (`CreateAtespace`, `CreateActor`, `GetActor`, `ResumeActor`, `SuspendActor`, `DeleteActor`, `CreateTag`, `GetTag`, `DeleteTag`) — `*RPCService` satisfies it.

The sequence: create the reserved `ate-golden` atespace if needed → create a temporary golden Actor → resume it → wait for warmup → suspend it (always MEMORY) → tag the snapshot → delete the golden Actor. `status.goldenSnapshotStatus.snapshotTime` is **when the warmup ends and the snapshot may be taken**.

Warmup is `goldenSnapshotWarmup = 20s` wall clock **for templates without a wakeup probe on every container**. With probes on every container, the probe decides instead — which is why `wakeupProbe` materially shortens template-ready time.

Failure reasons recorded in `status.goldenSnapshotStatus.error_message` (format `<Reason>: <detail>`, max 4096 bytes, **nonempty is terminal — it will not be retried**): `GoldenTagConflict`, `GoldenActorInvalid`, `GoldenActorCrashed`, `GoldenActorUnexpectedState`.

### `WorkerAssignmentReconciler` — epoch-based crash recovery

`controlapi/worker_assignment_reconciler.go` + `workflow_reconcile_assignments.go`. Two goroutines, driven by `workercache.AddHandler` so it sees every Worker change.

`needsAssignmentReconcile(worker)` is simply `worker.epoch > worker.status.observed_epoch`. A raised epoch means the worker container restarted, **taking with it the sandboxes of every Actor placed in an earlier epoch** — so those Actors are crashed and their assignments released, and only then is `observed_epoch` advanced. A failure leaves `observed_epoch` where it was so the next pass redoes the release. Nothing to do — including a Worker that is gone — is success.

**Known limitation, stated in the code:** every ateapi replica runs this reconciler and queues every Worker, with only a per-Worker lease (`errWorkerLeased`) keeping them from releasing the same one at once. The TODO proposes a single elected leader, possibly by moving controller loops out of the API server into their own Deployment.

## Crash semantics

`controlapi/crash.go`. Crash messages are **fixed strings** for non-atelet causes, so they're greppable and stable:

```
"actor has no worker assignment in a state that requires one"
"node holding the actor's local snapshot is unknown"
"actor has no local snapshot"
"actor has no durable snapshot"
"assigned worker no longer exists"
"assigned worker is draining"
"assigned worker no longer hosts the actor"
"assigned worker no longer satisfies the actor's placement constraints"
"worker pod went away while hosting the actor"
"worker restarted while hosting the actor"
```

A failed atelet call instead builds `ateletCrashMessage(rpc, err)` → `"atelet <rpc>: <message>"`, because **atelet's gRPC code is `Unknown` for most failures** so the text is all there is. (There's a TODO about sanitizing it.) `ateletErrorCrashesActor` decides which atelet failures are terminal. Max 4096 bytes, matching the proto's `maxLength`.

## Actor state logging

`workflow.go` has a small discipline worth internalizing:

```go
logActorStateChanged(ctx, actor, opName)  // reads state OFF the committed record
logActorDeleted(ctx, actor, opName)       // the one state named by hand — the row is gone
```

> "The state is read off the committed record, **never passed in**, so a record cannot claim a state the store did not hold. Pass the actor the store returned and call it **after `UpdateActor` returns, not inside the mutate closure, which can be retried.** Every state commit carries a version precondition, so a call that returns is the one that made the change."

`logActorDeleted` is the single exception, because after a delete there is no committed state left to read. Crashes go through `logActorCrashed` instead — so when you consume these events, read the state from the `ate.actor.state` attribute, not from the message.

## Authentication

`docs/authentication.md`, `cmd/ateapi/internal/apiauthn/`, `--authentication-config`.

Two credential types: **mTLS client certificates** and **bearer JWTs**.

```yaml
actorIdentityJWTProvider: kubernetes
jwtProviders:
- name: kubernetes
  issuer: https://kubernetes.default.svc.cluster.local
  audiences: [api.ate-system.svc]
  certificateAuthorityFile: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  discoveryTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
- name: google
  issuer: https://accounts.google.com
  audiences: ["32555940559.apps.googleusercontent.com"]
```

- Provider names **and** issuers must be unique; `issuer` must be HTTPS; `audiences` non-empty; a token is accepted when **any** configured audience matches.
- `certificateAuthorityFile`/`discoveryTokenFile` exist for OIDC discovery against private Kubernetes API servers.
- `actorIdentityJWTProvider` names the one provider allowed to call `ActorIdentity.MintJWT`.
- **Config is read at process startup** — restart the pods after changing the ConfigMap. OIDC signing keys are cached and refreshed on an unknown `kid`.
- **Critical caveat:** a principal ID does not record which provider issued the token. *Only configure providers you trust not to issue a token whose subject is another principal's ID.*

The Google Cloud CLI currently issues identity tokens with issuer `https://accounts.google.com` and audience `32555940559.apps.googleusercontent.com` (the Cloud SDK's shared client ID) — **examples, not defaults**; verify what your IdP actually issues.

## Authorization — embedded OpenFGA

`docs/authorization.md`, `cmd/ateapi/internal/authz/{registry.go,model.fga}`.

OpenFGA is **embedded in `ate-api-server`** and stores its data in the same PostgreSQL database. A gRPC interceptor makes every check **before the handler runs**. On by default (`--experimental-enable-authz`).

**The build-breaking rule:** every served unary RPC needs an entry in `defaultRPCPermissions` (`registry.go`). With enforcement on, an RPC without one is **denied**, and **a test fails the build**. When an RPC lists several checks, all must pass.

### Principals

| Caller | ID |
|---|---|
| Kubernetes ServiceAccount token | `system:serviceaccount:<ns>:<name>` |
| Other JWT provider | the token's `sub` (for Google, the **numeric** account ID, not the email) |
| Pod certificate (mTLS) | its SPIFFE ID, e.g. `spiffe://cluster.local/ns/benchmarking/sa/default` |

A denied call names the principal with `:` **percent-encoded** (`user:system%3Aserviceaccount%3A...`) — decode before using it.

### Bootstrap owners

`--authz-bootstrap-owners`. Always hold `owner` on the global scope, whatever the policies say. They come from **configuration, not the API**; `ate-api-server` **exits at startup if authz is on and the list is empty**; restarting with a new list takes effect immediately. This is the recovery path when policies lock everyone out.

> ⚠️ **`ate-setup` makes the shared `ate-client` ServiceAccount the default bootstrap owner** so demos/benchmarks/e2e work. Anyone who can mint a token for it — **including every `kubectl ate` user who doesn't pass `--token-file`** — is then a global owner. Set real bootstrap owners for anything shared.

### Roles

Granted on the **global** scope (covers every atespace) or on **one atespace**. Roles inherit: owner ⊃ editor ⊃ viewer, and a global role holds the same role in every atespace. Global policies accept `owner` and `viewer`; atespace policies also accept `editor`.

| | Global owner | Global viewer | Atespace owner | Atespace editor | Atespace viewer |
|---|:-:|:-:|:-:|:-:|:-:|
| Create/list atespaces | ✅ | list only | | | |
| Get the atespace | ✅ | ✅ | ✅ | ✅ | ✅ |
| Delete the atespace | ✅ | | ✅ | | |
| Get access policies | ✅ | ✅ | own | own | own |
| CUD access policies | ✅ | | own | | |
| Get/list actors, templates, tags, egress policies | ✅ | ✅ | ✅ | ✅ | ✅ |
| CUD actors and templates | ✅ | | ✅ | ✅ | |
| Pause/resume/suspend/revert | ✅ | | ✅ | ✅ | |
| CUD egress policies | ✅ | | ✅ | ✅ | |
| Create/publish/delete tags; create actor from a tag | ✅ | | ✅ | ✅ | |
| Workers: get/list/list assignments | ✅ | ✅ | | | |
| Workers: create/update/delete/drain | ✅ | | | | |

### System roles — not grantable through policy

Bound to **pod certificates** via flags (`--controller-spiffe-id`, `--egress-gateway-spiffe-id`, `--ingress-router-spiffe-id`, `--api-server-sidecar-spiffe-id`, and `atelet`'s via `--atelet-service-account`), defaulting to the install's identities.

| Role | Held by | Allows |
|---|---|---|
| `controller` | `ate-controller` | create/get/list/update/delete/drain workers |
| `egress_gateway` | `atenet-egress` | get actors + their egress policies; list actors across all atespaces; **mint actor JWTs and certificates** |
| `ingress_router` | `atenet-router` | resume actors; list actors across all atespaces |
| `atelet` | `atelet` | get actors' egress policies (for the worker that enforces them) |
| `api_server_sidecar` | the ate-api-server pod identity its sidecars share (today the objectstore control plugin) | get actors, to derive the snapshot prefixes a per-actor storage credential may cover |

**Only `egress_gateway` can mint actor credentials — not even a global owner can**, since a credential lets its holder act as the actor.

### `WorkerService` bypasses the interceptor

Each handler requires `atelet`'s pod certificate directly:

| RPC | Caller must be |
|---|---|
| `RegisterWorker` | `atelet` on the worker's node |
| `RequestActorSuspend` | `atelet` on the node of the worker hosting the actor |
| `MintWorkerActorJwt` | `atelet` on the node hosting the actor |
| `MintWorkerActorCertificate` | `atelet` on the node of the worker the actor is assigned to |

`ReportActorSnapshotState` has a rule but is **not served yet**.

### With authz off

Access-policy RPCs are **still checked**, so nobody can grant themselves access while it's off and keep it once it's on. Bootstrap owners still apply, so they can set policies up beforehand.

### Managing it

```sh
kubectl ate --token-file me.token create access-policy -a team-a -f policy.yaml
kubectl ate --token-file me.token get access-policy -a team-a -o yaml > current.yaml
kubectl ate --token-file me.token update access-policy -a team-a -f current.yaml
```

`update` **replaces the whole policy** and needs the `uid` and `version` from `get -o yaml`; it fails if the policy changed, or was deleted and recreated, since. The global policy can't be deleted — update it with no bindings.

### Known authorization limitations (read before trusting it)

- **Tags can copy an actor's data.** A tag is a durable copy of memory and disk, secrets included, outliving the actor; any atespace editor can tag, publish, and clone any actor in the atespace.
- **Principals are not qualified by issuer.** With multiple trusted JWT providers, any of them that can issue a token whose `sub` matches another principal's ID acts as that principal — bootstrap owners included. System roles are unaffected (they need pod certs).
- **The atespace is the only boundary.** No per-actor or per-template grants.
- **Any atelet can read any actor's egress policy** — every node's atelet shares one identity, so the `atelet` role isn't scoped to actors on its node (issue #1564).
- No sign-in flow or credential plugin in `kubectl ate`; no install-time JWT provider config beyond the cluster's SA issuer; no groups.

## Actor identity minting

| RPC | Produces |
|---|---|
| `MintActorCertificate` | a short-lived X.509 (default **1 h**) from the actor-identity CA (`--actor-id-ca-pool`), carrying the `ActorIdentity` extension (atespace, name, UID, purpose) and an actor URI SAN. Takes a DER CSR; **only the subject public key is used.** |
| `MintActorJwt` | a JWT signed by `--actor-id-jwt-pool`, issued by `--actor-jwt-issuer`, verifiable via OIDC discovery against `ate-idp-server`. |

JWT claims: `iss`; `sub = actor/<atespace>/<name>`; `aud` (always a JSON array); `exp`/`nbf`/`iat` (**`nbf` is 5 minutes before `iat`** for clock skew); `jti`; and `actor.ate.dev` = `{atespace, actorName, actorUID}`. Lifetime 300–3600 s.

**The subject prefix distinguishes delegation:** `WorkerService.MintWorkerActorJwt` — a token minted for a Substrate *component acting on the actor's behalf* — uses **`worker-for-actor`** in both `sub` and the claim name, so a relying party can grant the two different access. `actor_jwt` is marked `[debug_redact = true]`, so `internal/protoredact` masks it in logs.

## Key flags

| Flag | Default | Purpose |
|---|---|---|
| `--grpc-listen-addr` | `:443` | the Control + WorkerService endpoint |
| `--metrics-listen-addr` / `--health-listen-addr` | `:9090` / `:9091` | Prometheus; `/healthz` + `/readyz` |
| `--grpc-server-cred-bundle` | | serving cert (service-DNS signer) |
| `--authentication-config` | | the JWT-provider YAML |
| `--postgres-{read-write,owner}-connection-string` | | the two DSNs; `@env` flag pattern in the manifest |
| `--postgres-{read-write,owner}-role`, `--postgres-schema` | …, `substrate` | `SET ROLE`; schema overrides DSN `search_path` |
| `--experimental-enable-authz`, `--authz-bootstrap-owners` | `false`*, — | *(installer passes `true`)* |
| `--{controller,egress-gateway,ingress-router,api-server-sidecar}-spiffe-id` | install defaults | system-role bindings |
| `--atelet-service-account` | install default | the SA segment of the SPIFFE ID expected on atelet's cert |
| `--actor-id-jwt-pool`, `--actor-jwt-issuer`, `--actor-id-ca-pool` | | actor identity material |
| `--pod-identity-ca-certs` | | verifies **both** inbound client certs and atelet's serving cert. **Empty disables client-cert verification and makes atelet dials fail.** |
| `--atelet-client-cred-bundle` | | the client cert presented when dialing atelet |
| `--objectstore-plugin-socket` | `/run/objectstore-plugin/control.sock` | |
| `--template-resync-interval` | `20s` | golden-snapshot reconciler |
| `--drain-delay` / `--drain-timeout` | `13s` / `15s` | SIGTERM handling |
| `--default-egress-gateway-address` | | handed down to actors |
| `--preview` | | gates |

## Where to read more

- `cmd/ateapi/internal/controlapi/workflow.go` — read this first; the `stepSpan` doc comment is the design brief for the whole engine.
- `cmd/ateapi/internal/store/atepg/outbox.go` header comment — the outbox in four paragraphs.
- `docs/authorization.md`, `docs/authentication.md` — complete and current.
- `docs/postgres.md`, `docs/dev/postgresql-schema-evolution.md`, `docs/dev/cloud-sql-scaling-guide.md`, `tools/setup-gcp/cloud-sql.md`.
- `cmd/ateapi/internal/controlapi/functionaltest/` — functional tests that run the service against a real store; the fastest way to see an end-to-end control-plane path without a cluster.

# Observability: Logs, Metrics, Traces

Substrate's observability has a shape, not just a set of signals, and the shape follows from one constraint: **actor identity is barred from metric labels.** Everything else — the registry, the split between metrics and log events, the phase records — is downstream of that.

Authorities: `docs/observability.md` (519 lines), `docs/metrics/registry/` (an OTel Weaver registry), `docs/metrics/substrate.yaml`, and `docs/dev/best-practices/{metrics,logging,tracing,otel-collector}.md`.

## The model in one paragraph

An actor has no stable host, so none of the three signals alone can answer a question about it. **Metrics** aggregate to a bounded label set a TSDB can hold (template, pool, sandbox class, outcome). **Logs** carry actor identity, where cardinality is free, and are the only place you can attribute anything to one agent. **Traces** stitch a single activation across the router, the control plane, atelet and the worker — but the data plane is **head-sampled at 1%**, so they cannot answer distribution questions. Pick the signal by what you're asking, not by habit.

```
"what is p99 resume latency for this template?"      → metric (ate.actor.lifecycle.operation.duration)
"what is p99 for THIS actor?"                        → log (Restore timing breakdown) — the only one that can
"why was this one slow?"                             → trace, if it was sampled; the phase log otherwise
"what state is actor X in right now?"                → ask the control plane. NOT the logs.
"how did it get there, and when?"                    → log (Actor state changed)
"how many activations?"                              → the router's access log
```

## `internal/ateattr` is the single source of truth

> *"Package ateattr is the single source of truth for substrate's `ate.*` telemetry attributes: the identity keys stamped on spans/logs, and the bounded value sets used as metric labels. Centralizing them keeps a key (and value) meaning the same thing across every signal and binary."*

Do not invent an attribute key anywhere else. `ateattr.ActorLogAttrs` and `ateattr.ActorLogLabels` deliberately return **the same five keys**, and **a test holds them together** — so filtering on `ate.actor.uid` finds a component's record about an actor and the actor's own output alike.

The five identity keys:

```
ate.atespace  ate.actor.name  ate.actor.uid  ate.template.atespace  ate.template.name
```

On an actor-attributed **component** record they sit **flat at the top level**, not inside a label group, *"so a collector lifts the keys straight onto the log record's attributes"* — a component writes no envelope.

---

# 1. Logs

## The records that matter

| Record | Written by | When |
|---|---|---|
| `Actor state changed` | ateapi | **once per committed** state transition |
| `Actor crashed` | ateapi | once per committed transition into `CRASHED` |
| `Actor usage sampled` | both ateworkers | per actor per sampling interval, plus `initial` and `final` |
| `Restore timing breakdown` | atelet | once per restore, success **or** failure |
| `Checkpoint timing breakdown` | atelet | once per checkpoint, success **or** failure |
| `ateworker.actor.{restore,checkpoint}.duration.*` | both ateworkers | the same two messages, from inside the `worker_*` phases |
| `Sandbox has no processes left while the actor is hosted` | ateworker-gvisor | WARN, on the usage sweep |

### `Actor state changed` — the authoritative state stream

```json
{"time":"…","level":"INFO","msg":"Actor state changed",
 "ate.atespace":"ate-demo-counter","ate.actor.name":"counter-1","ate.actor.uid":"8f2a…",
 "ate.template.atespace":"ate-demo-counter","ate.template.name":"counter",
 "ate.actor.operation.name":"suspend","ate.actor.state":"suspended",
 "trace_id":"4bf92f…","span_id":"00f067…","trace_flags":"01"}
```

The guarantees, each with a mechanism behind it:

- `ate.actor.state` is the `ateapipb.ActorState` value **lowercased**, *"so the log vocabulary and the state machine cannot fork."*
- **The record goes out after the store commit, never before, and the state is read straight off the committed record rather than named by the caller.** Every state commit also has a version check, so if two writers race the loser writes nothing. **"You will never see a state here that the store did not actually hold."**
- `ate.actor.operation.name` says which operation drove the transition, *which the state alone does not*: an actor reaches `suspended` both when created and when suspended. **Pause and Suspend are one operation, both labeled `suspend`.**
- **Creating an actor counts as a change.** A new actor is born suspended and gets a record saying so with `operation.name: create` — otherwise an actor created and never resumed would have **no record at all**, however long you keep logs.
- **`deleted` is the only state with no `ActorState` behind it.** Written once the row is gone, so there's nothing to read back. It is the last record an actor ever gets. *"Without it, `deleting` would be the end of the story, and a delete that finished would look just like one that got stuck."*
- A consumer deriving state **selects on `ate.actor.state` under ateapi's resource (`service.name=ateapi`)**, not on the message. **The resource is what makes the record authoritative.**

**What this stream will not tell you** — quoted, because it is the most useful caveat in the whole doc:

> *"It only writes when something changes. So an actor that has been sitting in the same state since before your logs roll over has no record, and no state. **Ask the control plane what state something is in right now.** Use this stream to see how it got there and when. Records can also go missing, like any other log, and a gap looks the same as an actor that just sat still. If you want to count activations, use the router's access log instead."*

### `Actor crashed`

Written beside the `ate.actor.crashes` increment, **under the same already-crashed guard**, *"so the two can never disagree about how many crashes happened."* Severity ERROR / 17.

**The counter carries no actor identity, so this record is the only way to attribute a crash to one agent.** The decision-point line that precedes it (`Setting Actor to crashed due to error`) carries only `ate.atespace` and `ate.actor.name` — it is written before the Actor is loaded, so no uid exists yet.

### The phase breakdown records

```json
{"time":"…","level":"INFO","msg":"Restore timing breakdown",
 "ate.atespace":"ate-demo-counter","ate.actor.name":"counter-1","ate.actor.uid":"8f2a…",
 "ate.template.atespace":"ate-demo-counter","ate.template.name":"counter",
 "ate.snapshot.fidelity":"memory","ate.snapshot.kind":"latest","ate.sandbox.class":"gvisor",
 "ate.actor.restore.duration.download":0.310,
 "ate.actor.restore.duration.oci_unpack":0.050,
 "ate.actor.restore.duration.worker_restore":0.060,
 "ate.actor.restore.duration.total":0.420,
 "trace_id":"4bf92f…","span_id":"00f067…","trace_flags":"01"}
```

- **These are unsampled.** One per operation, succeeded or failed; a failed one also carries `error.type` (the gRPC code; `DeadlineExceeded`/`Canceled` for context errors), *"so a reader can leave it out of a latency distribution."*
- The duration keys are **the instrument's name with an `ate.snapshot.phase` value appended**, and hold **seconds**, matching the instrument's declared unit.
- **A phase that never ran is absent rather than zero**, and **phases overlap and do not sum to the total** (the download runs concurrently with the asset fetch and OCI unpack). Use `total` as the denominator.
- There is **no `ate.snapshot.phase` key on the record** — on a histogram datapoint it names the one step timed; this record carries them all.

> **"This is the record to use for a per-actor wake-up distribution. The histogram cannot answer that question at all, because actor identity is barred from metric labels; traces can, but the data plane is head-sampled at 1%."**

The ateworkers write records with **the same two messages** from inside the `worker_restore`/`worker_checkpoint` phases, under `ateworker.actor.{restore,checkpoint}.duration.<phase>`. **They are not instruments.** The same identity keys make the two layers joinable per actor. Phases are each runtime's own — `cmd/ateworker-gvisor/phaselog.go`, `cmd/ateworker-microvm/phaselog.go`, rendered by `internal/ateworker/phaselog`.

### `Actor usage sampled`

One per actor per interval, plus an `initial` record per activation and a `final` one at teardown, told apart by `ate.stats.kind`.

```json
{"time":"…","level":"INFO","msg":"Actor usage sampled",
 "ate.atespace":"ate-demo-counter","ate.actor.name":"counter-1","ate.actor.uid":"8f2a…",
 "ate.template.atespace":"ate-demo-counter","ate.template.name":"counter",
 "ate.workerpool.namespace":"ate-demo-counter","ate.workerpool.name":"counter-pool",
 "ate.sandbox.class":"gvisor","ate.stats.source":"cgroup","ate.stats.kind":"periodic",
 "ate.actor.epoch":1788903185000000000,
 "ate.stats.memory.usage":39845888,"ate.stats.memory.working_set":31457280,
 "ate.stats.cpu.time":0.196776,"ate.stats.memory.peak":52428800}
```

The semantics, which are easy to get wrong:

- Measurements are **named after the `ate.actor.stats.*` instruments and share their units** — `ate.stats.cpu.time` is **seconds**.
- They are **absent, not zero**, while the actor is not measurable; the record says so with `ate.stats.source` unspecified.
- **`ate.stats.cpu.time` restarts at zero with each `ate.actor.epoch`** (the unix-nano time the activation began), **so a lifetime figure is the sum over epochs of each epoch's highest value.** `memory.usage` and `memory.working_set` are absolute; `memory.peak` is as the source reports it.
- Order: `initial` (when an activation becomes measurable) → `periodic` (per interval) → `final` (at teardown: a checkpoint, a terminate, a graceful shutdown, or a re-host on micro-VM). The sampler reads an actor only between its initial and final records; until then the discovery read reports it **pending**. **A failed first reading writes no `initial` record, and periodic records start anyway, so an unreachable guest is still reported.**
- **An activation that ends any other way — a crash — has no `final` record, so take the highest CPU per epoch rather than wait for one.** The final reading is taken just before teardown, bounded to 10 ms on micro-VM; on failure or timeout the `final` record repeats the newest measured sample, dated when *that* was read, or is pending if none was.
- Cadence: `--usage-sample-interval` (default `1m`, **minimum `50s`**). `GetActiveWorkloadStats` serves the latest sample rather than reading again.
- Trace context: `initial` records carry the Run/Restore call's span (on micro-VM from a goroutine that call starts, so the record can land just before or after the call returns); `final` records carry the tearing-down call's span. **`periodic` records and a shutdown's `final` records serve no request and carry none.**
- **Delivery is best-effort**: the stdout record **ignores `--log-level`** and sits behind a bounded queue, so a stalled log consumer costs records, with a warning, **never the sampler.**

> ⚠️ **"Do not put actor identity on a log-based metric."** Aggregating these in the log store is what they're for, but a log-based metric over them must label **only** by the bounded set (template, sandbox class, source, pool). Promoting `ate.actor.uid` or `ate.actor.name` into a metric label **reintroduces exactly the per-actor cardinality this split keeps out of the TSDB.**

### `Sandbox has no processes left while the actor is hosted`

WARN from ateworker-gvisor when a running actor's gVisor sandbox has died — the sentry crashed, was killed, or was OOM-killed. Found by the usage sweep, so it appears within one sweep, once per activation. `ate.sandbox.oom_kills` is the cgroup's `oom_kill` count, so **non-zero points at the kernel OOM killer.**

Caveats: **the actor still reports `RUNNING` after this record; nothing moves it to `CRASHED` yet** (#2211). It does not fire when an application container exits while the sentry is up, nor during a checkpoint or terminate. The count covers the life of the **cgroup leaf**, so a leaf left by an earlier activation of the same actor **carries its count forward.**

## OTLP log events vs stdout

`OTEL_LOGS_EXPORTER`, a comma-separated list (names trimmed, lowercased, counted once; an unknown or empty item warns and is skipped while the rest apply; `none` with an exporter, or a value naming nothing known, warns and keeps the default `none`):

| Value | Effect |
|---|---|
unset / `none` | records on **stdout**. What every environment but kind uses today. |
| `otlp` | records go out as **OTLP log events** — a collector reads them without knowing Substrate's stdout envelope. **They are not in `kubectl logs`.** |
| `otlp,console` | **both**, on purpose. What kind does for debugging. |

`console` here means **the component's own JSON log line**, not the OTel SDK's console exporter format.

`atecontroller` copies `OTEL_LOGS_EXPORTER` into each worker pod, so the kind ConfigMap sets the ateworkers to `otlp,console` too.

Three `event.name` values — which is the OTLP LogRecord's **own field**, not an attribute:

| `event.name` | Body | Severity | Attributes |
|---|---|---|---|
| `ate.actor.state_changed` | `Actor state changed` | 9 | the five identity keys + `ate.actor.operation.name` + `ate.actor.state` |
| `ate.actor.crashed` | `Actor crashed` | 17 | the same |
| `ate.actor.usage_sampled` | `Actor usage sampled` | 9 | the five + `ate.workerpool.*` + `ate.sandbox.class` + `ate.stats.*` + `ate.actor.epoch` |

> *"A crash is its own name because an event name promises a set of attributes and a crash has a different severity and shape. **There is no name per state**: `ate.actor.state` already says which transition happened, so a consumer still selects on that one attribute and needs no map from a name to a state."*

All three are declared in `docs/metrics/registry/events.yaml`, which `hack/verify-all.sh` checks. The instrumentation scope is `go.ate.dev/substrate/internal/actorevent` for every actor event. **Trace context is not an attribute** — it goes on the record's own `TraceId`/`SpanId` fields.

### Three rules for consuming the event stream

1. **Never sample or filter the lifecycle stream.** *"A consumer takes the last event for an actor's uid, so one dropped record reports a stale state with no sign that anything is missing. This is the one stream where a sampling policy is a **correctness bug** rather than a cost trade."* The usage samples may be sampled.
2. **Each record goes to one place unless asked otherwise** — so a backend that also collects pod stdout (as GKE's logging agent does) doesn't store it twice. With `otlp,console` on such a cluster each record is stored twice and **the collector should drop one copy by `event.name`.**
3. **With `otlp` alone, a lifecycle record can be lost.** The OTLP processor batches and **drops records with no signal** when its queue is full, when the collector stays unreachable after retries, or when the process exits ungracefully. Records written after the logger provider shuts down go to stdout — and **that shutdown runs last, after the server drains, so records written during the drain are still queued and are lost if the final export fails.** Where that is unacceptable, use `otlp,console`.

## Secrets in component logs

Control-plane and node component logs **mask proto fields marked `debug_redact`** (actor JWTs, env var values): a logged request/response body shows `[REDACTED]`. **Not every process masks them** — `docs/dev/best-practices/logging.md#logging-protos-that-hold-secrets` lists the ones that do not. **An actor's own container output is not touched.**

## `kubectl ate logs`

`docs/observability.md#active-actor-inspection-via-cli` — inspect a running actor's output, with `--follow`/`-f` and filtering by container or source. A suspended actor has no live output.

## Actor-aware worker-pod logs

`internal/actorlog` forwards an actor container's stdout/stderr to the worker pod's stdout, **annotated with the `ate.*` identity labels**, and emits synthetic lifecycle events. So `kubectl logs <worker-pod>` shows the actor's own output, identity-tagged — the reason a failed e2e run deliberately keeps its namespaces.

---

# 2. Metrics

**20 instruments.** `docs/metrics/registry/metrics.yaml` is the only source of truth for labels, buckets and permitted values.

| Instrument | By | Type | Measures |
|---|---|---|---|
| `rpc.server.call.duration` | every gRPC server, via `otelgrpc` | histogram | per-method latency/rate/errors (`rpc.method`, `rpc.response.status_code`) |
| `rpc.client.call.duration` | gRPC clients | histogram | the client side |
| `ate.actor.crashes` | ateapi | counter | transitions into `CRASHED` |
| `ate.actor.lifecycle.operation.duration` | ateapi | histogram | per-operation latency + failure (`error.type` present = failure) |
| `ate.scheduler.assignment.duration` | ateapi | histogram | **one attempt** to assign a worker; outcome `assigned`/`no_capacity`/`error` |
| `ate.workerpool.workers` | ateapi | up/down counter | live workers per pool, by state `idle`/`partial`/`at_capacity`/`unschedulable` and class |
| `ate.workerpool.desired_workers` / `.ready_workers` | atecontroller | up/down counter | from `spec.replicas` / `status.readyReplicas` |
| `ate.actor.restore.duration` / `.checkpoint.duration` | atelet | histogram | **per-phase** node-side latency |
| `atelet.snapshot.size` | atelet | histogram | uncompressed **allocated** bytes per snapshot image (`st_blocks * 512`, **excluding sparse holes** in micro-VM `memory-ranges`) |
| `ate.actor.stats.{cpu.time,memory.usage,memory.working_set,sampled_actors}` | ateworkers | — | aggregate usage at the bounded label set |
| `ate.imagecache.requests` | atelet | counter | lookups by `ate.imagecache.outcome` |
| `atenet.router.route.duration` | atenet-router | histogram | **Envoy receiving a request → Envoy forwarding it to the resolved worker**, excluding actor compute and the response |
| `atenet.router.parking.{active,wait.duration,rejected}` | atenet-router | — | the parking lot |

## The label rules, which are the actual design

- **`ate.workerpool.namespace` + `ate.workerpool.name` always travel together**, on every instrument that names a pool. A WorkerPool is namespaced, so **the name alone merges same-named pools from different namespaces into one series.** The pair is what joins capacity (`workers`, `desired_workers`, `ready_workers`) to demand (`assignment.duration`, `lifecycle.operation.duration`, `crashes`) **by pool**.
- **"No keys" means the operation has no pool** — a crash before the actor reached a worker, or the `no_capacity` outcome. **No instrument sends the pair with both keys empty.**
- The three snapshot labels are **orthogonal** and mean the same on every histogram carrying them:
  - `ate.snapshot.kind` — `local` (node-local, from a pause), `latest` (the actor's own durable), `golden` (the template's), `boot` (from scratch; **never appears on the atelet histograms**)
  - `ate.snapshot.fidelity` — `volumes` / `rootfs` / `memory`
  - `ate.snapshot.phase` — restore: `volume_mount`, `manifest_fetch`, `sandbox_assets`, `download`, `oci_unpack`, `worker_restore`; checkpoint: `sandbox_assets`, `worker_checkpoint`, `persist`, `cleanup`; `total` on both
- **Phases overlap and do not sum to `total`.** Each is an independent observation; use `total` as the denominator. **A phase that never started is absent rather than zero.**
- `ate.sandbox.class` is derived from the runtime on the assignment or snapshot (each pool has exactly one class), **so it adds no extra series next to the pool label** — it exists so dashboards can aggregate by class without enumerating pool names.
- **The label set varies per operation**: resume carries the most dimensions, delete only the operation and error type.
- **"Already-running resume no-ops are not recorded, so the histogram tracks actual activations, not router traffic."**
- **High-cardinality actor identity stays off metrics entirely.**

### Two traps worth memorizing

**Do not compare the router's and the scheduler's `no_capacity` counts.** *"The scheduler records one sample on `ate.scheduler.assignment.duration` for each resume. The router records one sample for each **request**. Many requests can join one resume, and one parked request can cause more than one resume."*

**The image-cache hit ratio is `hit / (hit + miss)`, with failures and abandoned lookups out of the denominator.** `hit` means the node holds a *complete* record — every layer directory it names is present. `error` is a failed lookup whatever the cause; `cancelled`/`timeout` is the caller giving up. `error.type` appears **only** on `error` and carries the registry's HTTP status from a fixed allow-list (`401 403 404 429 500 502 503 504`) — *"because the registry client reports whatever the remote returned"* — with everything else `_OTHER`.

### `atenet.router.route.duration` labels

- `ate.router.outcome`: `ok`, `cancelled`, `timeout`, `no_capacity`, `failed_precondition`, `lock_conflict`, `not_found`, `unavailable`, `rate_limited`, `resume_error`.
- `ate.router.resume`: the **singleflight** state — `none` (actor already running), `triggered` (this request completed a cold activation), `joined` (waited on another request's resume), `unknown` (the resume did not complete, so whether an activation ran is unknown).
- `ate.template.atespace`/`name` hold the literal `unknown` when the router has no template to name.

### Why `desired_workers`/`ready_workers` are separate instruments

- **Supply-side saturation golden signal** — measures whether commanded capacity was delivered. Dedicated instruments match Kubernetes semantic conventions (`k8s.deployment.desired_pods`/`available_pods`) **because desired + ready is not a disjoint sum.**
- **Autoscaling anti-windup** — `desired - ready > 0` sustained beyond a few minutes means undelivered capacity (node-pool exhaustion, quota, stuck pods).

## The registry

`docs/metrics/registry/` is an **OpenTelemetry Weaver** registry. Weaver reads the whole directory, resolves each `ref`, and refuses an incorrect group or attribute — **so a new file in it is checked straight away.**

```sh
hack/verify/metrics.sh                          # what CI runs (uses a local weaver, else the image)
weaver registry check -r docs/metrics/registry  # the same, direct
```

| File | Content | Read by |
|---|---|---|
| `registry/manifest.yaml` | the registry name + schema URL | Weaver |
| `registry/metrics.yaml` | the `groups`: every metric, every attribute, every permitted value | Weaver |
| `registry/events.yaml` | the OTLP log events | Weaver |
| **`docs/metrics/substrate.yaml`** | **the rules Weaver cannot express** | a person, an agent |

`substrate.yaml` sits **beside** the registry, not in it, because *"Weaver permits only `groups` and `imports` at the top level of a registry file, and it refuses a file that has any other top-level key."* It records:

- **`upstream_semconv_version`** — the upstream semconv version Substrate borrows `error.type`, `file.name` and the `rpc.*` attributes from.
- **`bridged_metric_families`** — what atecontroller exports but Substrate does not define, **as prefixes** (see below).
- **`cardinality_rules`** — **this is where the bar on actor identity lives.** No rule is enforced today; each rule says what *could* enforce it.
- **`lint_exceptions`** — known debt.
- **`blind_spots`** — **"the subsystems with no metrics. Read this list before you give a cause to a fault."** Notably: **the store has no instruments, but every lifecycle operation uses it.**

### What the check does NOT do

> *"Weaver reads the registry, and not the Go code. Thus the build stays correct if a person adds an instrument to the code and not to the registry, or renames one in the code only."*

`weaver registry generate` could produce `internal/ateattr` from the registry, and `weaver registry live-check` could compare the registry against e2e telemetry. **Both are future work.** Until then, the registry is a convention held by review, not by CI — so update it in the same change, per `AGENTS.md` and `docs/dev/best-practices/metrics.md`.

## Bridged controller-runtime metrics

atecontroller bridges controller-runtime's **private Prometheus registry** onto its OTLP reader, so `controller_runtime_*`, `workqueue_*`, `certwatcher_*`, `rest_client_*`, `leader_election_*`, `go_*` and `process_*` reach the collector too — **keeping their Prometheus names, because they are upstream instruments and renaming them would break existing controller-runtime dashboards.**

Use them to ask whether the controller is keeping up: rising `workqueue_depth` or `workqueue_queue_duration_seconds` means reconciles are falling behind; `controller_runtime_reconcile_errors_total` says which controller.

They are recorded in `substrate.yaml` **as prefixes, not one metric at a time**: *"The upstream library owns the names, the labels and the buckets, and a version bump can add a family. A copy in the registry becomes wrong with no signal."* `controller_runtime_version` dates the list so a bump has an obvious place to check.

Two gotchas:
- controller-runtime enables **native histograms** on `controller_runtime_reconcile_time_seconds`, `workqueue_queue_duration_seconds` and `workqueue_work_duration_seconds`, so those three arrive as **OTLP exponential histograms**, not fixed-bucket ones.
- **A queue that has never processed an item bridges as an exponential histogram with no positive buckets, which the Cloud Monitoring OTLP endpoint rejects on ingest.** atecontroller gives each such datapoint one positive bucket with count 0 before the push (`cmd/atecontroller/metrics.go`), *"so an idle queue no longer causes an error every tick. The padding carries no observations, so other backends are unaffected."*

## Push vs scrape

`OTEL_METRICS_EXPORTER`, same list rules as `OTEL_LOGS_EXPORTER`. Default: `otlp,prometheus` — or **`otlp` alone on ateworker**, which serves no `/metrics` (there `prometheus` is skipped as unknown with a warning, and the push stays on unless the value is `none`).

ateapi, atelet, atenet-router and the credential provider **also** serve every instrument on Prometheus `/metrics`. A scraping cluster sets `OTEL_METRICS_EXPORTER=prometheus` so each series reaches the backend once — the components then install **no** OTLP metric reader. Traces and logs are unaffected.

> **`/metrics` stays on for every value today, including `otlp` and `none`. That is temporary until the probes move off `:9090`** (#2067).

atecontroller's `:8080` listener follows the variable:

| Value | Push | `:8080` |
|---|---|---|
| unset / `otlp,prometheus` | yes | serves controller-runtime families + `ate.workerpool.*` |
| `otlp` | yes | **closed** |
| `prometheus` | no | serves them |
| `none` | no | closed |

## Local metrics on kind

Prometheus is auto-provisioned in `otel-system`:

```sh
kubectl port-forward -n otel-system svc/prometheus 9090:9090
# http://localhost:9090 — run `up` (one series per target, value 1),
# then explore rpc_* via autocomplete. Status > Targets lists the discovered pods.
```

**Storage is `emptyDir`, so metrics are lost when the Prometheus pod restarts.**

---

# 3. Tracing

Spans cover the gateway, the router, the control plane, atelet and the worker pods.

**The workflow engine is the thing to know:** every step opens `step.<Name>` on the `controlapi` tracer, and a step whose postcondition already held is annotated `step.skipped=true` + `step.skip_reason` — **so a trace of a re-entered workflow shows exactly which steps fast-forwarded and where real work restarted.** An error is recorded on the span and wrapped `workflow failed at step <Name>: …`. This is the primary tool for a stuck lifecycle operation.

Distributed-lease acquisition **logs its own duration** (`PostgreSQL lease acquisition finished`), because *"acquisition runs before any workflow step span opens"* and would otherwise be an unattributable gap in the trace. Lease `Close` logs its two waits for the same reason.

**The one trace-propagation gotcha**, in the ingress handler:

> *"The dataplane doesn't propagate trace context into the ext_proc gRPC stream's metadata — the per-request traceparent arrives in the HTTP headers carried inside the `ProcessingRequest` payload. Extract from there so our span links to the gateway's ingress span."*

Get that wrong and every router span is an orphan.

**Sampling:** the data plane is **head-sampled at 1%**, which is why the phase-breakdown logs exist. `kubectl ate --trace` turns on **on-demand** tracing: the CLI generates a trace ID and signals the server to trace that request. `internal/contextlogging` ties log records to the active span, which is how `trace_id`/`span_id` reach every record.

### On kind

```sh
kubectl port-forward -n otel-system svc/jaeger 16686:16686
kubectl ate get actor my-counter-1 -a ate-demo-counter --trace
# then search http://localhost:16686 for the most recent trace
```

The kind overlay provisions the collector and a Jaeger all-in-one in `otel-system`; no extra setup.

### On GKE

Needs the **Cloud Trace API** enabled and the cluster on **Managed OpenTelemetry**. Clusters from `setup-gcp create cluster`/`bootstrap` always enable it; otherwise:

```sh
gcloud services enable cloudtrace.googleapis.com --project=PROJECT_ID
gcloud beta container clusters update CLUSTER_NAME --project=PROJECT_ID \
  --managed-otel-scope=COLLECTION_AND_INSTRUMENTATION_COMPONENTS --location=LOCATION
```

---

# 4. The transport

## Worker pods have no network path to the collector

`internal/otlprelay`: atelet serves OTLP on `atelet-otlp.sock` (a unix socket under the shared hostPath) and forwards to `OTEL_EXPORTER_OTLP_ENDPOINT`. **Node-scoped, not per-pod**, so one socket collapses N per-pod collector connections into one per-node connection.

The relay **carries logs, traces and metrics**, and **admits only ateworker resources** — which has a security consequence the doc spells out: *"once worker pods have no direct path to the collector, a record carrying these keys under any other resource cannot come from a worker pod. Until then this is a rule the consumer applies, not one the transport enforces."*

`--otlp-relay-socket` empty, or absent at startup, makes a worker export directly to `OTEL_EXPORTER_OTLP_ENDPOINT` instead.

## Actor telemetry needs an egress policy

**An OpenTelemetry SDK inside an actor cannot reach the collector until the actor has an `http` egress rule for it.** Default deny applies to the actor's own telemetry exactly as to anything else. This surprises everyone once.

## `ate-otel-config.yaml`

One ConfigMap, consumed by every control-plane component through `envFrom`, carrying the shared OTLP exporter settings. **The kind overlay replaces it with a same-named ConfigMap** so `envFrom` resolves to the in-cluster collector instead of the GKE one — which is also why it is listed in the bundle rather than in the per-component overlays (listing it in both would be a duplicate resource).

---

## Adding or changing an instrument — the checklist

1. Read `docs/dev/best-practices/metrics.md` — it covers instrument choice, labels, the Go, the test, and the registry entry.
2. **Use `internal/ateattr`** for every attribute key and bounded value. Never define one locally.
3. **No actor identity on a metric label.** If you need per-actor data, it belongs on a log record.
4. If the attribute may name a WorkerPool, carry **both** `ate.workerpool.namespace` and `ate.workerpool.name`, or neither.
5. Update `docs/metrics/registry/metrics.yaml` (or `events.yaml`), and `docs/metrics/substrate.yaml` if it touches cardinality rules, bridged families, exceptions or blind spots.
6. `hack/verify/metrics.sh`, then `hack/verify-all.sh`.
7. Remember the check does **not** read Go — the code↔registry link is held by review.

## Where to read more

- `docs/observability.md` — the authority; every section is worth reading once.
- `docs/dev/best-practices/metrics.md` (519 lines) — how to add an instrument properly.
- `docs/dev/best-practices/logging.md` — including how to log protos that hold secrets.
- `docs/dev/best-practices/tracing.md` (271 lines), `docs/dev/best-practices/otel-collector.md` (645 lines).
- `docs/metrics/substrate.yaml` — **read `blind_spots` before attributing a fault to a component.**
- `docs/dev/suspend-resume-phase-breakdown.md` — the runbook that turns these records into a report.
- `benchmarking/analysis/README.md` + `benchmarking/analysis/phase_report.py` — phase percentiles and waterfalls, joining the atelet and ateworker layers.
- `benchmarking/observability.md`, `benchmarking/monitoring.yaml`.

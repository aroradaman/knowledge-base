# Install, Configuration, and Operations

`cmd/ate-setup` is the installer (~26k lines of Go). `hack/install-ate.sh` is a **shim** over it, kept so existing command lines and CI jobs keep working; **it holds no install logic of its own.**

Authorities: `cmd/ate-setup/README.md` (+ `commands.md`, `differences.md`, `cli-diff.md`), `docs/operator-install.md`, `docs/upgrade.md`, `docs/postgres.md`, `manifests/ate-install/`.

## Two install modes

```sh
# Build everything from the checkout with ko (the default — what a developer wants)
ate-setup deploy ate-system --credential-provider='{"name":"k8s.io"}'

# Install published images: nothing is built, no registry is pushed to
ate-setup deploy ate-system \
  --credential-provider='{"name":"k8s.io"}' \
  --image-repo registry.example.com/substrate \
  --image-tag v0.0.0
```

With `--image-repo`/`--image-tag`, **the manifests still come from the checkout** and **each reference is pinned to the digest its tag names** — so the registry has to be readable **from here as well as from the cluster.** Both halves are required together: *"a tag with no repository would be dropped, leaving a build from source that looks like the release the tag names."*

```sh
# Publish a release: every image a pre-built install needs, all tagged TAG,
# including the docker-built envoy-dataplane
VERSION=TAG go run ./cmd/ate-setup publish release-images --ko-docker-repo REPO

# Just the two worker images (used by the upgrade runbook)
go run ./cmd/ate-setup publish worker-images --config new.yaml
```

## `--credential-provider` is required

An install must choose:

| Value | Effect |
|---|---|
| `{"name":"k8s.io"}` | deploys the bundled Kubernetes Secrets provider, with a NetworkPolicy admitting **only the egress gateway** |
| `{"enabled":false}` | egress credential injection **off** |

> ⚠️ **The bundled provider starts with an empty, default-deny namespace policy, so it resolves no Secrets until you grant each atespace the namespaces whose Secrets it may read.** See `docs/egress-credential-injection.md#enable-it`. This is the single most common "credential injection isn't working" cause.

`ATE_CREDENTIAL_PROVIDER` is the same choice.

---

## Configuration: three channels, one precedence

Every setting can come from a **flag**, an **environment variable**, or a **configuration document**.

```yaml
apiVersion: install.ate.dev/v1alpha1
kind: SubstrateInstall

namespace: ate-system
context: my-cluster
clusterSize: size0            # size10 assumes a dedicated PostgreSQL node
rolloutTimeout: 1m0s
atenet:
  dataplane: envoy            # envoy | agentgateway
images:
  repo: ghcr.io/agent-substrate/substrate
  tag: v0.4.1
storage:
  bucketName: ate-snapshots
ateapi:
  postgres:
    schema: public
```

```sh
ate-setup deploy ate-system --config substrate-install.yaml
ATE_CONFIG=substrate-install.yaml hack/install-ate.sh --deploy-ate-system
```

`ATE_CONFIG` is how the document reaches the shell installer, *"which forwards only a fixed set of flags and otherwise passes settings by inheriting the environment."*

A worked example: `docs/examples/substrate-install.yaml`.

### Precedence, and why

```
default  <  environment  <  configuration file  <  flag
```

> *"The file outranks the environment **deliberately**: the file is the artifact an operator wrote and can review, and ambient state left in a shell must not silently outrank it. A flag outranks both because it was typed for this run."*

A channel that does not mention a setting leaves the one below in place, **so a document need only name what differs from the defaults.**

### An unknown key is rejected, not ignored

```
Error: install.yaml: unknown setting "ateapi.postgres.schemas"
```

> *"A typo would otherwise leave the install running on the default with nothing on screen to say so."*

### Empty is a value

```yaml
namespace: ""
```
```sh
export ATE_NAMESPACE=      # the same thing
ate-setup deploy ate-system --namespace=
```

**All three clear the setting rather than falling through to the default. That is how a setting is removed from an installation: write its zero value and re-apply.** Only an *absent* setting falls through:

| Shell | Meaning |
|---|---|
| `unset ATE_NAMESPACE` | not set; the layer below applies |
| `export ATE_NAMESPACE=` | **set to empty** |
| `export ATE_NAMESPACE=ate-prod` | set to `ate-prod` |

> *"A script that exports an empty variable to mean 'unset' is configuring the setting."*

For a non-string setting, an empty value **is not a value of its kind and is rejected, naming the variable** — e.g. `ATE_INSTALL_PODCERT_WORKERS_PER_SIGNER=`.

### Every run reports what it resolved

```
configuration: 6 set, 35 default
  ateapi.expectedJWTIssuer                    ""            (from default)
  ateapi.postgres.readWrite.connectionString  <set>         (from ATE_API_POSTGRES_READ_WRITE_CONNECTION_STRING)
  atenet.dataplane                            agentgateway  (from ATE_ATENET_DATAPLANE)
  benchmark.sandboxClass                      microvm       (from --sandbox-class)
  benchmark.workerCount                       7             (from config.benchmark.workerCount)
  clusterSize                                 size0         (from default)
  context                                     kind-kind     (from the --kind profile)
  namespace                                   ate-prod      (from config.namespace)
```

> *"An install that succeeds with the wrong value never produces an error, so **this report is the only place an inherited variable or a forgotten file becomes visible** — and a value that came from a default is worth seeing for the same reason, because next release it may be a different default."*

A **secret** setting prints `<set>` with the channel still named, so you know where to change it; one nothing supplied prints its empty default. **No credential appears either way.**

`--no-report-defaults` / `ATE_NO_REPORT_DEFAULTS=1` lists only what was set.

### Errors name the channel

```
Error: atenet.dataplane must be envoy or agentgateway, got "nginx" (from config.atenet.dataplane)
Error: at least one of ATE_API_POSTGRES_READ_WRITE_CONNECTION_STRING or
       config.ateapi.postgres.readWrite.connectionString must be set, got none
```

---

## Install records — reproducing an install

A `deploy` command that installs Substrate **writes the settings it resolved to a file.** (`deploy demo` and `deploy benchmarks` write nothing — they deploy on top of the system with only their own settings.)

```
recorded this configuration in ~/.cache/ate-setup/installs/prod.yaml
```

Default record directory: `~/.cache/ate-setup` on Linux, `~/Library/Caches/ate-setup` on macOS; `ATE_RECORD_DIR`/`--record-dir` to move it.

| Outcome | Path | Retention |
|---|---|---|
| Succeeded | `<record.dir>/installs/<context>.yaml` | **replaced each run** — a cluster has one current configuration |
| Failed | `<record.dir>/failed/<context>-<timestamp>.yaml` | **kept** — each attempt is its own artifact |

**Nothing reads them on its own — a record only takes effect when you pass it to `--config`.**

```yaml
apiVersion: install.ate.dev/v1alpha1
kind: SubstrateInstall
cluster:                                    # skipped when the file is read back
  context: prod
  outcome: succeeded
  substrateVersion: v0.4.1                  # the release this run installed
  writtenAt: "2026-09-29T23:14:38Z"
  writtenBy: ate-setup v0.4.1
atenet:
  dataplane: agentgateway
context: prod
namespace: ate-prod
```

**Only settings something supplied are recorded.** *"A setting left on its default is left out on purpose, so it keeps following the default if a later release changes it."*

### A failed run tells you how to retry

```
Error: while reading kubeconfig: context "prod" does not exist
the settings this run used were written to ~/.cache/ate-setup/failed/prod-20260929T231438Z.yaml
retry with: ate-setup deploy atenet --config ~/.cache/ate-setup/failed/prod-20260929T231438Z.yaml
1 secret setting(s) were not written and must be supplied again: ateapi.postgres.readWrite.connectionString
```

**A retry is not a complete replay** — secrets are never written, so that last line lists what to supply again. And: *"A failed run changes nothing about later runs. Its record is inert until you name it — a value that caused the failure must not become the default that reaches the cluster on the next success."*

### Reusing a successful record

```sh
# Rebuild a cluster the way it was, or set up a second one to match.
ate-setup deploy ate-system --config ~/.cache/ate-setup/installs/prod.yaml

# Point record.dir at a checkout to version-control them: how several operators
# share one configuration, and how an install survives a cluster that no longer exists.
export ATE_RECORD_DIR=~/ops/substrate-installs
```

**Records hold no credentials, so they are safe to commit. They are written readable only by you regardless.**

---

## The settings table

`ate-setup --help` and `--help` on the owning command print the defaults. A setting with no default is unset unless something supplies it. Settings for the demos, benchmarks and e2e tests are documented **with them** (`demos/counter/README.md#settings`, `demos/claude-code-multiplex/README.md#settings`, `benchmarking/README.md#settings`, `internal/e2e/README.md#settings`).

| Key | Env | Flag | Default |
|---|---|---|---|
| `namespace` | `ATE_NAMESPACE` | `--namespace` | `ate-system` |
| `context` | `KUBECTL_CONTEXT` | `--context` | — |
| `kubeconfig` | `KUBECONFIG` | `--kubeconfig` | — |
| `clusterSize` | `ATE_INSTALL_CLUSTER_SIZE` | `--cluster-size` | `size0` |
| `rolloutTimeout` | `ATE_INSTALL_ROLLOUT_TIMEOUT` | `--rollout-timeout` | `1m0s` |
| `preview` | `ATE_PREVIEW` | `--preview` | — |
| `images.repo` / `.tag` | `ATE_IMAGE_REPO` / `ATE_IMAGE_TAG` | `--image-repo` / `--image-tag` | — |
| `ko.dockerRepo` / `.defaultPlatforms` | `KO_DOCKER_REPO` / `KO_DEFAULTPLATFORMS` | `--ko-docker-repo` / `--ko-default-platforms` | — |
| `docker.buildFlags` | `DOCKER_BUILD_FLAGS` | `--docker-build-flags` | — |
| `storage.bucketName` | `BUCKET_NAME` | `--storage-bucket-name` | — |
| `atenet.dataplane` | `ATE_ATENET_DATAPLANE` | `--atenet-dataplane` | `envoy` |
| `atenet.egress.credentialProvider` | `ATE_CREDENTIAL_PROVIDER` | `--credential-provider` | — (**required**) |
| `atenet.egress.additionalExtprocService` | `ATE_ADDITIONAL_EGRESS_EXTPROC_SERVICE` | `--experimental-additional-egress-extproc-service` | — |
| `ingressAuthMode` | `ATE_INGRESS_AUTH_MODE` | `--ingress-auth-mode` | `deprecated-insecure` |
| `ateapi.authz.enabled` | `ATE_API_EXPERIMENTAL_ENABLE_AUTHZ` | `--ateapi-experimental-enable-authz` | **`true`** |
| `ateapi.authz.bootstrapOwners` | `ATE_API_AUTHZ_BOOTSTRAP_OWNERS` | `--ateapi-authz-bootstrap-owners` | **`system:serviceaccount:<ns>:ate-client`** ⚠️ |
| `ateapi.postgres.readWrite.connectionString` | `ATE_API_POSTGRES_READ_WRITE_CONNECTION_STRING` | **no flag** | — (**secret**) |
| `ateapi.postgres.owner.connectionString` | `ATE_API_POSTGRES_OWNER_CONNECTION_STRING` | **no flag** | — (**secret**) |
| `ateapi.postgres.readWrite.role` / `.owner.role` | … | `--ateapi-postgres-{read-write,owner}-role` | `substrate_readwrite` / `substrate_owner` |
| `ateapi.postgres.schema` | `ATE_API_POSTGRES_SCHEMA` | `--ateapi-postgres-schema` | — (`substrate` in ateapi) |
| `ateapi.postgres.poolMaxConns` | `ATE_API_POSTGRES_POOL_MAX_CONNS` | `--ateapi-postgres-pool-max-conns` | — |
| `ateapi.postgres.serverCAFile`, `.storageClass` | … | … | — |
| `ateapi.postgres.cloudsql.{instance,gsa,iamAuth,ipType}` | `ATE_API_POSTGRES_CLOUDSQL_*` | `--ateapi-postgres-cloudsql-*` | — |
| `ateapi.expectedJWTIssuer` | `EXPECTED_JWT_ISSUER` | `--ateapi-expected-jwt-issuer` | — |
| `ateapi.logLevel` | `ATE_API_LOG_LEVEL` | `--ateapi-log-level` | — |
| `actorJWT.issuer` / `.algorithm` | `ACTOR_JWT_ISSUER` / `ACTOR_JWT_ALGORITHM` | `--actor-jwt-{issuer,algorithm}` | — / `ES256` |
| `podcert.workersPerSigner` | `ATE_INSTALL_PODCERT_WORKERS_PER_SIGNER` | `--podcert-workers-per-signer` | `0` |
| `cordonControlPlane` | `ATE_INSTALL_CORDON_CONTROL_PLANE` | `--cordon-control-plane` | `false` |
| `fakeDataPlane.enabled` | `ATE_INSTALL_FAKE_DATA_PLANE` | `--fake-data-plane` | `false` |
| `kindCluster.enabled` / `.name` | `ATE_INSTALL_KIND` / `KIND_CLUSTER_NAME` | `--kind` / `--kind-cluster-name` | `false` / `kind` |
| `csi.setup` | `SETUP_CSI` | `--setup-csi` | `none` *(command-scoped: `deploy ate-system`)* |
| `gcp.{projectID,clusterName,clusterLocation}` | `PROJECT_ID`, `CLUSTER_NAME`, `CLUSTER_LOCATION` | `--gcp-*` | — |
| `otlpEndpoint` | `ATE_OTLP_ENDPOINT` | `--otlp-endpoint` | — |
| `record.dir` | `ATE_RECORD_DIR` | `--record-dir` | — |

**Secrets have no flag, deliberately:** *"A command line is visible in `ps` to every user on the host and is written to shell history, so a credential is supplied through the environment or through a file you can permission."*

Most settings are root flags; a few are **command-scoped** (`csi.setup`, `benchmark.*`) but **still reachable from the environment and the file, so one document can describe a whole install.**

---

## The install bundle

`manifests/ate-install/`:

| File | Component |
|---|---|
| `ate-api-server.yaml` (14k) | the control plane + its objectstore-plugin sidecar |
| `atelet.yaml` (15k) | the DaemonSet + its objectstore-plugin sidecar |
| `ate-controller.yaml` | the Kubernetes controller |
| `ate-idp-server.yaml` | OIDC discovery + JWKS |
| `atenet-router.yaml` (16k) | ingress gateway (atenet + Envoy) |
| `atenet-egress.yaml` (**78k**) | egress gateway — the Envoy config is most of it |
| `atenet-router-monitoring.yaml` | scrape config |
| `pod-certificate-controller.yaml` | **applied/deleted by the installers directly**, not from the bundle |
| `sandboxconfig-gvisor.yaml` | the default `gvisor-default` SandboxConfig |
| `sandboxconfig-validation.yaml` | the **ValidatingAdmissionPolicy** enforcing per-class asset requirements |
| `ate-system-namespace.yaml` | the namespace |
| `ate-system-networkpolicies.yaml` | **every control-plane pod denies ingress by default**, admitting only the flows the control plane needs |
| `ate-otel-config.yaml` | the shared OTLP ConfigMap every component consumes via `envFrom` |

Overlays and components:

| Directory | For |
|---|---|
| `base/` | the common bundle |
| `kind/` | the local-dev overlay: rustfs (S3), in-cluster otel-collector + Jaeger, Prometheus, bundled postgres, and a **replacement** `ate-otel-config.yaml` |
| `postgres/`, `postgres-size10/` | the bundled database, at two sizes |
| `cloudsql/` | Cloud SQL |
| `podcert-size10/` | a larger podcertcontroller |
| `agentgateway/`, `agentgateway-router/`, `agentgateway-egress/`, `kind-agentgateway/` | the agentgateway dataplane |
| `components/` | kustomize **components**, e.g. `router-static-mtls` |
| `generated/` | generated CRDs |
| `microvm/` (top level) | micro-VM extras |
| `egress-credential-injection/` (top level) | the bundled credential provider |

### Control-plane isolation

When the installer is told to, control-plane workloads select a node label and tolerate the matching `NoSchedule` taint:

| Label | Holds |
|---|---|
| `ate.dev/workloadType=ate-control-plane` | every control-plane workload except postgres; replicas spread across its nodes **best-effort** |
| `ate.dev/workloadType=ate-postgres` | **a one-node pool for postgres alone** |

So a burst of worker or actor load cannot starve the control plane.

Related: node pools reserved for one sandbox class are tainted `ate.dev/sandboxClass=<class>:NoSchedule`, and `ate-controller` adds the matching toleration to every pool's worker pods. **atelet tolerates the key with `operator: Exists`**, so it runs on every node a WorkerPool actor may reach, for any current or future class value.

---

## The database

`docs/postgres.md`.

### Two connections, two roles

| Pool | Role | Used for | Max conns |
|---|---|---|---|
| read/write | `substrate_readwrite` | application queries (DML) | `--postgres-pool-max-conns` |
| owner | `substrate_owner` | **migrations and partition maintenance** (DDL) | 2 |
| watch | read/write | the outbox watch | 3 |
| readiness probe | read/write | `/readyz` | 1 |

Roles are `NOLOGIN`; `ateapi` runs `SET ROLE` after connecting. **`--postgres-schema` (default `substrate`) overrides any `search_path` in the DSN.** Login users come from the provider — ordinary users, IAM identities, whatever — and Substrate neither creates them nor requires specific names.

### The bundled database has no passwords

`ate-api-server` presents **two projected Kubernetes Pod Certificates** from `postgres.podcert.ate.dev/identity`, one per pool, with `userAnnotations.postgres.podcert.ate.dev/username` set to `substrate_owner_user` or `substrate_readwrite_user`. The signer sets the certificate CN to that login and **rejects missing annotations, unknown keys, and every other username.**

The server trusts **only this signer's dedicated CA** for client certificates, and its `hostssl … cert` rule requires CN == the requested username. `ateapi` uses `sslmode=verify-full` against the server's service-DNS certificate. New connections reread the projected credentials and trust roots; the **TLS-reloader checks every 60 seconds** and requests a config reload on change — **existing connections stay open.** Pod Certificates require a minimum lifetime of one hour; the signer's default is 24 hours. Local Unix-socket access inside the database pod stays trusted for bootstrap, health checks and TLS reloads.

**The real authorization boundary is the namespace + service account**, not the username annotation:

> *"Kubernetes validates those identity fields in each PodCertificateRequest; the username annotation only selects between the two logins authorized for that identity. **Anyone allowed to create pods using `ate-api-server` in that namespace can request either login**, so operators must restrict that capability to trusted control-plane administrators. Both credentials are available to the API-server process; separate logins do not isolate the owner role from a compromised API server."*

`ate-controller`'s `--postgres-client-namespace` (default `ate-system`) is the namespace the signer will issue to. **Set it to the API server's namespace when relocating an install** — the certificate controller runs in a separate namespace.

### Bring your own database

`pkg/postgressetup/setup.sql` is a `psql` script requiring all of: `substrate_schema`, `substrate_owner_role`, `substrate_owner_user`, `substrate_owner_password`, `substrate_readwrite_role`, `substrate_readwrite_user`, `substrate_readwrite_password`. **It has no built-in identity defaults** — `ate-setup` supplies the development values from `postgressetup.DefaultConfig`.

The external database must already contain, **before the first `ateapi` startup**:

- a database both connection strings select (any name)
- a schema owned by the owner role (default `substrate`)
- a stable **`NOLOGIN`** owner role (default `substrate_owner`)
- a stable **`NOLOGIN`** read/write role (default `substrate_readwrite`)
- one or two login users: the owner login a member of the owner role, the runtime login a member of the read/write role. **One login may be a member of both when separate credentials are unavailable.**

Grants: the runtime role needs `SELECT, INSERT, UPDATE, DELETE` on tables; `USAGE, SELECT, UPDATE` on sequences; `EXECUTE` on routines; `USAGE` on types. **Configure the owner's default privileges so newly migrated objects receive these grants.**

> **`ateapi` and `ate-setup` never provision an external database.** Cloud SQL with IAM database authentication and no passwords: `tools/setup-gcp/cloud-sql.md`. Scaling: `docs/dev/cloud-sql-scaling-guide.md`.

### Schema evolution

`docs/dev/postgresql-schema-evolution.md`, gated by `hack/verify/postgresql-migrations.sh` and a dedicated CI step ("Verify immutable PostgreSQL migrations"):

- **Every migration must keep working for the previous `ate-api-server` binary.**
- **Migrations are never rolled back.**
- Migrations are **immutable** once merged — add a new one.

Goose-style files in `cmd/ateapi/internal/store/atepg/migrations/`.

---

## GCP provisioning

`tools/setup-gcp` — its own Go module.

```sh
cp hack/ate-dev-env.sh.example .ate-dev-env.sh    # edit, then
source .ate-dev-env.sh
gcloud auth application-default login --project=${PROJECT_ID}

go run ./tools/setup-gcp bootstrap        # GKE + GCS + IAM, all of it
go run ./tools/setup-gcp create cluster   # or individual steps
go run ./tools/setup-gcp create bucket
go run ./tools/setup-gcp --help

hack/teardown.sh --all                    # reverse order, handles partial failures
```

> ⚠️ **The required Kubernetes beta APIs can only be enabled *at cluster creation*.** If you bring your own cluster, read the "Create Cluster" warning in `tools/setup-gcp/README.md` first.

On a fresh project, `bootstrap` also creates the **atelet Workload Identity IAM grants that snapshots depend on** — `tools/setup-gcp/README.md#what-create-iam-actually-grants` to audit them or apply them by hand.

Clusters created this way always enable **Managed OpenTelemetry**, which `kubectl ate --trace` needs.

---

# Upgrades

`docs/upgrade.md` (316 lines) implements `docs/version-skew-policy.md` and may add restrictions of its own. **Read both before upgrading anything.**

## What is supported

| From → to | Supported |
|---|---|
| a later **patch** of the same minor (1.2.1 → 1.2.4) | **Yes.** Patch releases can be skipped. |
| the **next minor** (1.2.x → 1.3.y) | **Yes** |
| **skipping** a minor (1.2.x → 1.4.y) | **No.** Upgrade to 1.3 first. |
| a new **major** (1.x → 2.x) | **No.** Reinstall. |
| a minor **before 1.0** (0.3.x → 0.4.y) | **No. Reinstall.** Before 1.0 breakage still exists between minors. |

**You can roll back to the release you upgraded from, and no further.**

Minors cannot be skipped because *"the two-release steps and the database schema evolution both assume each upgrade moves one minor release."*

The upgrade does **not** change the database engine, the CSI drivers `--setup-csi` installs, the Kubernetes version, or the node OS.

## The component order

Components are ordered: **`podcertificate-controller` → `atelet` → workers → `ate-api-server` → `ate-controller` → `atenet` → clients.**

- **A component may be one release newer than any component *after* it in this order.**
- **A component must not be newer than any component *before* it.**

So during an N→N+1 upgrade these mixtures are supported and tested: a new atelet driving old ateworkers on its node; a new atelet calling an old ate-api-server; an old ate-controller rendering Deployments for new workers; an old ate-controller and old clients talking to a new ate-api-server.

**The reverse pairs are not supported.** In particular: **a worker must not be newer than the atelet on its node, and `ate-api-server` must not be newer than any worker.** The reason: *"`ate-api-server` reaches the workers through `atelet`, and a new `ate-api-server` may rely on new worker behavior behind existing fields, so the workers go first."*

## The nine steps

```
1. podcertificate-controller                     ~a minute        actors see nothing
2. atelet, node by node                          secs–~6 min/node actors see nothing
3. every WorkerPool's spec.workerImage           secs–hours/pool  SIGTERM + 30 min, then SIGKILL
4a. ate-api-server (the first N+1 replica migrates the schema)
4b. the CRDs, then ate-controller
4c. atenet                                                        a request or outbound connection
                                                                  still open ~15s into a drain is cut
4d. the SandboxConfig admission policy + default SandboxConfig
4e. the rest of the system bundle
5. clients (go install ./cmd/kubectl-ate)
```

Rollback is the same list in reverse with the old release, **except that the schema migration and step 4e are not undone.** Clients stop sending N+1 fields before `ate-api-server` goes back, and **atelet goes back only after every worker is back at N.**

> [!CAUTION] **Do not run `ate-setup setup csi` or `ate-setup delete` during the upgrade, or `ate-setup deploy ate-system` before step 4e.** They change components out of order.

## The step that affects actors: step 3

- **Change the WorkerPool, not its Deployment** — `ate-controller` overwrites Deployment edits. **If your WorkerPools live in Git or are applied by CI, change `workerImage` there too**, or the next sync puts the old image back and rolls every worker again.
- **Each pool needs the image for its own sandbox class.** Check `echo "$NS/$WORKERPOOL $CLASS $NEW_IMAGE"` shows `ateworker-` + the class and a full `@sha256:` digest — *"A bad image still removes the pool's first old worker."*
- **Pace:** a pool replaces **10% of its workers per batch, rounded down, at least one.** With spare capacity every batch gets its `SIGTERM` within minutes and old workers drain in parallel; **without it, each batch waits up to 30 minutes for old workers to exit.**
- **Actors:** each gets `SIGTERM` and **30 minutes to be suspended. Substrate does not suspend them.** An actor not suspended by then becomes `CRASHED` and **loses its work since its last snapshot.** `docs/api-guide.md#eviction` says what an actor needs to keep its state.
- Start with a small pool of each class you can afford to disrupt; once the first pool of a class has one new worker running, patch the other pools of that class **without waiting** for the roll to finish.

## The irreversible step: 4a

> **"The first new API server pod migrates the database schema, and rolling back does not undo that. Back up the database first"** — `pg_dump` for the bundled one, an on-demand backup for Cloud SQL.

With an external database, **check `ATE_API_POSTGRES_READ_WRITE_CONNECTION_STRING` is set in this shell: without it, `deploy apiserver` points the API server at the bundled database.** (A record holds no connection strings.)

The schema is shared, so **while `ate-api-server` rolls, replicas at N and N+1 both use the N+1 schema.** The API schema is *not* shared: until every replica is at N+1 a request can reach an N replica, which rejects fields new in N+1 — **so clients must not use new fields until the roll finishes.**

## "Where am I?" — the state check

Designed for coming back after a break or a handoff:

```sh
kubectl -n podcertificate-controller-system get rs
kubectl -n ate-system get rs,controllerrevision
kubectl get rs -A -l ate.dev/worker-pool
kubectl get deploy -A -l ate.dev/worker-pool -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,UP-TO-DATE:.status.updatedReplicas,READY:.status.readyReplicas,WANTED:.spec.replicas,PAUSED:.spec.paused'
```

> *"Each time a Deployment moves to a new image it gets a new ReplicaSet, and the atelet DaemonSet a new ControllerRevision, so their `AGE` tells what has moved. A component has moved when the `AGE` of its ReplicaSet with pods (`DESIRED` above 0), or of atelet's ControllerRevision with the highest `REVISION`, is shorter than the time since the upgrade began. **Two ReplicaSets with pods mean a roll is under way.**"*

A pool is done rolling when `UP-TO-DATE` and `READY` equal `WANTED`.

## The setup block

The runbook has you clone **both** releases, keep both configs, and run everything from one shell:

```sh
NEW_RELEASE=v0.4.2
RECORD=~/.cache/ate-setup/installs/my-cluster.yaml
kubectl config use-context my-cluster
OLD_RELEASE=${OLD_RELEASE:-$(sed -n 's/^ *substrateVersion: //p' $RECORD)}
mkdir ~/ate-upgrade && cd ~/ate-upgrade
cp $RECORD old.yaml && cp $RECORD new.yaml
sed -i.bak "s/^  tag: .*/  tag: $NEW_RELEASE/" new.yaml
git clone --branch $OLD_RELEASE https://github.com/agent-substrate/substrate.git old
git clone --branch $NEW_RELEASE https://github.com/agent-substrate/substrate.git new
(cd old && go install ./cmd/kubectl-ate)      # the OLD client, for rollback
cd new
```

You need `git`, `go`, `kubectl`, and **`crane` for prebuilt images** (to pin digests). **Agree a time with actor owners** — their running actors get `SIGTERM` in step 3.

`kubectl ate revert actor <name> -a <atespace>` puts an actor that crashed along the way back to its last snapshot.

## The requirements each release must keep

These are what make the skew rules hold, from `docs/version-skew-policy.md`:

### The worker pod template
`ate-controller` upgrades **after** the workers, so workers start the N+1 image with the Deployment N rendered. **Within a major, `ate-controller` renders the same worker pod template for the same WorkerPool.** New ateworker behavior comes from the image or a new WorkerPool field, **not from the template** — so `ateworker` accepts the flags and environment the template passes for the whole major, and **no WorkerPool rolls at `ate-controller`'s step**, in an upgrade or a rollback.

### Unknown API fields
| Component | On a field it does not know |
|---|---|
| `ate-api-server` on a **request** | **reject** with `INVALID_ARGUMENT` |
| `ate-api-server` on a **stored object** | **drop it and serve the reduced object.** *"Such a field was written by a newer replica… Rejecting it would make the object unreadable. Keeping it would serve a field this release cannot validate."* |
| `atelet` | **reject.** Its callers are never newer in the supported order, so an unknown field means the order was not followed. |
| `ateworker` | **reject.** Its caller `atelet` *can* be one release newer, so **atelet sends a new field only on releases after ateworker learns it** (two-step releases). |
| a client that reads-modifies-writes | **keep** the unknown fields. *"An older client must not strip fields that a newer server still understands."* |
| `kubectl-ate` on a manifest file | **reject**, as the server does for a create |

**Wherever a caller can be newer than its server, the caller sends a new field only one release after the server learns it.** Today that is atelet→ate-api-server and atelet→ateworker.

### CRDs
The N+1 CRDs are applied with `ate-controller` N+1. Components upgrading before it read the N CRDs, **so users must not set new CRD fields until the upgrade completes.** Between compatible releases a CRD change only **adds optional fields.** The CRDs serve one version with no conversion — **so a rollback that re-applies the N CRDs removes every field N does not know from every object.**

### SandboxConfig
Each release ships its SandboxConfig under its own name and keeps the earlier ones; the upgrade adds the N+1 one and pools without a `configRef` switch to it. **That does not change the pod template, so workers do not roll.** An actor resolves its pool's SandboxConfig **each time it starts** — *"If the runtime its snapshot needs is gone, it cold boots, keeping its files and losing its memory."* Rollback deletes the N+1 SandboxConfig.

*(A TODO in the runbook notes step 4d currently overwrites `gvisor-default` in place; the per-release naming is the target state.)*

### Node data and snapshots
- **Node layout:** what N writes N+1 can read, **and what N+1 writes N can read.** So a layout change takes **two releases** — N+1 reads old and new but still writes old; N+2 writes new. This is what keeps a rollback possible. **atelet passes the per-actor directories to ateworker in the RPC, so only atelet knows the layout**, and the sockets they find each other by do not change within a major.
- **Snapshot `manifest.json`:** within a major, a release reads what any earlier release of that major wrote, and **a release ignores manifest fields it does not know** — so a snapshot taken at N+1 still restores after a rollback to N. **A new field must be safe to ignore.**

### The system bundle
Applied last, and **a rollback does not undo it.** So a component at N+1 must work with these resources at N, **and** a component at N must work with them at N+1. *"Changes that only take something away, such as a NetworkPolicy that denies traffic, can go in this step. Anything a new component **needs** to run, such as a new Secret or a rule that allows its traffic, ships in that component's own step."*

### Patch releases
**Add no API fields, public or internal, and no schema migrations.**

### Testing
Before a release is cut, an install at the previous compatible release is **upgraded to it with actors running, and rolled back**, in this order.

---

## Operational quick reference

```sh
# Re-deploy one component
hack/install-ate.sh --deploy-ate-apiserver
hack/install-ate.sh --deploy-atelet
hack/install-ate.sh --deploy-atenet
hack/install-ate.sh --help
hack/install-ate.sh --delete-all

# Demos
hack/install-ate.sh --deploy-demo-counter
hack/install-ate.sh --deploy-demo-counter-microvm
hack/install-ate.sh --deploy-demo-egress-microvm
hack/install-ate.sh --deploy-demo-counter-with-external-volume --preview=ExternalVolumes

# kind
hack/create-kind-cluster.sh ; hack/delete-kind-cluster.sh
hack/install-ate-kind.sh --deploy-ate-system --credential-provider='{"name":"k8s.io"}'

# GCP
go run ./tools/setup-gcp bootstrap ; hack/teardown.sh --all

# CSI drivers for local testing
hack/setup-csi-hostpath-kind.sh ; hack/setup-csi-nfs-kind.sh
```

The demos: `counter`, `sandbox` (Antigravity/Alpine), `claude-code-multiplex`, `multi-template`, `parking`, `autoscaled-workerpool`, `jupyter`, `egress`. Each has a README; the installer has a `--deploy-demo-*` flag per demo.

## Where to read more

- `cmd/ate-setup/README.md`, `commands.md` (every command with its `hack/install-ate.sh` equivalent), `differences.md` (where the Go port deliberately diverges from the shell scripts it replaced), `cli-diff.md` (what a shell-installer user can observe after the switch).
- `docs/operator-install.md` — the authority on settings and records.
- `docs/upgrade.md` + `docs/version-skew-policy.md` — read together.
- `docs/postgres.md`, `docs/dev/postgresql-schema-evolution.md`, `docs/dev/cloud-sql-scaling-guide.md`, `tools/setup-gcp/cloud-sql.md`.
- `docs/csi-deployment.md`, `docs/egress-credential-injection.md#for-cluster-admins`.
- `docs/examples/substrate-install.yaml`.
- `docs/integration-repos.md` — where integrations live, how their repos are named, how fixes flow back.

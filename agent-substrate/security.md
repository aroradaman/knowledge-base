# Security: Model, Identity, and Threats

Substrate's pitch is "zero-trust sandboxing for millions of agents." The security documentation is unusually honest about what is and is not built, and that honesty is itself the most useful thing about it: `docs/security.md` labels **every principle** Met / In Progress / Future / Out of Scope.

Authorities: `docs/security.md`, `docs/threat-model.md` (43 numbered threats), `docs/threats.json` (the same, machine-readable, kept in sync by `hack/verify/threats.sh`), `AGENTS.md#security-considerations`, and `docs/authorization.md`.

> `AGENTS.md`: *"The security story for Substrate is very early and many features are missing. However! Take care to respect security best practices when writing code in order to improve Substrate's security over time."*

## The scope boundary

**Substrate provides:**
- **Sandboxing** — built-in compute, storage and networking separation.
- **Identity** — issuance is built in and designed to federate with other identity systems.
- **Gateway** — built-in ingress and egress filtering, destination filtering, credential injection.

**The application layer creating actors is responsible for:**
- human-to-actor authentication and authorization;
- registering and maintaining an agent registry;
- maintaining agent hierarchy/state to reconstruct lineage — *"who made this sub-sub-sub agent."*

### Extensibility — asymmetric by design

| | Replaceable? |
|---|---|
| **Egress** (`atenet-egress`) | **fully** — extend with Envoy `ext_proc` callouts, or **replace the gateway entirely** with an external solution |
| **Ingress** (`atenet-router`) | **supports `ext_proc` callouts, but cannot be replaced** |

The reason ingress cannot be swapped out: *"Substrate's core orchestration relies on ingress routing hooks to manage actor state, resolve worker assignments, and trigger on-demand cold starts or resume operations from snapshots to wake up agents."*

This is where advanced prompt-injection scanning, LLM-driven authorization, DLP, or cloud IAM policy enforcement is meant to go — **outside the untrusted sandbox.**

---

## The principles, with their real status

### Defensible multi-tenancy

| Principle | Status |
|---|---|
| **Always sandboxed** — every actor in a strong compute sandbox; **a Linux container is not sufficient or supported** | **Met** (gVisor, CHV micro-VM) |
| **Per-actor storage segregation** — no traversal, enumeration or access to another actor's persistence; a restore must re-validate tenant authorization | **Met** (segregation) / **Future** (per-actor encryption, restore authorization) |
| **Per-actor network isolation** — segregated guest stacks; lateral actor↔actor blocked by default, same host or across the cluster | **Met** (private netns, atunnel) |
| **Delete guarantee** — reset/scrub/sanitize local volumes, cached state and worker runtime between scheduling cycles | **In Progress** (`atelet resetActorDirs`) |
| **Node blast-radius containment** | **Future** |
| **Atespace separation** | **In Progress** (embedded OpenFGA). Today it groups resources and partitions storage prefixes; **it does not yet enforce inter-atespace network policy or resource quotas.** |

### Other tenancy shapes, and their honest status

- **Multiple Substrate installations in one Kubernetes cluster** — *possible, not currently supported.* Nodes separated by taints/tolerations gets API separation but **gives up some node bin-packing efficiency**, and care is needed at the Kubernetes layer.
- **Multiple Kubernetes clusters behind one Substrate installation** — *may be possible, not supported.* *"We're expecting a single large K8s cluster to be capable of running 100s of millions of mostly-idle agents."*
- **Multi-tenancy inside an actor — out of scope.** *"There is no meaningful security boundary inside an actor. Actors are designed to be cheap and efficient enough to run many, including one per human user."*
- **Per-actor ingress/egress — not supported.** *"Due to the high cardinality and rapid lifecycle of actors, dedicated 1:1 proxies per actor are operationally and economically infeasible. While we may partition proxies further in the future, by atespace for example, proxies will remain multi-tenant by default."*

### Identity

| Principle | Status |
|---|---|
| **Zero ambient credentials** — default sandboxes execute **completely credential-free**; an actor may hold identifiers, encrypted credentials or unprivileged tokens that Substrate infrastructure hydrates or decrypts out of band | **Met** |
| **Distinct workload identity** — every actor gets a short-lived (**1 h**) SPIFFE X.509; an actor deleted and recreated with the same name is distinguishable by **UID** | **Met** (X.509) / **In Progress** (OIDC discovery) |
| **Universal component mTLS** — every control- and data-plane component authenticates over mTLS with node- and service-attested identities; unauthenticated traffic rejected, **exempting only local unauthenticated `/healthz` and `/readyz`** | **Met** (mTLS) / **In Progress** (authz) |
| **Sub-actor provenance** — delegated sub-actors inheriting signed lineage | **Out of Scope** — must be recorded by the system that calls Substrate |

### Storage

| Principle | Status |
|---|---|
| **Untrusted storage** — remote storage encrypted; local snapshot caching on node storage is multi-tenant and protected by encryption and strict filesystem permissions | **Future.** *"Storage is currently partitioned but is unencrypted."* |
| **Dual-party snapshot authorization** — reading or writing snapshot storage requires proving **both** the actor's identity **and** that `atelet` is operating on its behalf; sandboxed guests never touch snapshot storage directly | **Future** |

### Network

| Principle | Status |
|---|---|
| **No access by default** — actors and worker nodes launch default-deny, with **zero network or API access to `ate-api-server`, the Kubernetes API, or PostgreSQL**. Every `ate-system` pod also [denies ingress by default](https://github.com/agent-substrate/substrate/blob/main/manifests/ate-install/ate-system-networkpolicies.yaml). | **Met** (loopback, private netns) / **In Progress** (DNS filtering, #2075) |
| **Authenticated and policy-enforced ingress** | **Met** (atenet-router) / **In Progress** (authentication) |
| **Fail-closed policy-enforced egress** — identity validation, destination allowlists, TLS inspection, credential injection; **if the PEP is unconfigured or fails, egress is dropped** | **Met** (atenet-egress) / **In Progress** (fail closed; **UDP currently passthrough**) |

### Authorization

| Principle | Status |
|---|---|
| **Actor-to-actor authorization** | **Future.** Until then: install an authenticating reverse proxy, or do it at the application level. |
| **Atespace and node-level containment** (declarative RBAC, quotas, metadata/label authorization) | **In Progress** (embedded OpenFGA) |
| **Comprehensive quotas and anti-DoS throttling** (API call rates, memory/CPU, egress bandwidth) | **Future** |

### Auditability and response

| Principle | Status |
|---|---|
| **Structured audit trail** — every lifecycle transition, snapshot access, routing event and policy violation as telemetry linked to cryptographic identity and parent lineage; token log leaks prevented by sanitization | **In Progress** (OTLP logging) / **Future** (enhanced identity attribution, log leak protection) |
| **Dynamic quarantine and snapshot tainting** — immediate revocation and network blackholing; compromised suspended states get an **immutable cryptographic taint** preventing fleet-wide resumption | **Future** |

---

## What is actually enforced today

From `AGENTS.md#security-considerations` — keep this section updated when it changes:

| Control | Mechanism |
|---|---|
| **Workload isolation** | gVisor (`runsc`) or Cloud Hypervisor micro-VM. **Plain Linux containers are not supported.** |
| **JWT signing and verification** | `internal/localjwtauthority` (signing, JWK set encoding) and `internal/oidcjwt` (verification), **built on the standard library's crypto packages only.** *"Extend those packages rather than adding go-jose or another JOSE library, so the code that decides whether a token is valid stays small and supports only the algorithms Substrate uses."* |
| **Ingress authentication** | `--ingress-auth-mode=static-mtls` requires a SPIFFE certificate signed by `--ingress-client-ca-file` whose ID is in `--ingress-allowed-spiffe-ids`. **That list is the whole authorization policy, so any listed client may reach any Actor.** The default, `deprecated-insecure`, checks a certificate only if one is presented and also serves plaintext listeners accepting any client. |
| **Control API authorization** | Every Control RPC checked against an embedded OpenFGA model (`cmd/ateapi/internal/authz/model.fga`), **on by default.** Every served unary RPC needs a `defaultRPCPermissions` entry — **without one it is denied and a test fails the build.** |

## Component identity: pod certificates

`cmd/podcertcontroller` is a **polyfill for Kubernetes PodCertificate signers that will eventually ship upstream (with different names).** Three signers, all backed by `internal/localca`:

| Signer | Issues | Identity derived from |
|---|---|---|
| `servicedns.podcert.ate.dev/identity` | serving certificates | **the Services that select the pod** — so a workload must be selected by a Service to get one |
| `podidentity.podcert.ate.dev/identity` | the component mTLS identity, equivalent to a ServiceAccount token | the pod's namespace + ServiceAccount → `spiffe://cluster.local/ns/<ns>/sa/<sa>` |
| `postgres.podcert.ate.dev/identity` | PostgreSQL login certificates for `ate-api-server` | namespace + ServiceAccount, with the login selected by the `postgres.podcert.ate.dev/username` annotation |

Each publishes a `ClusterTrustBundle` named `<signer>:identity:primary-bundle`, labeled `podcert.ate.dev/canarying=live`. Every component mounts a `credential-bundle.pem` (`internal/credbundle`: one PEM file, private key first, then the chain) and a `trust-bundle.pem`. **There are no static TLS Secrets anywhere in the install.**

This requires three Kubernetes feature gates (`ClusterTrustBundle`, `ClusterTrustBundleProjection`, `PodCertificateRequest`) plus `certificates.k8s.io/v1beta1`, which `hack/create-kind-cluster.sh` enables. Without them nothing gets an identity and the install never comes up.

## Actor identity

| | Certificate | JWT |
|---|---|---|
| Minted by | `MintActorCertificate` / `MintWorkerActorCertificate` | `MintActorJwt` / `MintWorkerActorJwt` |
| From | the actor-identity CA (`--actor-id-ca-pool`) | the authority pool (`--actor-id-jwt-pool`) |
| Lifetime | short-lived, **1 h by default** | **300–3600 s** |
| Carries | the `ActorIdentity` X.509 extension (atespace, name, **UID**, purpose) + an actor URI SAN | `sub = actor/<atespace>/<name>`, `aud`, `jti`, and `actor.ate.dev = {atespace, actorName, actorUID}` |
| Verified by | the egress PEP, against the actor-identity CA | any relying party, via **OIDC discovery** against `ate-idp-server` |
| Held by | **`atunnel` only** — the private key is generated at activation and **never leaves atunnel** | the egress gateway, injected into approved requests |

The `ActorIdentity` extension OID is `1.3.6.1.4.1.11129.2.12.2`, allocated under Google's enterprise number, and **will change after the CNCF donation is complete.** `internal/substratex509` creates and parses it, modeled on upstream Kubernetes.

**Delegation is visible in the subject.** A token minted for a Substrate *component acting on the actor's behalf* (`WorkerService.MintWorkerActorJwt`) uses **`worker-for-actor`** in both `sub` and the claim name instead of `actor`, *"so a relying party can grant the two different access."* `nbf` is five minutes before `iat` for clock skew. `actor_jwt` is `[debug_redact = true]`.

**Only the `egress_gateway` system role can mint actor credentials — not even a global owner can**, since a credential lets its holder act as the actor.

## Authorization — the quick version

See [control-plane.md](control-plane.md) for the full tables. Three things to internalize:

1. **An RPC with no rule is denied, and a test fails the build.** `cmd/ateapi/internal/authz/registry.go` + `model.fga`.
2. **System roles (`controller`, `egress_gateway`, `ingress_router`, `atelet`, `api_server_sidecar`) are bound to pod certificates and cannot be granted through an access policy.**
3. **With authorization off, access-policy RPCs are still checked**, so nobody can grant themselves access while it's off and keep it once it's on.

### The four known limitations — read before trusting it

- **Tags can copy an actor's data.** A tag is a durable copy of memory **and disk, including secrets**, that outlives the actor. **Any atespace editor can tag, publish and clone any actor in the atespace.**
- **Principals are not qualified by issuer.** With several trusted JWT providers, any of them that can issue a token whose `sub` matches another principal's ID **acts as that principal, bootstrap owners included.** (System roles are unaffected — they need the pod certificate.)
- **The atespace is the only boundary.** No per-actor or per-template grants.
- **Any atelet can read any actor's egress policy.** Every node's atelet shares one identity, so the `atelet` role is not scoped to actors on its node ([#1564](https://github.com/agent-substrate/substrate/issues/1564)).

### And the install-time warning

> ⚠️ **`ate-setup` makes the shared `ate-client` ServiceAccount the default bootstrap owner** so the demos, benchmarks and e2e tests work. Anyone who can mint a token for it — **including every `kubectl ate` user who doesn't pass `--token-file`** — is a global owner. For anything shared, set real bootstrap owners.

Production hardening (`docs/authorization.md#production-readiness`): give people their own identities and have them pass `--token-file`; and **limit who can act as a ServiceAccount bootstrap owner** — anyone who can create `serviceaccounts/token`, pods, or Secrets in its namespace, or change RBAC there, can get its token, and a `kubernetes.io/service-account-token` Secret gets one **that never expires.**

```sh
kubectl auth can-i create serviceaccounts/ate-client --subresource=token \
  -n ate-system --as alice@example.com
```

## The worker pod's privilege posture

Not privileged. `drop: ALL`, then 13 capabilities for gVisor (15 for micro-VM), with `seccompProfile: Unconfined` and `appArmorProfile: Unconfined` because **the sandbox child calls `pivot_root()`** and both default profiles deny it. Full rationale per capability group in [node-and-sandbox.md](node-and-sandbox.md).

This replaced a formerly **privileged** worker, and the code notes "replacing Unconfined with a tailored profile per runtime is a follow-up." **`atelet` holds only `DAC_OVERRIDE`**, for one specific job: emptying a directory a non-root actor created inside its `0777` durable dir.

Threat **T-21** is exactly this item, and the posture above is its partial mitigation.

---

# The threat model

`docs/threat-model.md` — **43 threats, T-01 to T-43**, each with a priority, mitigating invariants, suggested concrete mitigations, and notes. Mirrored in `docs/threats.json`, with `hack/verify/threats.sh` keeping the two in sync (`hack/export-threats.py` regenerates).

## How to read it

It was written deliberately **against the class of system, not this implementation**:

> *"Substrate is an early, fast moving product… It also has little to no security hardening at this time. Therefore, this threat model is focused on how to achieve security with respect to the general problem Substrate is trying to solve… The threat model uses Substrate's current implementation and roadmap as a reference point, but focuses on threats that emerge from this overall class of system, rather than threats specific to the current implementation."*

Its stated intended outcomes are worth noting: the suggestions should be **added to the official roadmap** after community agreement, and **security-review SKILLs for AI-assisted review should be extracted from it** for continuous review of the repo.

Non-goals: it does not predict future architectural decisions, does not demand a particular mitigation, and does not cover the AI frameworks built on top (shown in the diagram only).

## The seven categories

| § | Threats | Theme |
|---|---|---|
| External networks | T-01 – T-03 | internet reachability of actors, nodes, the API and the database. All **Critical**, and all largely **outside Substrate's ability to configure directly** — they depend on the surrounding Kubernetes/cloud configuration. *"This should at least be a default deny policy for ingress in Substrate."* |
| Internal network, API clients | T-04 – T-11 | an internal caller reaching what it shouldn't |
| Misconfiguration | T-12 – T-14 | the install being wrong rather than attacked |
| **Attacks from actors** | **T-15 – T-35** | **the core of the document — 21 threats, 14 of them Critical** |
| Attacks from nodes | T-36 – T-38 | a compromised node escalating |
| Insider attacks | T-39 – T-40 | insider access to snapshots and to disks |
| Detection and response | T-41 – T-43 | forensics, detection, containment |

## The actor threats worth knowing by number

These are the ones that explain why the architecture looks the way it does.

| ID | Threat | The invariant |
|---|---|---|
| **T-15** | Breakout to the node or other actors via container escape | **"Actors are always sandboxed using a hardened sandbox solution like gvisor or microvm. Traditional containers are not a secure sandbox."** Run the sentry in a user namespace, `pivot_root`, seccomp for sentry/gofer, limit directfs/sentry capabilities, keep sidecars low-privileged — and **"sandbox lifecycle must be controlled from outside the sandbox."** |
| **T-16** | Reaching node-local endpoints over the network | network policy must block instance metadata, host netns, host interfaces. *(This is the nftables link-local drop.)* |
| **T-17** | Actor→actor over the network | **default-deny ingress and egress, synchronized with actor lifecycle.** The note suggests implementing it **in ateworker rather than Kubernetes, "to speed up policy updates when actor groups do need to communicate."** |
| **T-18** | Breakout via the filesystem | map each actor to a unique uid, or use user namespaces; protect against symlink/mount traversal. Vectors listed: arbitrary file read in the fs implementation, symlink traversal, **a vuln in OCI image unpacking**, path traversal in mount configuration, shared directories gated only by filesystem permissions with workers running as root. |
| **T-19** | Escalation via local Substrate services (atelet, ateworker) | *"Ensure system services are aware of actor identity, and authenticate and authorize actors when access is required."* |
| **T-20** | Escalation via the Substrate API | **"Especially prevent self-modification, which was a common escalation path in K8s."** Do not let actors or workers read/write their own snapshots or self-label; **keep resource-modifying logic in the control plane, not the data plane.** |
| **T-21** | Escalation via excessive worker pod permissions | deprivilege the worker pods: non-root or user namespace, not privileged, only required caps/devices. Cited example: [#288](https://github.com/agent-substrate/substrate/issues/288), accidentally putting things in the same cgroup. |
| **T-22** | Escalation via the Kubernetes API | **"Strong preference on blocking actor access to Kubernetes."** Block network access; ensure the default SA token in each pod has zero privileges. |
| **T-23 / T-24** | Reading / **overwriting** other actors' snapshots | **"Worker Pods and actors must not have direct access to snapshots."** If actor identity gates snapshot access, require an extra claim identifying the caller as **atelet**, or require atelet's mTLS channel. Envelope-encrypt and tie decryption to actor identity. **"Avoid snapshotting sensitive credentials in the filesystem, where possible."** |
| **T-25** | A corrupt snapshot is restored | **cryptographically verify every snapshot before restore** — a trusted digest or a signature checked against a trusted key, delivered over a trusted channel. |
| **T-26** | **Stale policy propagated out of band with scheduling** | **"All network and authorization policies must be fully synced before an actor starts running."** Suggests actor lifecycle APIs program authz and network policy as part of create/resume. |
| **T-27** | **Worker reuse leaking state across tenants** | **"All actor-specific worker state, including process state, filesystem, env vars, mounted config, network policy, and security policy, must be completely reset between actors."** Suggested: tightly synchronize local policy updates with the lifecycle; **exercise suspend/resume for every sandbox technology and enumerate system state; use "honeypots" on each side of the boundary to detect leaks.** |
| **T-28** | Tricking the identity broker into issuing another actor's credentials | credential issuance must **check the actor→worker scheduling assignment**; identities should carry claims tying them to the worker and node; **rescheduling must immediately invalidate old credentials.** *(This is why the egress PEP must confirm with ate-api-server that the actor exists, the UID matches, and it is running.)* |
| **T-29** | **"Agent leaks credentials exposed in sandbox, because LLMs are unreliable. Due to prompt injection or just agent silliness."** | **"Credentials are not exposed in sandboxes by default."** Opt-in only; a credential-injecting proxy; delegation to "drivers" rather than sandboxed clients; encrypted credentials unwrapped by a network proxy. *(This threat is the entire reason `atenet-egress` and credential injection exist.)* The note allows exposure where there are mitigating factors — *"For example, the process is NOT an AI agent."* |
| **T-30** | Lateral movement to other nodes via suspend/resume, **especially if self-suspend is allowed** | *"Suspend/resume is guaranteed to provide some acceptable locality with respect to lateral movement. **TBD.**"* Suggested: actor pinning to node groups, or a statistical bias toward resuming on the same node. |
| **T-31** | DoS via a malicious container image | **enforce limits when pulling and extracting** — limits on uncompressed layers when untarring, a time limit on unpacking. *"For example, image can contain a zip bomb."* |
| **T-32** | A compromised in-actor process uses actor identity to **write its own snapshot** | *"Example: nonroot process with access to the actor credential rewrites a snapshot so that it can `su` to root after the next restore."* Invariant: issue separate credentials for snapshot access, or require **dual-identity authorization** proving atelet is acting on the actor's behalf. |
| **T-33** | Creating a huge number of actors (direct, or a fork-bomb equivalent) | **"Direct ability to create actors should be considered a privileged permission."** Rate quotas, throttling, accounting for child actors, a namespace-like concept to attach quotas to. |
| **T-34** | Discovering cluster network topology from inside the sandbox | **"Don't expose cluster internal DNS to actors. Don't mount worker Pod resolv.conf in actors, instead provision a new file for the actor."** |
| **T-35** | Discovering actor or namespace names | the note is the interesting part: actor names are opaque UUIDs, so *"What's more consequential is what you can actually send traffic to, since actors get the IP of the worker Pod and those are IPv4 allocated from the cluster's IP space, **it's possible to enumerate them with a network scan**."* |

### Node, insider and detection threats

| ID | Threat | Invariant |
|---|---|---|
| **T-36** | A compromised node accesses **all** snapshots in the cluster | *"Node access to snapshots must be scoped to actors actively scheduled to the node."* Per-actor JWT for atelet only (never in the sandbox) with claims usable in conditional IAM; GCP CredentialAccessBoundary; data wipeout on suspension; **snapshots write-once per version "to prevent malicious actors from overwriting 'golden' snapshots."** |
| **T-37** | A compromised node reaches other nodes' network filesystems | *"TBD — depends on network filesystem implementation."* |
| **T-38** | A compromised node escalates to other nodes or the control plane | tightly scoped ingress/egress per pod; default-deny actor↔actor; TLS + authn/authz on any allowed actor↔actor; **limit the number and depth of child actors.** |
| **T-39** | Insider access to snapshots | customer-managed encryption keys (like the Kubernetes `kmsplugin`); a storage layout mapping to per-actor identity |
| **T-40** | Insider access to disks (the backend DB) | FDE by default (most clouds already); envelope encryption with HSMs if secrets ever land in the DB |
| **T-41** | **Malicious actions are not recorded for forensics** | **"Enable audit logging for all Substrate components that serve APIs, including node-local components."** Audit logging on the snapshot bucket; audit logging for ateapi requests. |
| **T-42** | Malicious activity undetected | pluggable threat-detection integrations; making Substrate API, node and sandbox telemetry available for continuous analysis; **pluggable per-node or per-worker security sidecars** |
| **T-43** | A detected malicious actor cannot be contained | **quarantine or suspension**: dynamic policy updates for quick quarantine; automatically snapshot/suspend on malicious activity **but taint the snapshot so it can't be resumed in production.** |

## Mapping the threat model onto the code

Useful when deciding whether a change is security-relevant:

| Threat | Where it lives in the code |
|---|---|
| T-15, T-21 | `workerSecurityContext` in `cmd/atecontroller/internal/controllers/workerpool_apply.go`; `cmd/ateworker-*/` |
| T-16, T-17, T-34 | `internal/ateworker/actornet/sandbox.go` (nftables), `internal/atunnel/dns` |
| T-18, T-27 | `cmd/atelet/main.go` `resetActorDirs`/`resetSandboxAssetsDir`; `internal/imagecache`; `internal/volumepath` |
| T-19 | `internal/ateletdial`, `nodepath.WorkerSupportSocket` |
| T-20, T-22 | the actor has no route to either API — `actornet`'s rules plus the `ate-system` NetworkPolicies |
| T-23, T-24, T-32, T-36 | the **objectstore-plugin sidecar** pattern: no main process holds storage credentials; snapshot prefixes keyed on owner UID |
| T-25 | **not yet done** — snapshots are not signed; the sha256 chain covers *sandbox assets*, not snapshots |
| T-26 | `ensureVolumesAttached` / the egress policy cache TTL; the `EgressPolicy` read at activation |
| T-28 | the egress PEP's `GetActor` check (exists / UID matches / RUNNING); `worker_epoch` on the assignment |
| T-29 | the whole of `atenet-egress` + `docs/egress-credential-injection.md` |
| T-30 | `scheduling.Constraints.RequiredNodes`, `status.assignedNode` |
| T-31 | `maxAssetBytes` (8 GiB) and `--asset-staging-capacity` bound **assets**; image unpack limits are weaker |
| T-33 | **not yet done** — no quotas |
| T-41 | `internal/actorevent` + the lifecycle event stream |

---

## A security-review checklist for a change

1. **Does it put a credential, or a path to one, inside the sandbox?** T-29 says no. Credentials belong at the egress gateway.
2. **Does it give the actor a new network path?** Default deny is the invariant (T-16, T-17, T-22). A new path needs an explicit policy, and policy must be in place **before the actor runs** (T-26).
3. **Does it let an actor or a worker reach its own snapshot?** T-23/T-24/T-32 say no — only atelet, and ideally with dual-party authorization.
4. **Does it add per-actor state on a worker?** Then it must be reset between actors (T-27). Add it to `resetActorDirs` and test the suspend/resume cycle.
5. **Does it widen the worker pod's privileges?** T-21. Justify each capability in a comment, as the existing sets do.
6. **Does it let an actor or worker modify its own resource?** T-20: *"Logic to modify resource definitions… lives in the control plane, rather than the data plane, as much as possible."*
7. **Does it add an unbounded resource an actor can consume?** T-31, T-33. Bound it.
8. **Does it add a new RPC?** It needs a `defaultRPCPermissions` entry or it is denied and the build fails.
9. **Does it add a field that may hold a secret?** `[ debug_redact = true ]`, in the same change.
10. **Does it change a lifecycle transition?** The event stream is the audit trail (T-41). Keep it accurate; **never sample it.**

Run `/security-review` on the branch (a skill in this environment), and `hack/verify/threats.sh` if you touched the threat model.

## Where to read more

- `docs/security.md` — the principles with their real status. Short; read it whole.
- `docs/threat-model.md` + `docs/threats.json` + `docs/assets/threat-model-diagram.svg`.
- `docs/authorization.md`, `docs/authentication.md` — complete and current.
- `docs/network-egress.md` — the normative PEP contract (MUST/MUST NOT).
- `docs/egress-traffic.md` — exactly what an actor may and may not send.
- `docs/egress-credential-injection.md` — the fail-closed table.
- `docs/roadmap.md#security` — the planned items, which largely track the Future rows above.
- `AGENTS.md#security-considerations` — what is implemented **right now**; keep it updated.

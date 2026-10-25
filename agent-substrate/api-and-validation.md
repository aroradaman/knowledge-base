# The API: Style Guide, Declarative Validation, Codegen

Substrate's gRPC API is the project's most rule-bound surface. The two authorities are `docs/api-style-guide.md` (744 lines) and `docs/api-validation.md` (443 lines), both written as normative specs. **Read them before touching `pkg/proto/ateapipb/ateapi.proto`** — a review will hold you to them.

The short version: it is **Google AIP with four deliberate divergences**, validated by **Kubernetes `validation-gen`** driven off comments in the `.proto`.

## The four divergences from AIP, and why

| AIP | Substrate | Reason given |
|---|---|---|
| **AIP-122**: one opaque path string (`publishers/123/books/les-mis`) | **two-field identity**: `(atespace, name)` in a `ResourceMetadata` | *"analogous to how Kubernetes identifies objects and avoids the ambiguity of parsing hierarchical path strings"* |
| **AIP-133**: `parent` + `{resource}_id` separate from the body | **identity lives in the embedded resource only** | AIP-133 splits them because AIP-122 makes `name` output-only. Here `atespace`/`name` are caller-specified fields on the resource, so duplicating them *"adds no information and creates ambiguity about which one wins."* |
| **AIP-134**: patch with a field mask | **full replacement** (HTTP PUT semantics) | the resource in the request **is** the resource the client wants to exist |
| **AIP-154**: an opaque `etag` string | **`int64 version`** | *"Substrate maintains an in-memory worker cache that guards against applying stale watch events over newer cached state using a numeric `>=` comparison. An opaque string cannot serve this role — the ordering guarantee IS the implementation contract, so the field type should reflect it."* |

The name `version` is also deliberate: it increases on every write (like Kubernetes `resourceVersion`) **and** is a transparent comparable integer (like `generation`), so neither Kubernetes name fits cleanly.

Also diverging, more mildly: `ListOptions` groups `page_size`/`page_token` instead of putting them at the request's top level, following the `DeleteOptions` pattern *"so every List method carries the same controls with the same validation."* And Delete returns the **deleted resource**.

## Standard-method shapes

Memorize these; a reviewer will.

```proto
rpc GetActor(GetActorRequest) returns (Actor) {                 // returns THE RESOURCE
  option (method).resource = "Actor";
}
message GetActorRequest { ObjectRef actor = 1; }                // field named after the
                                                                 // resource's snake_case type
                                                                 // name — not "name" or "ref".
                                                                 // NO other fields.

rpc ListActors(ListActorsRequest) returns (ListActorsResponse) { ... }
message ListActorsRequest  { string atespace = 1; ListOptions options = 2; }
message ListActorsResponse { repeated Actor actors = 1; string next_page_token = 2; }

rpc CreateActor(CreateActorRequest) returns (Actor) { ... }
message CreateActorRequest { Actor actor = 1; }                  // identity inside the resource

rpc UpdateActor(UpdateActorRequest) returns (Actor) { ... }
message UpdateActorRequest { Actor actor = 1; }                  // uid + version REQUIRED

rpc DeleteActor(DeleteActorRequest) returns (Actor) { ... }      // returns the DELETED resource
message DeleteActorRequest { ObjectRef actor = 1; DeleteOptions options = 2; }

rpc SuspendActor(SuspendActorRequest) returns (SuspendActorResponse) { ... }  // CUSTOM method:
message SuspendActorResponse { Actor actor = 1; }                // NEVER returns the resource
                                                                 // directly — wrap it, so
                                                                 // method-specific output can
                                                                 // be added later
```

### Rules that are easy to violate

- **Get/Create/Update return the resource itself**, never a `...Response` wrapper. **Custom methods must not** — put the resource in a field of the response.
- A custom method's name **must not** start with a standard verb. One that does *is* a standard method and must follow those rules.
- The `ObjectRef`/embedded-resource field is named after the resource's own snake_case type name (`actor`, `tag`, `worker`) — **not** `name`, `ref`, `resource` or `body`.
- `next_page_token` must be present on **every** List response and empty only on the last page. **Clients must iterate until it is empty and must not treat `len(items) == 0` as the end.**
- `page_size` above the maximum (**1000**) is **silently coerced**; a negative value is `INVALID_ARGUMENT`.
- **Sorting and filtering (AIP-132) are not supported.**
- Non-resource control fields (dry-run, idempotency token) go in a shared `CreateOptions`/`DeleteOptions`/`ListOptions` as an `options` field — **never as loose top-level request fields.**

### Status codes

| Condition | Code |
|---|---|
| resource does not exist (Get/Update/Delete) | `NOT_FOUND` |
| `(atespace, name)` already taken (Create) | `ALREADY_EXISTS` |
| `atespace` set on a global resource, or unset on an atespaced one | `INVALID_ARGUMENT` |
| an unknown field at any nesting level | `INVALID_ARGUMENT`, **naming the field** |
| an update that changes an immutable field — **including by omitting it** | `INVALID_ARGUMENT`, naming the field |
| an update missing `uid` or `version` | `INVALID_ARGUMENT` (decided *before* the resource is read) |
| a guard that does not match the server's value | `ABORTED` |
| negative `page_size` | `INVALID_ARGUMENT` |

## Optimistic concurrency: two guards, neither substituting for the other

This is the part most worth understanding properly.

- **`version`** guards against **concurrent modification**. It changes on every write, so echoing it back detects that someone else wrote in between (the lost-update problem). *"A `version` guard can legitimately reject an otherwise-valid read-modify-write — that is the point."*
- **`uid`** guards against **name reuse across lifecycles**. `(atespace, name)` is unique only at a point in time; `uid` is unique across time. Because `uid` is immutable within a lifecycle it never causes a spurious rejection — it fires only when the name was deleted and recreated as a different resource. **This catches the ABA case `version` alone cannot: `version` restarts at 1 for each new lifecycle, so a stale write's `version` can happen to match the *new* resource.**

`version` is monotonic but **the magnitude of the increase is not part of the contract — clients must not assume consecutive versions differ by exactly 1.** It bumps on system-internal changes too (the scheduler binding a worker), not just user-visible ones.

### Update is always read-modify-write

**Both guards are required on Update.** There is no blind by-name update: a client that has not read the resource cannot update it, and one that has read it already holds both values. Delete's guards stay optional (the zero value skips the check), and Create assigns rather than checks — **so Update never gives last-writer-wins, while Delete still does by default.**

```go
// 1. Read.
actor, _ := client.GetActor(ctx, &GetActorRequest{
    Actor: &ObjectRef{Atespace: "my-space", Name: "my-actor"},
})

// 2. Modify THE MESSAGE YOU READ, in place (or clone it).
//    Do NOT build a fresh Actor.
actor.WorkerSelector = &Selector{MatchLabels: map[string]string{"tier": "paid"}}

// 3. Write it back. actor.metadata already carries uid and version.
updated, _ := client.UpdateActor(ctx, &UpdateActorRequest{Actor: actor})
```

Three client rules with teeth, all from the style guide:

1. **Modify what you read; do not reconstruct it.** *"This matters when the server is newer than the client: a reconstructed message does not carry the fields the client does not recognize, and an update that omits an immutable field is rejected rather than silently rewriting it."*
2. **Do not round-trip a resource through JSON between read and write.** `protojson` cannot represent unknown fields — **marshalling silently drops them.**
3. **Do not retry `ABORTED` updates server-side.** *"`ABORTED` means someone else wrote something you have not seen. A blind retry reapplies your change on top of a concurrent change nobody reviewed. Surface the conflict instead."*

The asymmetry about unknown fields is precise: a request carrying a field **the server** does not know is `INVALID_ARGUMENT`. Fields only **the client** does not know are a different case — a client that modifies what it read sends them back unchanged, *"so an update from an older client does not clear fields a newer server set."*

## Status: the output-only split

Output-only fields **must not** appear top-level on a resource. They are grouped into one nested `{Resource}Status` field named `status`. A resource with no output-only fields needs no `status`.

`ResourceMetadata` is the **one exemption** — it deliberately mixes caller-specified identity (`atespace`, `name`) with server-managed fields (`uid`, `version`, timestamps).

## Unions: never `oneof`

```proto
message Volume {
  // +k8s:required
  string name = 1;
  // Exactly one of the three must be set.
  // +k8s:optional
  // +k8s:unionMember
  DurableDirVolumeSource durable_dir = 2;
  // +k8s:optional
  // +k8s:unionMember
  ExternalVolumeTemplate external_volume_template = 3;
  // +k8s:optional
  // +k8s:unionMember
  ImageVolumeSource image = 4;
}
```

- **Protobuf `oneof` is forbidden** — it does not work with declarative validation. The sibling-fields form is **wire-compatible** with a `oneof`.
- Every member is `+k8s:optional` + `+k8s:unionMember`; generated validation enforces exactly one.
- **A primitive member must use the protobuf `optional` keyword**, so presence is explicit on the wire. Protobuf elides zero-valued primitives, so without it the server cannot tell a member set to `0`/`false`/`""` from an unset one. Use with care: a primitive cannot later be augmented with more fields.
- A `map` or `repeated` member takes no `optional` and needs none, but **has no presence either** — validation treats it as set when non-empty, so **a client cannot select such a member by sending it empty.** If "empty" must be a meaningful choice, wrap it in a message, which does have presence.
- **Unions must not carry a discriminator** (a `type`/`kind` enum naming which member is set). *"The member that is set is the discriminator. A separate discriminator is a second source of truth that can contradict the payload, and it forces every client to keep the two in sync for no gain."*

## Annotations — the two options every resource and RPC carries

```proto
message Actor {
  option (resource) = { scope: SCOPE_ATESPACE  plural: "Actors" };
}
message EgressPolicy {
  option (resource) = { parents: "Actor"  singleton: true  plural: "EgressPolicies" };
}
message AccessPolicy {
  option (resource) = { parents: ["Global", "Atespace"]  singleton: true  plural: "AccessPolicies" };
}

rpc GetActorEgressPolicy(...) returns (EgressPolicy) {
  option (method).resource = "EgressPolicy";    // the SUBRESOURCE, not its parent
}
rpc SuspendActor(...) returns (SuspendActorResponse) {
  option (method).resource = "Actor";
}
```

- `scope` **must** be set on a top-level resource and **must not** be set on a subresource (which has its parent's scope).
- `plural` **must** be set on every resource, singletons included — it's the plural `List` methods and their repeated field use.
- A non-resource message (a request, a status message, `ObjectRef`) carries **no** `(resource)` option.
- **Every RPC of `Control` must carry `(method).resource`.** A subresource's method names the subresource.
- **When you add, remove, or re-parent a resource, update `docs/resource-model.md` in the same change.**

## Sensitive fields: `debug_redact`

A field holding a secret **must** carry the standard protobuf `debug_redact` option. `internal/protoredact` masks labeled fields so request/response bodies can be logged.

| Label it | Do **not** label it |
|---|---|
| bearer tokens and JWTs (`MintActorJwtResponse.actor_jwt`) | a **reference** to a secret, e.g. an `ate-secret://` URI (`CredentialHeader.credential_uri`) — *"It names the secret and grants nothing by itself."* |
| keys, passwords, secret bytes | identifiers, names, list cursors (`page_token`) |
| user-supplied values that may hold secrets (`EnvVar.value`) | |
| a URL that grants access by itself, e.g. a signed URL | |

Two rules of thumb: **prefer labeling the leaf over the containing message**, so the rest stays readable (a log line still shows *which* env var was set, just not its value); and **add the label in the same change that adds the field** — *"A label added later does not help lines already logged."*

---

# Declarative validation (DV)

> *"All fields of all APIs must be validated. This is especially critical for the main resource-oriented API because (once we hit 1.0) when bad data has been written to a database, we can't easily fix it. Retroactively fixing validation to make it stricter will break users."*

Tooling: Kubernetes **`validation-gen`**. **Protobuf comments are copied directly into the generated Go code**, so comments in the `.proto` drive the generator. Almost all validation should be DV; anything else is *"rare and exceptional."*

```sh
./hack/run-tool.sh validation-gen --docs   # the supported tags
```

## Tag syntax

```
// +k8s:<tag-name>                                   # no payload
// +k8s:<tag-name>=<tag-value>                       # with payload
// +k8s:<tag-name>(<value>)                          # unnamed single argument
// +k8s:<tag-name>(<arg>: <val>)                     # named argument
// +k8s:<tag-name>(<a1>: <v1>, <a2>: <v2>)           # multiple (always named)
// +k8s:<tag-name>=+k8s:<other-tag>                  # nested
// +k8s:required # EOL comments use '#'
```

Unquoted strings must match `[A-Za-z_][A-Za-z0-9_-.]*`. Tags attach to a **field** or a **struct**; field tags apply to every use of the enclosing struct. Tags are mostly order-independent, but some are mutually exclusive and some **short-circuit** — e.g. a `required` field that is unspecified is not validated against most other tags.

**Alpha/beta tags** warn unless acknowledged with a prefix:

```proto
// +k8s:alpha(since: "0.0")=+k8s:monotonic   # TODO: get rid of alpha prefix
```

You will see these all over `ateapi.proto` (on `version`, `last_assigned_generation`, the `subfield(atespace)` forbids). The TODOs are real.

## The rules, by field kind

**Every field must be either `+k8s:required` or `+k8s:optional`.** No exceptions.

> *"Any field whose value is the zero value for that type is considered 'unspecified'. This is about the **value** of the field and NOT about whether the value was present on the wire. Protobuf generally omits zero values from the wire, but that is not a requirement. For example, if we received a protobuf message with an optional int32 field set to 0, that field would be considered unspecified even though it was present on the wire. If that field is marked `+k8s:required`, the message would be rejected."*

| Kind | Expectation |
|---|---|
| **int** | `+k8s:minimum` and/or `+k8s:maximum`. A `required` int may not hold zero unless it also has `+default=0` (note: different syntax, no `k8s:` prefix). |
| **string** | a `+k8s:format=…` or a length bound. Formats in use: `k8s-short-name` (DNS-1123 label), `k8s-long-name`, `k8s-uuid`, `k8s-label-key`, `k8s-label-value`. |
| **struct** | nothing extra if the type's own tags suffice |
| **repeated** | `+k8s:maxItems`, a `+k8s:listType`, and per-item validation with `+k8s:eachVal` |
| **map** | `+k8s:maxProperties`, plus `+k8s:eachKey` / `+k8s:eachVal` |

**Be strict.** *"It is easier to loosen validation than to tighten it. If you are unsure about a validation rule, make it strict."*

### List types

| `+k8s:listType` | Meaning |
|---|---|
| `set` | set-like; no duplicate items |
| `map` + `+k8s:listMapKey=<field>` | map-like; the key field identifies an item |
| `atomic` | replaced wholesale; no per-item identity |

You will see all three in `ateapi.proto`, e.g.:

```proto
// +k8s:maxItems=16
// +k8s:listType=atomic
// +k8s:unique=map
// +k8s:listMapKey=generation
repeated Snapshot snapshots = 11;

// +k8s:maxItems=256
// +k8s:listType=set
// +k8s:customValidation # format
repeated string hostnames = 1;

// +k8s:maxProperties=10
// +k8s:eachKey=+k8s:format=k8s-label-key
// +k8s:eachVal=+k8s:format=k8s-label-value
map<string, string> match_labels = 1;
```

### Update tags

| Tag | Meaning |
|---|---|
| `+k8s:immutable` | set once at creation, fixed thereafter. An optional immutable field left unset at create may never be set later. |
| `+k8s:update=NoSet` | may not go unspecified → specified; may otherwise change |
| `+k8s:update=NoUnset` | may not go specified → unspecified; may otherwise change |
| `+k8s:update=NoModify` | may not change value-to-value; **may be set or cleared** |
| `+k8s:update=NoAddItem` | items may be removed or modified, not added |
| `+k8s:update=NoRemoveItem` | items may be added or modified, not removed |

`+k8s:update` may be specified more than once. Real examples:

```proto
// +k8s:update=NoModify   # can be set and cleared, but not changed in place
WorkerAssignment worker_assignment = 2;

// +k8s:update=NoModify
// +k8s:update=NoUnset    # set-once; immutable would reject the create ratchet's
                          # nil->set when the final object is validated as an update
string name = 1;
```

That second comment is the pattern for a **set-once** field: `immutable` is the wrong tag when the object is validated as an update after a nil→set transition.

**Unchanged values are not re-validated on update** — which is what lets an object created while a preview gate was on stay usable after the gate is turned off.

### Subfields

A shared type's internal rules are often looser than a given use site wants:

```proto
message ObjectRef {
  // +k8s:optional
  string atespace = 1;
  // +k8s:required
  string name = 2;
}

// at an atespaced use site:
// +k8s:required
// +k8s:subfield(atespace)=+k8s:required
ObjectRef actor = 1;

// at a global use site:
// +k8s:required
// +k8s:subfield(atespace)=+k8s:forbidden
ObjectRef worker = 6;
```

**Subfields may not be nested** (planned for a future release). For deeper needs, use `+k8s:customValidation`.

### Custom validation

```proto
message Foo {
  // +k8s:optional
  // +k8s:customValidation  # say WHAT is being validated — this comment is required by convention
  string bar = 1;
}
```

`validation-gen` then looks for:

```go
func ValidateCustom_Foo_Bar(ctx context.Context, op operation.Operation,
    fldPath *field.Path, val, oldVal <type>) field.ErrorList
```

It receives a pointer to the struct (struct field), a pointer to the scalar (scalar field), or a slice/map (repeated/map field). **Every `+k8s:customValidation` must be commented in the proto saying what it checks** — and in practice the proto does this consistently (`# must be a well-formed image reference, pinned by digest`, `# no control characters`, `# a SPIFFE ID`, `# cpu and memory limits are both set`, `# no two rules tie on a name and port`).

### `+k8s:opaqueType` — the big hammer

Cancels the validation rules attached to a type at one use site. *"This is a big hammer, use it with care."*

## Preview gating in the API

```proto
// +ate:preview=MyFeature
// +k8s:ifDisabled(MyFeature)=+k8s:forbidden
// +k8s:optional
MyFeatureConfig my_feature = 7;
```

`+ate:preview=<Gate>` is **documentation only** — it enforces nothing. `+k8s:ifDisabled(<Gate>)=+k8s:forbidden` is what rejects the field when the gate is off for the request.

**The mechanism only works if the validator is built correctly.** The gate state reaches validators through `apivalidation.MakeCreateOp(ctx)` / `MakeUpdateOp(ctx)`:

> *"Every `Validate*` wrapper **must** build its `operation.Operation` with one of these, not with a bare `operation.Operation{...}` literal; otherwise the field is never gated."*

**Never gate a field the server writes**, such as `status`: the server writes it while doing system work (deleting an actor, for instance), and gating it can leave objects that cannot be cleaned up after the gate is turned off. See [preview-gates-and-compat.md](preview-gates-and-compat.md).

## Where validation lives

| Package | Validates |
|---|---|
| `cmd/ateapi/internal/apivalidation` | the Control + WorkerService RPC surface (generated code lives with the handlers) |
| `cmd/atelet/internal/apivalidation` | the Atelet RPC surface |
| `internal/ateworkervalidation` | the Worker RPC surface — **a separate package** because both worker binaries implement the service in package `main`, and generated code can't be shared from there |
| `pkg/api/v1alpha1/*_validation_test.go` | the CRDs, via **CEL `+kubebuilder:validation:XValidation`** rules, not DV |
| `cmd/ateapi/internal/controlapi/validate.go` | the handful of hand-written checks |

Note the two different systems: **gRPC resources use `validation-gen` + DV tags; CRDs use kubebuilder markers + CEL.** `.golangci-kal.yaml` runs `kube-api-linter` over the CRD types (`hack/verify/kube-api-linter.sh`).

Error construction goes through `internal/resources.ToAPIError(field.ErrorList)` → an `apierror`, which is deliberately **not** a gRPC status (no `GRPCStatus` method) so an accidental error cannot masquerade as an intended one.

## Code generation

```sh
hack/update/codegen.sh       # regenerate everything
hack/verify/codegen.sh       # CI gate: fails if the tree is stale
hack/verify-all.sh           # runs every verifier
```

What is generated:

| From | To |
|---|---|
| `*.proto` | `*.pb.go`, `*_grpc.pb.go` (`hack/protoc.sh` pins protoc) |
| proto comments + DV tags | the `apivalidation` / `ateworkervalidation` packages |
| `pkg/api/v1alpha1/*_types.go` | `zz_generated.deepcopy.go`, CRD YAML |
| `pkg/api/v1alpha1` | `pkg/client/{clientset,informers,listers}` (standard Kubernetes code-generator) |
| `pkg/api/v1alpha1/gen.go` | the markers that drive the above |
| Python protos | `hack/verify/python-protos.sh` (the benchmarking harness) |

Other verifiers worth knowing: `hack/verify/proto-fmt.sh`, `hack/verify/metrics.sh` (the Weaver registry), `hack/verify/postgresql-migrations.sh`, `hack/verify/ateapi-deps.sh` (guards what `cmd/ateapi` may import), `hack/verify/threats.sh` (`docs/threats.json` ↔ `docs/threat-model.md`), `hack/verify/licenses.sh`, `hack/verify/boilerplate.sh`, `hack/verify/shellcheck.sh`, `hack/verify/kube-api-linter.sh`, `hack/verify/golangci-lint.sh`.

## Adding a field: the checklist

1. **Is it user-set or server-set?** Server-set → it belongs under `status`, and **must not** be preview-gated.
2. Add it to the `.proto` with a real doc comment plus `+k8s:required`/`+k8s:optional` and the bounds for its kind.
3. Immutable or set-once? Add `+k8s:immutable`, or `+k8s:update=NoModify` + `+k8s:update=NoUnset`.
4. Preview feature? Add `+ate:preview=<Gate>` **and** `+k8s:ifDisabled(<Gate>)=+k8s:forbidden`.
5. Holds a secret? Add `[ debug_redact = true ]` **in the same change**.
6. Enum field? Add `+k8s:minimum=1` and `+k8s:maximum=<N>` **and** the `# keep this in sync with the <Enum> enum` comment. Also add the mirrored comment on the enum's last value. *(This hand-maintained pair is all over the proto — it is the convention, follow it.)*
7. New resource, or a re-parent? Update the `(resource)` option **and `docs/resource-model.md`**.
8. New RPC? Add `(method).resource`, **and an entry in `defaultRPCPermissions`** (`cmd/ateapi/internal/authz/registry.go`) — a served RPC without one **fails a test and is denied at runtime**.
9. `hack/update/codegen.sh`, then `go test -race ./...` and `hack/verify-all.sh`.
10. **Before v1.0.0, do not add a compatibility shim.** `AGENTS.md`: remove or rename fields and RPCs outright and update every caller in the same change. No proto `reserved`, no deprecated aliases.

## Where to read more

- `docs/api-style-guide.md` — the authority. All 12 sections are normative.
- `docs/api-validation.md` — the authority for DV.
- `docs/resource-model.md` — keep in sync with the `(resource)` options.
- `docs/code-style-guide.md`, `docs/dev/best-practices/logging.md#logging-protos-that-hold-secrets`.
- `pkg/proto/ateapipb/ateapi.proto` — the best worked example of all of the above.

# ADR 0027 — Cross-repo proto contract, drift control, and wire evolution

**Status**: Accepted (2026-08-20)

## Context

[ADR 0004](0004-per-domain-repo-topology.md) makes the `server` repo the home of
wire contracts. In practice there are two hand-synchronized copies of the same
schema:

- `server/proto/sarnaut/v1/{common,handshake,movement,replication}.proto`, with
  generated Go **committed** under `server/gen/sarnaut/v1/`. Server CI regenerates
  with protoc **35.1** and `protoc-gen-go` **v1.36.12**, then runs
  `git diff --exit-code -- gen`, so server-side drift between schema and generated
  code cannot merge.
- `client/src/SarnautCore.Network/Proto/sarnaut/v1/*.proto`, with **no** committed
  generated code. `SarnautCore.Network.csproj` uses `Grpc.Tools` **2.83.0** with
  `<Protobuf Include="Proto\sarnaut\v1\*.proto" ProtoRoot="Proto" GrpcServices="None" />`
  and generates at build time.

The asymmetry is the problem. Edit a server proto without touching the client copy
and both repos still build green: the client compiles cleanly against its stale
schema and the mismatch surfaces as a silent mis-parse against a live shard. Nothing
in either CI compares the two trees.

Content is the same class of hazard one level up. Two peers can agree perfectly on
message shape and still disagree on what an item id or spell id means, because they
loaded different compiled data.

## Decision

### Canonical location and direction of flow

`server/proto/sarnaut/v1/` is canonical. Every wire change is a `server` pull
request first. The client tree is a **generated copy**, never an edit site; the
files carry a leading comment saying so.

### PROTO_LOCK.sha256, committed in both repos

A lock file is committed at `server/proto/PROTO_LOCK.sha256` and at
`client/src/SarnautCore.Network/Proto/PROTO_LOCK.sha256`. The two files are
**byte-identical**. Format is plain `sha256sum` output over the `.proto` set,
relative to the proto root, sorted bytewise by path under `LC_ALL=C`, LF line
endings:

```
<64 hex>  sarnaut/v1/common.proto
<64 hex>  sarnaut/v1/envelope.proto
<64 hex>  sarnaut/v1/handshake.proto
<64 hex>  sarnaut/v1/movement.proto
<64 hex>  sarnaut/v1/replication.proto
```

- Written by a new `server/scripts/proto-lock.ps1`, invoked from the `generate`
  target in `server/Makefile` so `make generate` refreshes lock and `gen/` together.
- The server CI step "Check generated protobuf code" widens its assertion to
  `git diff --exit-code -- gen proto/PROTO_LOCK.sha256`.
- The client CI job adds a step that recomputes digests over
  `src/SarnautCore.Network/Proto/sarnaut/v1/*.proto` and compares them to the
  committed lock, which catches a hand-edited client proto.

### Prevention, not only detection

Detection alone leaves a window where the two trees differ and someone has to
reconcile them by hand. A new `client/scripts/sync-proto.ps1` removes the window:

- Parameters `-ServerRepo <path>` (default: the sibling `../server` checkout) and
  `-Check`.
- Copies `<ServerRepo>/proto/sarnaut/v1/*.proto` into
  `src/SarnautCore.Network/Proto/sarnaut/v1/`, **deletes** client-side protos that
  no longer exist upstream, and copies `PROTO_LOCK.sha256` alongside them.
- With `-Check` it writes nothing and exits non-zero if any copy would have changed
  a byte.

Client CI runs it in `-Check` mode against a second `actions/checkout` of
`SarnautCore/server` at `main` into `${{ runner.temp }}/server`. A client build is
green only if its proto tree is byte-identical to the canonical tree on server
`main`. The committed lock stays useful independently: it makes a mismatch
diagnosable offline and in a release artifact, where the server checkout is not
available.

Rejected alternatives for the same job: a **git submodule** of `server` in `client`
(submodule pins drift silently and quietly, and it drags the whole Go tree into a
Godot checkout that globs `Proto\sarnaut\v1\*.proto` from inside the project
directory); and a **published NuGet package of pre-generated C#** (adds a release
step to every wire change and one more version to bump against
[ADR 0008](0008-latest-stable-toolchain-policy.md)'s eager-upgrade cadence, for no
gain over a file copy).

### How ProtocolVersion evolves

`ProtocolVersion` in `sarnaut/v1/common.proto` is the single compatibility knob.

- One new enum value per **breaking** wire change: `PROTOCOL_VERSION_2 = 2`, and so
  on. A build speaks exactly the highest value defined in its schema.
- Both ends compare for **exact equality** — already the behavior in
  `Server.exchangeHello`, `Client.Handshake`, and `GameSession.ConnectAsync` — and
  both ends additionally reject `PROTOCOL_VERSION_UNSPECIFIED`, so a
  default-constructed hello never matches another default-constructed hello.
- **Additive** changes do not bump it: a new oneof case in
  [ADR 0026](0026-wire-message-envelope.md)'s envelope, a new optional field, a new
  enum value in a payload message. Receivers reject unknown oneof cases (ADR 0026),
  so an additive change is only visible to a peer that asks for it.
- **Breaking** changes do bump it: removing or renumbering a field, changing a
  field's type, and any change to the meaning of an existing field while its number
  and type stay the same. The last case is the dangerous one and is explicitly in
  scope.
- Removed enum values are `reserved` and never reused.
- The proto package stays `sarnaut.v1`. A `sarnaut/v2/` directory is reserved for
  the case where the message set is incompatible in a way an enum value cannot
  express; it is a deliberate repo-level event, not routine maintenance.

### Content identity in the handshake

`ClientHello` and `ServerHello` each gain `string pack_id = 3;`. The value is the
manifest `pack_id` from [ADR 0029](0029-runtime-pack-format.md): the lowercase hex
BLAKE3-256 digest, 64 characters, of the compiled runtime pack the peer loaded.

`Server.exchangeHello` checks `protocol_version` first, then `pack_id`. On mismatch
it logs at error level with both digests and the target zone, writes its own
`ServerHello` so the client can display the server's digest, writes a
`ServerMessage{error: PACK_MISMATCH}`, and closes — it never proceeds to
`EnterZoneRequest`. `GameSession.ConnectAsync` performs the symmetric check right
after it reads `ServerHello`, before it sends `EnterZoneRequest`, and throws.

A pack mismatch is a **clear logged rejection, not a mis-parse**: identical message
shapes plus different content tables produce plausible-looking but wrong gameplay,
which is far more expensive to diagnose than a refused connection. An empty client
`pack_id` is accepted only when the shard config sets
`content.allow_unverified_pack: true`, which defaults to false.

## Consequences

- A wire change touches three things in lockstep: the canonical `.proto`, the lock
  file, and the client copy produced by `sync-proto.ps1`. Reviewers see all three or
  the change is incomplete.
- Client CI gains a cross-repo checkout and therefore depends on `server` staying
  public, which ADR 0004 already commits to.
- Shipping a client build against a shard on a different content pack now fails at
  connect time with a named error instead of drifting.
- Local development without a sibling `server` checkout still works; only the
  `-Check` path needs one.

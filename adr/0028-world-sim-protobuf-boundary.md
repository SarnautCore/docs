# ADR 0028 — The world sim owns domain types; protobuf stops at the session layer

**Status**: Accepted (2026-08-20)

## Context

[ADR 0010](0010-protocol-posture.md) states that gateway and session abstractions
must not leak protobuf specifics into the world sim, because a retail-1.1 front-end
has to be addable later without surgery on the simulation.

`server/internal/world/world.go` does not honor that today. It imports
`sarnautv1 "github.com/SarnautCore/server/gen/sarnaut/v1"` and uses generated types
throughout its public surface and its state:

- `SnapshotSink.OfferSnapshot(*sarnautv1.SnapshotBatch)` — the replication boundary.
- `Zone.ApplyMoveIntent(entityID uint64, intent *sarnautv1.ClientMoveIntent)` — the
  input boundary.
- `entity.kind` and `entity.animation` are `sarnautv1.EntityKind` and
  `sarnautv1.AnimationState`, so persistent simulation state is typed by the wire
  schema.
- `interestedEntitiesLocked` returns `[]*sarnautv1.EntitySnapshot`, and
  `protobufVec` exists purely to bridge `world.Vec3` into the generated type.

Two costs follow. A second wire format would have to either translate into the first
one's generated types or fork the zone, which is exactly the surgery ADR 0010 rules
out. And [ADR 0026](0026-wire-message-envelope.md)'s snapshot-sharing hazard lives on
this boundary: `publishSnapshot` shares one generated message pointer across sinks
because the boundary type is a mutable protobuf message rather than a value.

## Decision

### `internal/world` exposes domain types only

After migration, `internal/world` imports no generated protobuf package. The
observable test is `grep -c sarnautv1 server/internal/world/*.go` returning zero, and
the enforcement is a `depguard` rule in `server/.golangci.yml` denying
`github.com/SarnautCore/server/gen/...` from `internal/world`, run by the existing
golangci-lint CI step (action `golangci/golangci-lint-action@v9`, linter v2.13.0).

New domain types in `internal/world`:

- `type EntityKind uint8` with `EntityKindUnspecified`, `EntityKindPlayer`,
  `EntityKindNPC`.
- `type AnimationState uint8` with `AnimationStateIdle`, `AnimationStateMoving`.
- `type MoveIntent struct { Seq uint64; Input Vec3; Heading float32; Duration
  time.Duration }`. `Duration`, not `DtSeconds`: clamping against
  `maxIntentDuration` is a simulation invariant, while float seconds is a wire
  encoding choice.
- `type EntitySnapshot struct { EntityID uint64; Kind EntityKind; Position Vec3;
  Heading float32; Velocity Vec3; Animation AnimationState }`.
- `type Snapshot struct { ServerTick uint64; Entities []EntitySnapshot }` — a value
  with a value slice, documented immutable once published.

Boundary signatures become `SnapshotSink.OfferSnapshot(Snapshot)` and
`Zone.ApplyMoveIntent(entityID uint64, intent MoveIntent) error`. Validation stays in
`world` where it is a domain invariant (non-finite rejection, sequence-number
ordering, speed clamping); decoding and range-checking of wire bytes stays in
`session`.

### All protobuf mapping lives in `internal/session`

One new file, `server/internal/session/mapping.go`, holds the entire translation
layer and nothing else:

- `func moveIntentFromProto(*sarnautv1.ClientMoveIntent) (world.MoveIntent, error)`
- `func snapshotToProto(world.Snapshot) *sarnautv1.SnapshotBatch`
- the two enum tables, each with an exhaustive default case that returns an error
  rather than a zero value.

`internal/session` becomes the only package above the transport layer that imports
`gen/sarnaut/v1`. `internal/transport` keeps its `proto.Message` dependency in
`framing.go`: it is the wire layer, and the rule under discussion is about the
simulation, not about serialization plumbing.

`snapshotSender.queue` becomes `chan world.Snapshot`, and `snapshotSender.send` maps
to protobuf immediately before chunking. That allocates the protobuf message once per
session per publish, which retires ADR 0026's shared-batch hazard by construction
rather than by review discipline.

### Scope: it happens inside the combat skeleton

This migration is executed **as part of the M2 combat-skeleton task** (Linear, M2
Vertical Slice project, `server` label per
[ADR 0023](0023-linear-structure.md)), not as a standalone refactor and not deferred
past it. The combat skeleton adds ability activation and combat events to the same
two boundaries; doing it in the other order means writing new protobuf-typed world
APIs and then rewriting them within the same milestone. The gate is concrete: the
combat-skeleton pull request is not mergeable while `internal/world` imports `gen/`,
and it carries the `depguard` rule that makes that permanent.

### Alternatives considered and rejected

- **Leave the generated types in `world` and rely on review.** Rejected: it is the
  status quo, ADR 0010 already forbids it in prose, and prose did not hold. It also
  makes the simulation's persistent state depend on
  [ADR 0027](0027-proto-contract-and-wire-evolution.md)'s wire evolution rules for
  no reason.
- **A third `internal/domain` package shared by `world` and `session`.** Rejected: a
  package with types and no behavior, which drains `world` of the invariants that
  justify its mutex. The types belong to the module that enforces them.
- **Generate the world types from the protos with a code generator.** Rejected: it
  inverts the dependency, making the network schema the source of truth for
  simulation state, which is the failure this ADR exists to end.

## Consequences

- `internal/world/world_test.go` and `internal/session/snapshot_test.go` stop
  constructing generated messages; mapping gets its own table-driven test that is the
  only place enum coverage is asserted.
- Adding a wire field that the simulation does not model becomes a one-file change in
  `mapping.go`.
- A retail-protocol front-end (ADR 0010) becomes a second mapping file next to
  `mapping.go`, with `internal/world` untouched — which is the property ADR 0010
  asked for.

## Amendments

### 2026-08-20 — golangci-lint moves to v2.13.0

Go 1.27 requires a linter binary built with Go 1.27 or newer. The v2.12.2
release was built with Go 1.26 and rejects modules targeting Go 1.27. Version
v2.13.0 is the first golangci-lint release with Go 1.27 support, so the server
CI pin and the version recorded above move together. The depguard decision is
unchanged.

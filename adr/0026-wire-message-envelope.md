# ADR 0026 — Wire message envelope, channel assignment, and snapshot ownership

**Status**: Accepted (2026-08-20)

## Context

After `EnterZoneResponse` the post-handshake stream is **positionally typed**. The
server loop in `server/internal/session/handshake.go` calls `receiveMoveIntent`
forever and always parses a `ClientMoveIntent`; the client's
`GameSession.ReadSnapshotAsync` in `client/src/SarnautCore.Network/GameSession.cs`
always parses with `SnapshotBatch.Parser`. Nothing on the wire says what a frame is.

Protobuf makes that failure silent rather than loud: an unexpected message decodes
against whatever field numbers happen to overlap and the rest lands in unknown
fields. A wrong-type frame is **mis-parsed, not rejected**. That is survivable while
exactly one message class travels in each direction. M2 adds five more client
classes and seven more server classes, so it stops being survivable.

Two further gaps are visible in the same code. `session.Server.handle()` stops
reading the reliable stream entirely once it enters its move-intent loop, so when
`transport.Connection.SupportsUnreliable()` reports true, **nobody reads the
stream**. And `world.Zone.publishSnapshot` (`server/internal/world/world.go`, lines
224-238) builds one `*sarnautv1.SnapshotBatch` under the zone mutex declared at line
58 and hands that same pointer to every subscribed sink. It is race-free today only
because no sink mutates it.

## Decision

### One envelope per direction

A new file `server/proto/sarnaut/v1/envelope.proto` (package `sarnaut.v1`,
`go_package` `github.com/SarnautCore/server/gen/sarnaut/v1;sarnautv1`), mirrored to
`client/src/SarnautCore.Network/Proto/sarnaut/v1/envelope.proto` by the sync tooling
in [ADR 0027](0027-proto-contract-and-wire-evolution.md). Every post-handshake frame
in either direction is one of these two messages and nothing else:

```proto
message ClientMessage {
  uint64 client_seq = 1;
  oneof payload {
    ClientMoveIntent move_intent   = 10;
    AbilityUse       ability_use   = 11;
    Interact         interact      = 12;
    LootTake         loot_take     = 13;
    QuestAccept      quest_accept  = 14;
    QuestTurnIn      quest_turn_in = 15;
  }
}

message ServerMessage {
  uint64 server_tick = 1;
  oneof payload {
    SnapshotBatch   snapshot_batch   = 10;
    CombatEvent     combat_event     = 11;
    DeathEvent      death_event      = 12;
    LootOffer       loot_offer       = 13;
    LootResult      loot_result      = 14;
    InventoryUpdate inventory_update = 15;
    QuestState      quest_state      = 16;
    Error           error            = 17;
  }
}
```

That case list is the complete M2 message set. Payload field numbers start at 10 so
envelope-level fields can grow in the 1-9 block. A retired case is `reserved`; case
numbers are never renumbered or reused.

`Error` is a typed message, not a string: `Error { ErrorCode code = 1; string detail
= 2; }` with `ErrorCode` covering `UNSPECIFIED`, `UNSUPPORTED_MESSAGE`,
`MALFORMED_PAYLOAD`, `UNAUTHENTICATED`, `NOT_IN_ZONE`, `RATE_LIMITED`,
`PACK_MISMATCH`, `INTERNAL`. `UNAUTHENTICATED` is the refusal for a ticket that is
expired, already burned, unknown, or blocked by the play lock: admission terminates
at the shard for M2 ([ADR 0030](0030-auth-account-service-session-tokens.md),
[ADR 0033](0033-m2-module-topology-gateway-deviation.md)), so the envelope needs a
code for it and `INTERNAL` would misreport a client-side condition as a server fault.
Because `ProtocolVersion` is compared for exact equality at both ends
(`Server.exchangeHello`, `Client.Handshake`, and `GameSession.ConnectAsync`), an
unset or unrecognized oneof case is a protocol violation, not a forward-compatibility
event: the server replies `ServerMessage{error: UNSUPPORTED_MESSAGE}` and closes the
connection; the client throws and disconnects.

Framing does not change. Both envelopes travel through
`transport.WriteMessage`/`transport.ReadMessage` with the existing four-byte
big-endian length prefix and `transport.MaxFrameSize` of 1 MiB.

### Channel assignment

Under [ADR 0017](0017-quic-protobuf-transport.md)'s reliable-stream versus datagram
split, exactly **two** oneof cases are eligible for datagrams:
`ClientMessage.move_intent` and `ServerMessage.snapshot_batch`. Both are
latest-value-wins state where staleness beats retransmission. Every other case —
including **all combat traffic** — travels on the reliable ordered stream:

- Reliable stream, client to server: `ability_use`, `interact`, `loot_take`,
  `quest_accept`, `quest_turn_in`.
- Reliable stream, server to client: `combat_event`, `death_event`, `loot_offer`,
  `loot_result`, `inventory_update`, `quest_state`, `error`.

A combat command is therefore a `ClientMessage.ability_use` on the reliable stream,
never a datagram. Dropping an ability activation is a gameplay bug; dropping a
movement sample is not.

A datagram carries a **whole envelope**, not a bare submessage, so both carriers
share one parser and one dispatch table. The datagram size cap stays
`transport.MaxUnreliableMessageSize` (1100 bytes) and `splitSnapshot` in
`internal/session` chunks against the size of the encoded `ServerMessage`, not the
size of the bare `SnapshotBatch`.

When `SupportsUnreliable()` is false, the identical envelope goes on the stream. The
carrier changes; the bytes and the dispatch do not.

### Who reads the reliable stream

`session.Server.handle()` becomes a supervisor that owns no I/O loop itself. Per
admitted session it starts three goroutines and waits:

1. **`readReliable`** — new function in a new file `server/internal/session/reader.go`.
   Loops `transport.ReadMessage` into a `*sarnautv1.ClientMessage` and dispatches by
   oneof case. **Always started**, whether or not datagrams were negotiated. This is
   the goroutine that closes the gap: today no one reads the stream when
   `SupportsUnreliable()` is true, so a reliable client frame would sit in the QUIC
   receive buffer until flow control stalled the connection.
2. **`readUnreliable`** — same file. Loops `connection.ReceiveUnreliable(ctx)` and
   dispatches the decoded envelope. **Started only when
   `connection.SupportsUnreliable()` is true.** When it runs, `move_intent` arrives
   here and `readReliable` rejects a `move_intent` on the stream with
   `UNSUPPORTED_MESSAGE`, so a client cannot pick its own carrier per frame.
3. **`snapshotSender.run`** — unchanged, drains the coalescing queue and writes.

Shutdown: `handle()` derives `sessionCtx, cancel := context.WithCancel(ctx)` and
gives every goroutine a buffered `chan error` of capacity three. It blocks on the
first value, then calls `cancel()` **and** `connection.Close()`, then drains the
remaining sends before returning. Both calls are needed: `ReceiveUnreliable` takes a
context and unblocks on cancellation, but `stream.Read` inside `readReliable` does
not observe a context and only unblocks when the QUIC stream closes. Because
`handle()` drains all three before returning, its deferred `zone.Leave(entityID)`
runs strictly after the last goroutine is gone, so no sink is used after
unsubscription.

### Snapshots are never shared mutable state

`world.Zone.publishSnapshot` must not hand one shared `*sarnautv1.SnapshotBatch` to
multiple sinks under any condition where a sink could write to it. Three rules:

1. **Immutable after publish.** Any value crossing the `world.SnapshotSink`
   boundary is read-only for the sink and for everything the sink hands it to. A
   sink may not set fields, may not call `proto.Merge` into it, and may not append
   to its `Entities` slice. `splitSnapshot` complies today because it allocates
   fresh batches and only copies existing `*EntitySnapshot` pointers into new
   slices.
2. **Per-sink content means per-sink values.** The moment a snapshot differs per
   recipient — SAR-19's spatial interest query, or per-session field masking —
   the shared-pointer form is illegal, and `publishSnapshot` builds one value per
   sink inside the same loop that walks `zone.sessions`. Sharing is an
   optimization that is valid only while every recipient gets identical content.
3. **The type at the boundary changes.**
   [ADR 0028](0028-world-sim-protobuf-boundary.md) replaces the protobuf pointer
   with a domain `world.Snapshot` value, and the protobuf message is then built
   inside `snapshotSender.send`, once per session. After that migration the hazard
   is gone by construction rather than by convention.

The invariant is documented on the `SnapshotSink` interface and covered by a
`go test -race` case in `internal/world` that subscribes two sinks and asserts both
observe the same content for the same tick.

### Alternatives considered and rejected

- **One QUIC stream per message class.** Rejected: `transport.Connection` is a
  single `io.Reader`/`io.Writer`, and stream identity would leak QUIC concepts
  through the seam ADR 0017 declares a hard boundary. It also forces a stream-accept
  loop per class into the .NET client. Revisit only if head-of-line blocking is
  measured, not assumed.
- **A `uint16` type tag ahead of the protobuf frame.** Rejected: the tag registry
  would be maintained by hand in the `.proto`, in Go, and in C#, with no generated
  exhaustiveness check in any of the three and nothing generated at all on the
  Grpc.Tools side.
- **`google.protobuf.Any`.** Rejected: a type URL string on every frame, no closed
  set to validate against, and reflective resolution in both runtimes.

## Consequences

- `envelope.proto` lands before any combat work; the combat skeleton adds cases to
  an existing oneof rather than inventing a second framing.
- The proto lock and the client proto copy change in the same pull request
  (ADR 0027).
- `splitSnapshot` is re-measured against envelope size, which slightly reduces
  entities per datagram.
- `client/tools/SarnautCore.NetSmoke` and `server/scripts/sar20-client-smoke.ps1`
  move to the envelope in the same change; there is no transition period in which
  both framings are accepted.

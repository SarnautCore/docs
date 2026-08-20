# Spec 0004 — Session lifecycle (M2 vertical slice)

**Status**: Draft
**Milestone**: M2
**Owner**: netcode
**Last reviewed**: 2026-08-20

## 1. Scope

Covers one player's connection from the first packet to the last: connect, admission by
ticket, `EnterZone`, the command and event loop, and disconnect. Character listing,
creation and selection happen against the auth service before the connection exists
(§5.3); this spec covers what the shard does with the result. It also fixes where character **load** and **save** checkpoints attach,
which is the part most likely to be got wrong, because the zone teardown already runs
through a `defer` on every error path.

This spec describes SarnautCore's own protocol. Per ADR 0010 the wire format is clean
— it is not a reconstruction of the retail protocol — so nothing here is derived from
reference data and there are no retail values to cite. Every constant in §3 is either
a value already committed in the SarnautCore server repository or a curated decision.

Out of scope: the QUIC transport itself and its certificate handling (ADR 0017 puts
both behind the transport seam; game code never sees QUIC types), replication interest
management, the gateway's routing between shards, and account creation. Combat, loot,
and quest payload semantics belong to the specs under
[`../mechanics/`](../mechanics/); this spec only says which channel carries them and
in what order.

## 2. Vocabulary

| Term | Meaning |
|---|---|
| **Connection** | One transport-level connection. Owned by `transport.Connection`, which abstracts QUIC. |
| **Session** | One authenticated account bound to one connection. Outlives no connection. |
| **Ticket** | The opaque, single-use `sarnaut_tk_…` credential minted by auth for one `(account_id, character_id)` pair and redeemed by the shard (ADR 0030). |
| **Character** | A persistent record. One session drives at most one character at a time. |
| **World entity** | The zone's in-memory representation of the character. Created by `Zone.Join`, destroyed by `Zone.Leave`. Its `entity_id` is per-zone-visit and is **not** stable across reconnects. |
| **Reliable channel** | An ordered QUIC stream. Carries the handshake, commands, and events. |
| **Unreliable channel** | QUIC datagrams. Carries move intents up and snapshots down. |
| **Checkpoint** | A point at which character state is written to durable storage. |

## 3. Constants

Prefixes as in [`../mechanics/combat.md`](../mechanics/combat.md) §3.

| Constant | Value | Unit | Provenance |
|---|---|---|---|
| `PROTOCOL_VERSION` | `PROTOCOL_VERSION_1` | enum | `SERVER:proto/sarnaut/v1/common.proto` — `ProtocolVersion` |
| `MAX_UNRELIABLE_MESSAGE_SIZE` | 1100 | bytes | `SERVER:internal/transport/framing.go` — `MaxUnreliableMessageSize`, sized to stay under the common QUIC path limit |
| `MAX_INTENT_DURATION_MS` | 250 | milliseconds | `SERVER:internal/world/world.go` — `maxIntentDuration`; caps how far one move intent may integrate |
| `TICK_INTERVAL_MS` | 33.333333 | milliseconds | `SERVER:config.example.yaml` — `world.tick_interval` |
| `SNAPSHOT_INTERVAL_MS` | 66.666666 | milliseconds | `SERVER:config.example.yaml` — `world.snapshot_interval` |
| `HANDSHAKE_TIMEOUT_S` | 10 | seconds | curated SarnautCore decision — an unauthenticated connection must not hold a slot indefinitely |
| `SHARD_TICKET_TTL_S` | 60 | seconds | curated SarnautCore decision, fixed by [ADR 0030](../../adr/0030-auth-account-service-session-tokens.md) §2 — the auth→shard ticket is opaque, single-use, and burned on redemption |
| `SAVE_TIMEOUT_S` | 5 | seconds | curated SarnautCore decision — bounds the disconnect-path save (rule 5.7.5) |
| `PERIODIC_SAVE_INTERVAL_S` | 60 | seconds | curated SarnautCore decision — bounds worst-case progress loss on an unclean shard exit |

## 4. Model

```
SessionState = enum {
  Connected,      // transport up, nothing verified
  Versioned,      // hello exchange complete, content pack agreed
  Authenticated,  // ticket redeemed; account and character identified
  CharacterBound, // the character record is loaded, not yet in a zone
  InZone,         // world entity exists, snapshots flowing
  Draining,       // teardown started; no new commands accepted
}

Session
  connection      transport.Connection
  account_id      uuid.UUID   // UUIDv7, minted by auth (ADR 0030); zero until Authenticated
  character_id    uuid.UUID   // UUIDv7; zero until Authenticated
  entity_id       uint64      // 0 until InZone; not stable across visits
  zone_id         string
  state           SessionState

CharacterSnapshot            // what a checkpoint writes
  character_id    uuid.UUID
  zone_id         string
  position        Vec3
  heading         float32
  level           uint8
  current_hp      int32
  inventory       []Stack
  quest_instances []QuestInstance
  save_seq        uint64      // monotonic per character; the stale-write guard (ADR 0031 §6)
  saved_at_tick   uint64      // diagnostic only; resets with the process
```

Message families on the reliable channel: `ClientHello` / `ServerHello` and
`EnterZoneRequest` / `EnterZoneResponse`
(`SERVER:proto/sarnaut/v1/handshake.proto`), then the `ClientMessage` and
`ServerMessage` envelopes of
[ADR 0026](../../adr/0026-wire-message-envelope.md)
(`SERVER:proto/sarnaut/v1/envelope.proto`, rule 5.5). Every post-handshake frame in
either direction is one of those two envelopes and nothing else, on either carrier:
a datagram carries a whole envelope, not a bare submessage. On the unreliable
channel the eligible cases are exactly `ClientMessage.move_intent` up and
`ServerMessage.snapshot_batch` down (`SERVER:proto/sarnaut/v1/movement.proto`,
`.../replication.proto`).

## 5. Rules

### 5.1 Connect and version handshake

1. The client opens a connection to the shard and sends
   `ClientHello{protocol_version, build_id, pack_id}` on the reliable channel as the
   first message. Nothing else may precede it.
2. The server rejects and closes if `protocol_version != PROTOCOL_VERSION`. The check
   is exact equality, not a minimum: there is no negotiation and no compatibility
   window in M2. `PROTOCOL_VERSION_UNSPECIFIED` is rejected by both ends, so two
   default-constructed hellos never match each other
   ([ADR 0027](../../adr/0027-proto-contract-and-wire-evolution.md)).
3. On success the server replies `ServerHello{protocol_version, build_id, pack_id}`.
   The client applies the same equality check to the response, so a mismatch is caught
   on both ends rather than leaving one side believing the session is healthy.
4. `pack_id` **is** a gate, checked after `protocol_version`. It is the manifest digest
   of the compiled runtime pack each peer loaded
   ([ADR 0029](../../adr/0029-runtime-pack-format.md)). On mismatch the server logs
   both digests, still writes its own `ServerHello` so the client can display the
   server's digest, writes `ServerMessage{error: PACK_MISMATCH}`, and closes without
   ever accepting an `EnterZoneRequest`; the client performs the symmetric check on
   the `ServerHello` before it sends one. Two peers that agree on message shape and
   disagree on content tables produce plausible-looking wrong gameplay, which is far
   more expensive to diagnose than a refused connection. An empty client `pack_id` is
   accepted only under `content.allow_unverified_pack: true`, default false.
5. `build_id` is carried for diagnostics and is **not** a gate. Mismatched builds are
   logged, not refused; ADR 0008's latest-stable policy means development builds drift
   constantly and refusing on drift would make local testing miserable. `pack_id`
   differs precisely because it is a statement about content, not about binaries.
6. The connection moves to `Versioned`. A connection that has not reached
   `Authenticated` within `HANDSHAKE_TIMEOUT_S` of accept is closed.

### 5.2 Authenticate

The shard does not verify credentials. ADR 0016 puts that in a separate auth service
and [ADR 0030](../../adr/0030-auth-account-service-session-tokens.md) fixes the
mechanism.

1. Before it connects, the client authenticates against the auth service's HTTP API,
   receives an opaque account session token (`sarnaut_as_…`, 12 h), and asks for a
   shard ticket for one character. The ticket is opaque, 32 random bytes, minted for
   one `(account_id, character_id)` pair, and lives `SHARD_TICKET_TTL_S`.
2. The client presents the ticket in `EnterZoneRequest{zone_id, ticket}` (rule 5.4.1).
   There is **no separate authenticate message** on the game connection: the ticket
   already names the character as well as the account, so a second round trip would
   carry nothing new. [ADR 0033](../../adr/0033-m2-module-topology-gateway-deviation.md)
   records why redemption terminates at the shard rather than at ADR 0016's gateway,
   and what moving it later costs.
3. The shard redeems it over NATS `sarnaut.auth.v1.ticket.redeem`, 2 s timeout, **no
   retry** — redemption is a single `GETDEL`, so a retried request cannot succeed and
   would only turn one failed login into two. The reply carries `account_id`,
   `character_id`, `character_name` and `chargen_option_id`, and that reply is the
   only place the shard learns them; it holds no credentials for the `auth` schema.
   There is nothing to verify locally: the token is opaque, not signed, so expiry,
   single use and the play lock are all decided by auth at redemption.
4. Failure — expired, already burned, unknown, or the play lock held elsewhere —
   closes the connection with `ServerMessage{error: UNAUTHENTICATED}`. There is no
   retry loop on the same connection: a client whose ticket was refused reconnects
   with a fresh one.
5. Success sets `account_id` and `character_id` and moves the session to
   `Authenticated`.
6. **Every subsequent command derives its actor from `session.account_id` and
   `session.character_id`, never from a field in the message.** The current move path
   already works this way — `handle` passes its own `entityID` to
   `zone.ApplyMoveIntent` and ignores anything the client might have put in the
   envelope — and rule 5.5.4 generalises it. This is the single rule that prevents
   every "act as another entity" exploit class, so it is stated here rather than
   repeated in each mechanics spec.

### 5.3 Character select happens out of band

1. Listing and creating characters are auth-service HTTP calls made by the launcher or
   the web UI **before the game connection exists** — `POST /v1/sessions`, then the
   account's character list, then `POST /v1/characters` for creation (ADR 0030,
   [ADR 0032](../../adr/0032-character-creation.md)). The shard serves no character
   list and no character-select message.
2. Having chosen, the client asks auth for a ticket bound to that `character_id`.
3. Ownership is therefore checked before a connection is opened, by the only process
   that can check it: auth is the sole holder of credentials for `auth.characters`
   (ADR 0030 §4). A ticket for a character the account does not own is never minted,
   so the shard has no `ErrNotYourCharacter` path to get wrong.
4. **One live session per character.** `ticket.redeem` takes the Valkey play lock
   `auth:playing:<character_id>` with `NX EX 60` and refuses redemption while a
   different holder has it; the shard renews it every 20 s and releases it on
   disconnect. That single-session guarantee is what makes the load and save
   checkpoints in §5.7 safe to write without cross-session merge logic, and the TTL is
   what keeps a shard that died without releasing from locking the character out
   permanently.
5. **Load checkpoint L1 does not run here.** With selection out of band, nothing is
   loaded until the ticket redeems; L1 runs inside `EnterZone` — see rules 5.4.4 and
   5.7.1.
6. The session reaches `CharacterBound` inside `EnterZone` (rule 5.4.4), not here.

### 5.4 EnterZone

1. The client sends `EnterZoneRequest{zone_id, ticket}`.
2. The server rejects if the zone is not hosted by this shard. In M2 a shard hosts a
   fixed zone set from configuration and there is no redirect and no routing hop: the
   client connects to the shard directly, because ADR 0033 keeps the gateway out of the
   M2 data path. The address comes from the launcher alongside the ticket.
3. The server rejects if the session is not `Versioned`, then redeems the ticket
   (rule 5.2.3) before anything else in this rule runs. A session that fails
   redemption never reaches `CharacterBound` and never touches the zone. Today's code
   reads `EnterZoneRequest` straight after the hello exchange
   (`SERVER:internal/session/handshake.go`) and admits any peer, because auth does not
   exist yet. Redemption inserts here; nothing in the existing `EnterZone` handling
   changes shape, it gains a precondition and a field.
4. **L1 runs here, then the world entity is created**, in this order:
   1. `charstore` loads the `CharacterSnapshot` for the redeemed `character_id` — this
      is checkpoint L1, rule 5.7.1. The session moves to `CharacterBound`.
   2. If no `shard.character_state` row exists, `charstore` materializes one from the
      `chargen` table — spawn, level, stats, `starting_loadout`, `starting_quests` — in
      the single transaction ADR 0031 requires, committed **before** the spawn is
      granted, and idempotent on `character_id` (ADR 0032 §5.4). A redeem that races or
      retries produces one character with one set of starting gear.
   3. The server creates the world entity. This is where the M2 change to `world.Zone`
      is required: `Zone.Join()` takes no arguments and always spawns at the zone's
      configured `PlayerSpawn`, which is correct for a fresh character and wrong for a
      returning one. `EnterZone` must call a `JoinAt(position, heading)` variant seeded
      from checkpoint L1, falling back to `PlayerSpawn` when the loaded record has no
      position for this zone.
   4. Checkpoint S0 stamps `zone_id` (rule 5.7.8).
5. **Teardown is armed immediately after the entity exists**, in the same statement
   sequence — see rule 5.6.1. Not after the response is written, and not after the
   subscription succeeds.
6. The server replies `EnterZoneResponse{zone_id, own_entity_id, spawn_position}`
   reliably. The client must treat `spawn_position` as authoritative and snap to it;
   it is the server's answer, not a confirmation of a client request.
7. Only after that response is written does the server subscribe the session to
   snapshots. The ordering matters: a snapshot that arrives before the client knows its
   own `entity_id` cannot be interpreted, and datagrams can overtake nothing but they
   can certainly arrive first if the subscription is opened early.
8. The session moves to `InZone`.

### 5.5 The command and event loop

1. Three loops run concurrently for the lifetime of `InZone`, supervised by the
   session handler, which owns no I/O itself (ADR 0026):
   - **`readReliable`**, reading `ClientMessage` envelopes off the ordered stream.
     It is **always** started, whether or not datagrams were negotiated — otherwise
     nobody reads the stream and a reliable frame sits in the receive buffer until
     flow control stalls the connection.
   - **`readUnreliable`**, started only when the transport reports datagram support.
   - the **send** path draining the snapshot queue, which today runs as its own
     goroutine whose error is selected on each loop iteration.
2. The unreliable channel carries exactly two envelope cases:
   `ClientMessage.move_intent` up and `ServerMessage.snapshot_batch` down. Movement and
   snapshots are the only traffic where a dropped packet is preferable to a
   retransmitted stale one (ADR 0017), and they are latest-value-wins. When datagrams
   are in use, a `move_intent` arriving on the stream is refused with
   `UNSUPPORTED_MESSAGE`, so a client cannot pick its own carrier per frame.
3. Everything else is a `ClientMessage` case on the reliable channel: `ability_use`,
   `interact`, `loot_take`, `quest_accept`, `quest_turn_in`, `quest_abandon` and
   `logout`. Replies and unsolicited notifications are `ServerMessage` cases on the
   same channel — `combat_event`, `death_event`, `loot_offer`, `loot_result`,
   `inventory_update`, `quest_state_update`, `error` — so ordering between a command's
   effect and the events it causes is preserved. **All combat traffic is reliable**: dropping an ability activation is a
   gameplay bug, dropping a movement sample is not. That case list is the complete M2
   message set (ADR 0026); a case not on it does not exist on the wire, and an
   unrecognized or unset oneof case is a protocol violation answered with
   `UNSUPPORTED_MESSAGE` and a close, not a forward-compatibility event.
4. Every command is validated in this order: session state, then actor derivation
   (rule 5.2.6), then the mechanics spec's own preconditions. A command arriving in
   `Draining` is dropped without a reply.
5. Move intents are ordered by `seq` and stale ones are discarded; an intent's
   integration window is clamped to `MAX_INTENT_DURATION_MS` so that a client claiming
   a large `dt_seconds` cannot teleport. Both behaviours already exist in
   `SERVER:internal/world/world.go` and are restated here because they are protocol
   guarantees, not implementation details.
6. Snapshots are produced on `SNAPSHOT_INTERVAL_MS`, independently of the
   `TICK_INTERVAL_MS` simulation, and a slow client's queue holds only the newest
   batch — an older undelivered batch is dropped rather than queued behind. Snapshot
   delivery is lossy by design.
7. A batch whose **encoded `ServerMessage`** exceeds `MAX_UNRELIABLE_MESSAGE_SIZE` is
   split into several batches carrying the same `server_tick`. The measurement is of
   the whole envelope, not of the bare `SnapshotBatch`, because a datagram carries an
   envelope (ADR 0026); measuring the payload alone would produce datagrams that are
   over the cap by exactly the envelope overhead. A client must therefore treat a batch
   as a partial view and merge by `server_tick`, never as a complete world state.
8. When the transport cannot do datagrams, the identical envelopes travel on the
   reliable stream. The carrier changes; the bytes and the dispatch do not. This is
   **not** a test-only path: per the 2026-08-20 amendment to
   [ADR 0017](../../adr/0017-quic-protobuf-transport.md), .NET 10's `System.Net.Quic`
   exposes no datagram API, so the shipped C# client runs the ordered-stream fallback
   for movement and snapshots and reports it in `GameSession.TransportMode`. The
   datagram path is exercised by the Go client, the Go integration tests, and the smoke
   client. Rule 5.5.6's lossy-by-design snapshot policy still holds on the fallback:
   the newest-only queue drops stale batches before they are written, rather than
   relying on the carrier to lose them.

### 5.6 Disconnect

1. Zone teardown is armed with a `defer` at the point the entity is created
   (`SERVER:internal/session/handshake.go` arms `zone.Leave(entityID)` immediately
   after `zone.Join()`). Consequently teardown runs on **every** return path from the
   session handler: a client that closed the connection, protocol error, transport
   error, decode failure, a failed subscription, and an error surfaced from any of the
   three loops in rule 5.5.1. There is no path out of the handler that skips it. The
   handler cancels the session context **and** closes the connection — `stream.Read`
   does not observe a context — and drains all three loops before the deferred teardown
   runs, so no sink is used after unsubscription (ADR 0026).
2. That property is the reason this spec puts the save checkpoint at the same site
   (rules 5.7.2 and 5.7.5) rather than at each `return`. Any scheme that saves on some returns is
   wrong by construction, because the set of returns grows every time a command is
   added.
3. Teardown removes the snapshot subscription and the world entity. The `entity_id` is
   not reused for this character and carries no meaning after this point.
4. **A clean exit is `ClientMessage.logout`** (2026-08-20 amendment to ADR 0026,
   §7.6). `readReliable` receives it, the handler returns, and the same teardown runs
   as on any other path — the verb changes when teardown starts, not what teardown
   does. The shard acknowledges nothing: a client that has already stopped reading
   would not see it, and the S1 checkpoint at rule 5.7.5 is what the client actually
   cares about, which happens after the last frame either way. Closing the connection
   without a `logout` remains legal and reaches identical teardown; what it loses is
   the guarantee that S1 begins before the transport goes away rather than racing it.
5. The session moves to `Draining` before teardown begins, so that a command already
   in flight is dropped by rule 5.5.4 rather than mutating state that is being
   persisted.

### 5.7 Character load and save checkpoints

This is the section to get right. There are exactly five checkpoints, and they are the
session-layer subset of the six save points ADR 0031 §5 enumerates; the two it does not
name here — character creation and shard shutdown — belong to `charstore` and to
`cmd/shard`, not to one connection's lifecycle.

1. **L1 — load, at zone entry, immediately after the ticket redeems** (rule 5.4.4.1).
   Reads the full `CharacterSnapshot`. It runs before the entity is created because
   `EnterZone` needs the loaded position to place it (rule 5.4.4.3), and before the
   `EnterZoneResponse` is written so that a load failure is still reportable while the
   session is healthy. A load failure answers `ServerMessage{error: INTERNAL}` and
   closes; the session never reaches `InZone`, and nothing was mutated.
2. **S1 — save, on the disconnect path** (rule 5.6.1). The important one.
3. **S2 — save, periodically while `InZone`**, every `PERIODIC_SAVE_INTERVAL_S`. Bounds
   how much progress an unclean shard exit — a panic, a kill -9, a host failure —
   can destroy. Without S2, S1 is the only writer and any crash loses the entire
   session's progress.
4. **S3 — save, on committing transactions that must not be lost**: a quest reaching
   `turned-in` ([`../mechanics/quests.md`](../mechanics/quests.md) rule 5.7.4) and a
   loot grant ([`../mechanics/loot.md`](../mechanics/loot.md) rule 5.6.2). These are
   the state changes a player would notice losing, and they are rare enough that
   writing synchronously is affordable. **The commit happens before the client is told
   it happened** (ADR 0031 §6): a crash in that window replays as "you already have it"
   on next login, never as a loss, whereas acknowledging first produces the bug report
   nobody can answer. Every S3 write covers `character_state`, `character_inventory`
   and `character_quests` in **one** transaction, so inventory and quests can never
   disagree.

**How S1 attaches, and the three ways it goes wrong.**

5. S1 runs in the session layer's teardown `defer`, **before** `Zone.Leave`, on a
   snapshot the zone hands back in one call taken under its own mutex — read
   `Zone.SnapshotCharacter(entityID) (CharacterSnapshot, bool)`, then `Zone.Leave`.
   `Zone.Leave` itself **stays a pure in-memory eviction**: no I/O, no error return, no
   persistence hook, ever (ADR 0031 §8). Hanging the save off `Leave` would put a
   database round trip under the zone mutex, on paths where the session has already
   failed, with the failure returned to a `defer` that has no caller left to report to.
   The three failure modes this arrangement avoids:
   1. **Saving after `Leave`.** `Leave` deletes the entity from the zone's map, so a
      read afterwards finds nothing and the save silently writes a stale or zero
      position. This is the default outcome if S1 is simply added as a second `defer`,
      because deferred calls run last-in-first-out and the teardown `defer` was armed
      first.
   2. **Reading the entity's fields without the zone lock.** The zone's tick loop
      mutates position under the zone mutex on `TICK_INTERVAL_MS`, so a field-by-field
      read from outside can save a position the player was never at — a torn mix of two
      ticks. `SnapshotCharacter` copies the whole entity under one acquisition of the
      same mutex, which is why it is a zone method rather than session code reaching
      into zone state. The residual window between that call and `Leave` is at most one
      tick, the session is already `Draining` so no further intent is applied, and
      position is reversible state under ADR 0031 §6 in any case.
   3. **Reusing the request context.** The disconnect path is frequently reached
      *because* the context was cancelled — shard shutdown cancels the context that
      `Serve` passes into the handler. A save issued on that context fails immediately,
      every time, on exactly the shutdown that most needs it to succeed. S1 must use a
      fresh context bounded by `SAVE_TIMEOUT_S`, derived from the process's background
      context, not from the request's.
6. S1 cannot return an error to anyone: the deferred call has no caller left to report
   to, and the connection is already going away. A failed S1 therefore logs at error
   level with `character_id` and the reason, increments a counter, and appends the
   snapshot to `charstore`'s bounded on-disk retry queue, which retries with backoff
   and alerts (ADR 0031 §8). It never swallows the failure, never panics, and never
   blocks past `SAVE_TIMEOUT_S`.
7. Saves are idempotent and last-writer-wins on `save_seq`, the monotonic per-character
   counter ADR 0031 §6 defines; `saved_at_tick` travels in the snapshot for diagnostics
   but is not the guard, because tick counters restart with the process. A retried S1
   that lands after the character has already reconnected and been saved again is
   rejected because its `save_seq` is not greater than the stored value, and rule
   5.3.4's one-session-per-character rule is what makes that simple comparison
   sufficient.
8. **S0 — save, at zone entry** (rule 5.4.4.4), after L1 has read the record and the
   entity exists, stamping `zone_id` on `shard.character_state` (ADR 0031 §5.2). It is
   not a redundant write-back of what L1 just read: it is the record that this character
   is now in this zone, which is what a later load and any operator inspection depend
   on. It is the one checkpoint whose failure is reported to the client as a normal
   error, because the session is still healthy at that point.

## 6. Worked example

### 6.1 A complete M2 session

| # | Direction | Channel | Message | Session state after |
|---|---|---|---|---|
| 1 | — | — | out of band: auth `POST /v1/sessions`, character list, `POST /v1/tickets` → `sarnaut_tk_…` | — |
| 2 | C → S | reliable | `ClientHello{PROTOCOL_VERSION_1, "dev", pack_id}` | `Connected` |
| 3 | S → C | reliable | `ServerHello{PROTOCOL_VERSION_1, "dev", pack_id}`; both ends compare `pack_id` | `Versioned` |
| 4 | C → S | reliable | `EnterZoneRequest{"InstLeague1", ticket}` | — |
| 5 | — | — | NATS `sarnaut.auth.v1.ticket.redeem` → `{account_id, character_id, …}`; play lock taken | `Authenticated` |
| 6 | — | — | **L1**: load `CharacterSnapshot` (materialize from chargen if absent) | `CharacterBound` |
| 7 | — | — | `JoinAt(loaded position)`; teardown armed | — |
| 8 | — | — | **S0**: stamp `zone_id` | — |
| 9 | S → C | reliable | `EnterZoneResponse{zone_id, own_entity_id, spawn_position}` | — |
| 10 | — | — | subscribe to snapshots | `InZone` |
| 11 | C → S | unreliable | `ClientMessage{move_intent}` ×N | `InZone` |
| 12 | S → C | unreliable | `ServerMessage{snapshot_batch}` every 66.7 ms | `InZone` |
| 13 | C → S | reliable | `ClientMessage{ability_use}` ×6 (`../mechanics/combat.md` §6.1) | `InZone` |
| 14 | S → C | reliable | `ServerMessage{combat_event}` ×6, then `ServerMessage{death_event}` | `InZone` |
| 15 | C → S | reliable | `ClientMessage{loot_take}` | `InZone` |
| 16 | — | — | **S3**: loot grant committed, *then* `ServerMessage{loot_result}` and `{inventory_update}` | `InZone` |
| 17 | C → S | reliable | `ClientMessage{quest_turn_in}` | `InZone` |
| 18 | — | — | **S3**: quest turn-in committed, *then* `ServerMessage{quest_state_update}` | `InZone` |
| 19 | C → S | reliable | `ClientMessage{logout}`, then the client closes (rule 5.6.4) | `Draining` |
| 20 | — | — | `SnapshotCharacter(entityID)`, then `Zone.Leave` | `Draining` |
| 21 | — | — | **S1**: persist the snapshot taken at step 20 | closed |

Steps 6, 8, 16, 18, and 21 are the only writes, and steps 16 and 18 commit **before**
the events they announce are written (rule 5.7.4). S2 would appear between steps 12 and
19 on any session lasting longer than `PERIODIC_SAVE_INTERVAL_S`.

### 6.2 The same session, killed by a shard shutdown at step 13

Shutdown cancels the session context and closes the connection, so all three loops in
rule 5.5.1 return and the handler returns. Step 19 never happens — the client never got
to close anything, and there is no acknowledgement owed either way. Steps 20 and 21
still happen, because teardown is a `defer` (rule 5.6.1), and step 21 succeeds only if
rule 5.7.5.3 was honoured. An implementation that passed the cancelled context
into the save loses the character's position on every shutdown, and the bug is
invisible in any test that only exercises the clean-logout path.

The test for this is: cancel the server context mid-session and assert the persisted
`CharacterSnapshot` matches the entity's last simulated position, not its spawn
position.

## 7. Open questions and placeholders

### 7.1 Reconnect and session resumption

A disconnected character is fully torn down. There is no grace period, no lingering
world entity, and no in-combat logout penalty. Reconnecting is a fresh session that
replays §5 from step 1 and re-enters at the saved position. Whether combat should
prevent instant logout is a mechanics question the combat spec has not reached.

### 7.2 Zone transfer

M2 has one zone. Moving between zones on the same shard, and between shards, both need
a checkpoint pair that this spec does not define: a save on leaving the source zone
that is not a disconnect, and a load on entering the destination that is not a
character select. The natural shape is to reuse S1 and L1 with the session staying
`Authenticated` in between, but nothing validates that yet.

### 7.3 The command envelope — built 2026-08-20

`SERVER:proto/sarnaut/v1/envelope.proto` exists, and so do `PROTO_LOCK.sha256` in both
repositories and the client's `scripts/sync-proto.ps1`. The cutover happened in one
merge window with no transition period, as ADR 0026 required: the shard, the Go probe,
`client/tools/SarnautCore.NetSmoke` and `server/scripts/sar20-client-smoke.ps1` all
moved together.

What is not built is behind the cases rather than in them. `CombatEvent`, `DeathEvent`,
`LootOffer`, `LootResult`, `InventoryUpdate` and `QuestStateUpdate` are declared with
no fields, and the shard drops the client verbs whose mechanics tasks have not landed
— it reads them, so nothing stalls, and it answers nothing. Filling those messages in
is additive and does not bump `ProtocolVersion` (ADR 0027).

Where ADR 0026 and a rule here disagree, the ADR wins and this section is the
amendment site.

### 7.4 Backpressure on the reliable channel

Snapshot delivery is explicitly lossy (rule 5.5.6) and needs no backpressure. Commands
and events are not lossy, and a client that stops reading will eventually stall the
server's writes. What the server does then — buffer, disconnect, or block — is
unspecified.

### 7.5 The handshake timeout is not enforced yet

`HANDSHAKE_TIMEOUT_S` is a curated constant with no enforcement point in the current
code. Rule 5.1.6 states the requirement; implementing it means giving the accept path
a deadline that is cleared on reaching `Authenticated`.

### 7.6 Logout and quest abandon — closed 2026-08-20

This section recorded that ADR 0026's case list carried neither a client `logout` case
nor a `quest_abandon` case. Both were added by the 2026-08-20 amendment to
[ADR 0026](../../adr/0026-wire-message-envelope.md), before the envelope was built,
and the rules above are written against the amended list:

- `quest_abandon` carries `QuestAbandon` and makes
  [`../mechanics/quests.md`](../mechanics/quests.md) transitions T14 and T15 reachable
  from a client. They were specified and unreachable.
- `logout` carries `Logout`, an empty message whose actor is the session (rule 5.2.6).
  It exists so save checkpoint S1 runs ahead of the disconnect instead of racing it
  (rule 5.6.4). The shard acknowledges nothing.

Neither bumps `ProtocolVersion`: both are additive oneof cases (ADR 0027).

What is still open is the **server** side of abandon. `QuestStateUpdate`, the renamed
`quest_state_update` case, has no fields yet; the quest task fills them in against
`../mechanics/quests.md` rule 5.2, and adding fields to an existing message is
additive too.

## 8. Sources

Everything cited is SarnautCore's own source, not reference material. This spec
describes a clean protocol (ADR 0010) and derives nothing from the retail wire format.

SarnautCore server-repository paths (`SERVER:` = `server/`):

- `proto/sarnaut/v1/common.proto` — the `ProtocolVersion` enum.
- `proto/sarnaut/v1/handshake.proto` — `ClientHello` and `ServerHello` with
  `protocol_version`, `build_id` and the `pack_id` of rule 5.1.4.
  `EnterZoneRequest` / `EnterZoneResponse` live in `replication.proto`, and the
  `ticket` field of rule 5.4.1 is on the request.
- `proto/sarnaut/v1/envelope.proto` — `ClientMessage`, `ServerMessage`, `Error` and
  `ErrorCode`, per ADR 0026 and its 2026-08-20 amendment; see §7.3.
- `proto/PROTO_LOCK.sha256` and `internal/session/reader.go` — the lock over the
  `.proto` set (ADR 0027) and the two reader goroutines of rule 5.5.1.
- `proto/sarnaut/v1/movement.proto` — `ClientMoveIntent` with `seq`, `input`,
  `heading`, `dt_seconds`; and `Vec3`.
- `proto/sarnaut/v1/replication.proto` — `SnapshotBatch` and `EntitySnapshot`.
- `internal/transport/framing.go` — `MaxUnreliableMessageSize` and its rationale.
- `internal/session/handshake.go` — the current hello-then-`EnterZone` ordering, the
  point at which zone teardown is armed relative to entity creation (rule 5.6.1), the
  response-before-subscribe ordering (rule 5.4.7), the newest-only snapshot queue
  (rule 5.5.6), the size-bounded batch splitting (rule 5.5.7), and the datagram
  fallback to the reliable channel (rule 5.5.8).
- `internal/world/world.go` — `Zone.Join` taking no position argument (rule 5.4.4.3),
  `Zone.Leave` as pure in-memory eviction, which rule 5.7.5 and ADR 0031 §8 keep that
  way, `maxIntentDuration`, and the sequence-number staleness check (rule 5.5.5).
- `config.example.yaml` — `world.tick_interval`, `world.snapshot_interval`.

Governing ADRs: [0010](../../adr/0010-protocol-posture.md) (clean protocol),
[0016](../../adr/0016-server-modular-monolith.md) (separate auth service),
[0017](../../adr/0017-quic-protobuf-transport.md) (QUIC, protobuf, streams versus
datagrams, the transport seam, and the 2026-08-20 amendment behind rule 5.5.8),
[0026](../../adr/0026-wire-message-envelope.md) (message envelope, channel assignment
and the reader topology, §5.5), [0027](../../adr/0027-proto-contract-and-wire-evolution.md)
(the `pack_id` handshake check in rule 5.1.4 and the additive-change rule in §7.6),
[0029](../../adr/0029-runtime-pack-format.md) (what a `pack_id` is a digest of),
[0030](../../adr/0030-auth-account-service-session-tokens.md) (the auth service, the
opaque ticket and the play lock in §5.2 and §5.3),
[0031](../../adr/0031-persistence-and-migrations.md) (the storage the checkpoints in
§5.7 write to, and `Zone.Leave` staying pure),
[0032](../../adr/0032-character-creation.md) (what exists to select in §5.3 and what
materializes in rule 5.4.4.2), and
[0033](../../adr/0033-m2-module-topology-gateway-deviation.md) (why admission
terminates at the shard in M2).

Where any of those ADRs and a rule here disagree, the ADR wins and the rule is
amended in place with a change-log entry.

Per ADR 0011, nothing here is quoted, paraphrased, or transliterated from decompiled
code, and no game data appears in this document.

## 9. Change log

| Date | Change |
|---|---|
| 2026-08-20 | Created for M2. |
| 2026-08-20 | Reconciled against ADRs 0026, 0027, 0029, 0030, 0031, 0032, 0033 and the ADR 0017 amendment: envelope message names and channel assignment in §4 and rule 5.5; `pack_id` gate in rule 5.1; ticket redemption replacing `AuthenticateRequest` in §5.2 and out-of-band character select in §5.3; zone-entry load/materialize/S0 in rules 5.4.4 and 5.7.8; `Zone.Leave` kept a pure eviction in rule 5.7.5; `save_seq` as the stale-write guard in rule 5.7.7; §7.6 added for the missing `logout` and `quest_abandon` cases. |
| 2026-08-20 | Amended for the ADR 0026 amendment of the same date: `quest_abandon` and `logout` join the case list in rule 5.5.3, rule 5.6.4 makes `logout` the clean exit rather than a bare close, `quest_state` becomes `quest_state_update`, and §7.3 and §7.6 close now that `envelope.proto`, the proto lock and the client sync script exist. |

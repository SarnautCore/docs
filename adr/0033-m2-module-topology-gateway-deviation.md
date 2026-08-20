# ADR 0033 — M2 shard module topology, enforced by depguard; gateway deviation

**Status**: Accepted (2026-08-20)

## Context

ADR 0016 grants the shard four internal modules — world, combat, quests, chat —
and requires that "module boundaries inside the shard are reviewed as if they were
service boundaries — no cross-module reach-ins". M2 needs three things that ADR
0016 never named a home for:

- **inventory and loot**, required by "kill one mob with one ability → loot" (ADR 0003);
- **character persistence**, required by ADR 0031;
- **interest management** — `world.Zone.interestedEntitiesLocked` currently sorts
  every entity id in the zone and ships all of them to every player, carrying
  `TODO(SAR-19): replace all-entities interest with a spatial query`.

Nothing enforces ADR 0016's rule. "Reviewed as if they were service boundaries" is
a sentence, and there is no artifact that can fail.

ADR 0016 also grants a thin gateway terminating transport and sessions. The code
does not have one in the data path: `cmd/shard` builds its own QUIC listener and
runs `session.Server.Serve` against it directly, while `cmd/gateway` dials the
shard, exchanges a hello, logs the result and serves health probes. Client traffic
would go straight to the shard.

## Decision

### 1. Module homes

Eight shard modules for M2, each one Go package under `server/internal/`. Four are
ADR 0016's; four are new and are named here.

| Module | Package | Owns | State |
|---|---|---|---|
| `world` | `internal/world` | Zone entity registry, fixed-rate tick, movement integration | exists |
| `interest` | `internal/interest` | Spatial index; per-observer visibility sets | new |
| `combat` | `internal/combat` | Ability use, damage application, death, threat | new |
| `loot` | `internal/loot` | Loot tables, corpse loot rights, roll resolution | new |
| `inventory` | `internal/inventory` | Per-character containers, equip/unequip, stack rules | new |
| `quests` | `internal/quests` | Quest state machine, objective tracking, reward grant | new |
| `chat` | `internal/chat` | Channels and delivery | new |
| `charstore` | `internal/charstore` | The only writer of `shard.*` (ADR 0031); load, save, materialize | new |

Three of those placements are the actual decisions:

**Inventory and loot are two modules, not one.** Loot decides *what drops and who
is entitled to take it*. Inventory decides *whether a character can hold it and
where it goes*. They fail in opposite directions — a loot bug duplicates items, an
inventory bug loses them — so they want separate tests and separate blast radii.
Loot is per-corpse and lives for seconds; inventory is per-character and outlives
the session. The seam between them is one call: `loot` offers `inventory` an award,
`inventory` accepts it or refuses for space, and `loot` handles the refusal.

**Interest management is its own module, not part of `world`.** The all-entities
scan is the algorithm we already know is wrong, and it will be replaced more than
once — a uniform grid first, then whatever profiling asks for. Behind an interface
that is a swap. Inside `world`'s locked snapshot path it is a rewrite of the tick
loop, which is the code we least want to keep reopening.

**Character persistence is `charstore`, not a method on `world`.** ADR 0031 §7 and
§8 give the reason: persistence must be unreachable from the tick loop and from
`Zone.Leave`. Making it a separate package is what makes "unreachable" checkable.

Types that several modules share — entity ids, `Vec3`, item ids, character ids —
live in a leaf package `internal/gametypes` that everything may import and that
imports nothing but the standard library. Without it, the first shared struct
creates a cycle in the allow-list below.

### 2. The no-cross-module-reach-in rule, made mechanical

From M2 the rule is an import-graph check that runs in CI and fails the build. It
is not reviewer judgement.

**Mechanism: the `depguard` linter, in the golangci-lint run the repo already has.**
`.github/workflows/ci.yml` runs `golangci-lint-action` at pinned `v2.12.2`, and
`.golangci.yml` is already v2-format. depguard supports per-path rules — a `files`
selector plus an `allow`/`deny` list — which is exactly the shape needed. This adds
no tool, no second config file, and no new CI step: an illegal import fails the
existing **Lint** step and `make lint` locally.

`.golangci.yml` gains one depguard rule per module, each naming the internal
packages that module may import. The graph is acyclic and flows one way:

| Module | May import |
|---|---|
| `internal/session` | `world`, `combat`, `loot`, `inventory`, `quests`, `charstore`, `chat`, `transport`, `gametypes` |
| `internal/world` | `interest`, `combat`, `gametypes` |
| `internal/combat` | `loot`, `gametypes` |
| `internal/loot` | `inventory`, `pack`, `gametypes` |
| `internal/quests` | `inventory`, `pack`, `gametypes` |
| `internal/charstore` | `inventory`, `quests`, `gametypes` (snapshot types only) |
| `internal/inventory` | `pack`, `gametypes` |
| `internal/interest` | `gametypes` |
| `internal/gametypes` | nothing outside the standard library |

Any internal import not on a module's row fails lint. `transport`, `pack`
([ADR 0029](0029-runtime-pack-format.md)'s reader, which replaces `internal/content`)
and `gametypes` are leaves rather than modules, and appear only on the right-hand
side. `internal/session` sits above all eight because it is the dispatch layer for
[ADR 0026](0026-wire-message-envelope.md)'s envelope: every `ClientMessage` oneof case
— `ability_use`, `interact`, `loot_take`, `quest_accept`, `quest_turn_in` — is routed
from `session` to the module that owns it, so a row that omitted `combat`, `loot`,
`inventory` or `quests` would make ADR 0026's dispatch table unimplementable. Two
further rules in the same config carry the seams from other ADRs:

- **No package may import `internal/infra`, `github.com/jackc/pgx/...`,
  `github.com/redis/...` or `github.com/nats-io/...` except `internal/charstore`,
  `internal/auth` and `cmd/...`.** This is ADR 0031's "the 30 Hz tick performs no
  database I/O" turned into something that fails a build. One exemption, narrow and
  dated: `internal/session` may import `github.com/nats-io/...` **only**, because §3
  puts ticket redemption (`sarnaut.auth.v1.ticket.redeem`, ADR 0030) there for M2. It
  stays banned from `pgx`, `redis` and `internal/infra`, so the exemption cannot grow
  into a second database caller, and the line is deleted from `.golangci.yml` when
  redemption moves to the gateway — which makes reverting the deviation visible in the
  same file that records it.
- **No package may import `github.com/quic-go/...` except `internal/transport`.**
  This is ADR 0017's transport seam, likewise.

Changing one of these allow-lists is an architecture change. It gets reviewed as
one, and the diff makes that visible, which prose in an ADR never did.

### 3. Deviation from ADR 0016: the gateway is not in the M2 data path

**Deviation note — 2026-08-20.** ADR 0016 grants a thin gateway terminating
transport and sessions. For M2, SarnautCore **keeps the direct client-to-shard QUIC
connection** that `cmd/shard` and `internal/transport` implement today, and
**terminates authentication at the shard**: the client presents the ADR 0030 ticket
in `EnterZoneRequest`, and `internal/session` redeems it over
`sarnaut.auth.v1.ticket.redeem` before granting a spawn. `cmd/gateway` stays in the
repo as the handshake conformance client it already is and carries no player
traffic.

This is a deliberate, scoped deviation from ADR 0016, not an oversight and not a
supersession. **ADR 0016 stands**; this ADR records a temporary departure from one
clause of it, the reason, and the bill.

#### What ADR 0016's thin edge is supposed to own

Everything in this list is therefore absent in M2, or sitting in the wrong process:

- **Connection and session termination** — QUIC handshake, TLS certificates,
  connection migration, idle timeouts — held off the simulation process, so
  rotating a certificate or upgrading the transport does not restart the world.
- **Authentication and admission** — ticket redemption and rate limiting at the
  edge, so an unauthenticated peer never reaches a process holding zone state.
- **The public attack surface** — the only internet-facing listener, letting shards
  bind to a private network and never accept an untrusted connection.
- **Fan-out and routing** — one client connection routed to whichever shard hosts
  the player's zone, and the seam a zone handoff lives behind. This is what makes
  ADR 0016's "further splits along the existing module seams" possible at all.
- **Backpressure and abuse control** — per-connection budgets applied before a
  message costs the simulation anything.
- **The retail-protocol front-end door** (ADR 0010) — a second wire format would
  terminate at the edge, not inside the sim.

#### What checking the ticket at the shard duplicates

- The redemption call, its NATS client and the `sarnaut.auth.v1` protobufs get
  written against `internal/session` now and wired up again at the edge later. The
  redemption logic itself moves rather than being rewritten; the session-state
  plumbing around it — who holds the redeemed identity, when it is dropped, what a
  half-admitted connection looks like — is written twice.
- `session.Server` grows admission, rate limiting and per-connection accounting
  that the gateway is meant to own. Those are the parts that get **deleted** rather
  than moved, so they are pure duplicate cost.
- Dev TLS material and QUIC listener configuration live on the shard today
  (`internal/transport/tls.go`, `cmd/shard`) and will live on the gateway later.
  Until then both places have to be kept correct.

#### What undoing the shortcut will cost

Paid at M3, or the first time a second shard or a second zone exists — whichever
comes first.

- **Interpose the gateway.** It terminates the client QUIC connection and opens a
  second internal connection to the shard. Because ADR 0017 already hides QUIC
  behind `transport.Connection`/`transport.Listener`, the shard side is close to a
  listen-address change and the client side is a hostname change.
- **Move redemption out of `internal/session` into the gateway**, and change the
  shard's admission input from "a ticket" to "an identity asserted by a trusted
  peer". This introduces a shard↔gateway trust boundary that does not exist today,
  with the mutual-TLS or shared-secret story that goes with it. **This is the
  genuinely new work**, and it is the reason the deviation is cheap now: that
  boundary is the hard part, and building it for a topology with one shard would be
  building it against no requirements.
- **Split `internal/session` in two.** The connection-level half moves to the
  gateway; the zone-level half — `EnterZone`, move intents, `snapshotSender` — stays
  on the shard. The snapshot back-pressure decisions in `snapshotSender` have to be
  re-made once there are two hops, because dropping a stale snapshot at the edge and
  dropping one at the sim are different policies.
- **Add the routing table** that maps a zone to a shard, plus the client's reconnect
  behaviour when the mapping changes.

Estimated at one focused work wave, contained to `internal/session`,
`internal/transport` and `cmd/gateway`, with **no change to `world`, `interest`,
`combat`, `loot`, `inventory`, `quests`, `chat` or `charstore`**. That containment
is the condition the deviation is taken on. If any change makes a simulation module
aware of the transport or of a ticket, the deviation has failed its own test and the
gateway goes in immediately rather than at M3. The depguard rules in §2 are what
detect that: a simulation module importing `internal/transport` is already a lint
failure.

## Alternatives rejected

- **One `internal/game` package containing all eight modules**, with unexported
  seams. This is what the code grows into if left alone, and it is faster for the
  first month. Rejected because Go's only mechanical boundary is the package, so
  collapsing them makes ADR 0016's "reviewed as if they were service boundaries"
  permanently unenforceable — every reach-in becomes a legal field access.
- **Separate Go modules** (`go.mod` per component) to force the boundary at the
  toolchain level. Genuine enforcement. Rejected because it buys version skew and
  `replace` directives inside a single binary, for a rule that a lint rule in an
  existing CI step enforces just as reliably and far more legibly.
- **`go-arch-lint`, or a hand-written `go list -deps` script in `make verify`.**
  Both work. Rejected because each adds a tool to install and pin, and a second
  place where import rules live, when golangci-lint is already installed, already
  pinned to `v2.12.2` in CI, and already fails the build.
- **Relying on ADR 0016's prose plus code review.** This is the status quo, and the
  status quo has produced no artifact that can fail. Rejected on the grounds that a
  rule nobody can run is a rule nobody checks on the day it matters.
- **Building the gateway now, as ADR 0016 says.** One extra process, one routing
  table and one trust boundary — for a milestone with one shard, one zone and no
  handoff. Every message would cross a hop that exists only to be correct on paper,
  and the trust boundary, which is the expensive part, would be designed against no
  real requirements.
- **Deleting `cmd/gateway` since it is not in the data path.** Rejected: it costs
  nothing to keep, it is a real conformance client for the handshake, and deleting
  it would make the deviation invisible in the tree. A dead binary named `gateway`
  is a standing reminder that something is owed.

## Consequences

- ADR 0016 is not superseded and is not amended. This ADR is the record of a
  time-boxed departure from its gateway clause; ADR 0016's module-boundary clause is
  strengthened here, not weakened.
- `.golangci.yml` becomes an architecture document. Reviewers treat a depguard
  allow-list change with the weight of an ADR amendment.
- The seven new packages land as real, mostly empty packages as M2 features arrive.
  A module with no code still gets its depguard rule on day one, so the *first*
  import into it is checked rather than the fiftieth.
- `internal/gametypes` must stay a leaf. If it ever imports a module, the whole
  allow-list collapses into a cycle and the check stops meaning anything. Its own
  depguard rule enforces that.
- Any M2 work that makes the shard's transport handling harder to lift out is a
  violation of this ADR and is reviewable against the cost list above.

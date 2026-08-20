# ADR 0031 — Persistence: PostgreSQL, goose migrations, checkpoint saves

**Status**: Accepted (2026-08-20)

## Context

`internal/infra` builds a `pgxpool.Pool` from `SARNAUT_POSTGRES_DSN`, pings it, and
that is the entire relationship the server has with PostgreSQL. Nothing issues a
query. There is no SQL in the repo, no migrations directory, and no migration
library in `go.mod`.

The infrastructure is already there: `infra/compose/docker-compose.yml` runs
`postgres:18` with database and user `sarnaut`, password from `POSTGRES_PASSWORD`
(dev default `sarnaut_dev`), published on host port **5433** mapped to the
container's 5432.

M2 is the first milestone where a character has to exist between two logins, and
the first where losing state is visible to a player. The interesting decisions are
not "use Postgres" — they are *when* we write, *what a crash costs*, and *which
code is allowed to touch a database at all*.

## Decision

### 1. PostgreSQL 18 is the only durable store

Everything a player would be upset to lose lives in PostgreSQL. Driver stays
**`github.com/jackc/pgx/v5 v5.10.0`** with `pgxpool`, already a direct requirement.
Dev DSN:

```
postgres://sarnaut:sarnaut_dev@127.0.0.1:5433/sarnaut?sslmode=disable
```

supplied through the existing `SARNAUT_POSTGRES_DSN` variable.

Valkey holds only what may be lost without consequence: sessions, tickets, play
locks (ADR 0030), and caches. ClickHouse holds telemetry, and nothing in it is
authoritative for gameplay. If a decision depends on a value, that value is in
PostgreSQL.

### 2. Migration tool: goose

**`github.com/pressly/goose/v3 v3.27.3`**, used both as a library at process start
and as a CLI during development.

Migrations are embedded with `embed.FS` and applied over `database/sql` using
`github.com/jackc/pgx/v5/stdlib` — a package of the pgx module already required, so
this adds one dependency, not two. goose wraps each migration in a transaction and
takes a session-level advisory lock, so two shard replicas starting at once do not
race and a failed migration leaves nothing half-applied.

### 3. Migrations directory and naming

One flat directory, `server/migrations/`, covering both schemas:

```
server/migrations/00001_initial_schema.sql
server/migrations/00002_....sql
server/migrations/migrations.go     // //go:embed *.sql
```

- Five-digit zero-padded sequential numbers starting at `00001` (`goose create -s`),
  then `snake_case` description, then `.sql`.
- Each file carries `-- +goose Up` and `-- +goose Down` sections.
- **Every migration is reversible.** One that genuinely cannot be reversed says so
  in a comment naming what would be lost, and its `Down` section contains an
  explicit no-op rather than being omitted, so the intent is visible in review.

Sequential numbering rather than timestamps: timestamps solve collisions between
long-lived concurrent branches, and this project has one repo with a linear main
(ADR 0022, ADR 0023). Sequence numbers make ordering legible at a glance, which
matters more here.

### 4. Table ownership

Two schemas in one database. **No cross-schema foreign keys** — `shard` references
a `character_id` that `auth` minted, and validates it only through ticket
redemption (ADR 0030). Each table has exactly one owning module.

| Table | Owner | Contents |
|---|---|---|
| `auth.accounts` | auth service | `account_id uuid pk`, `email citext unique`, `password_hash text` (PHC string), `created_at`, `disabled_at` |
| `auth.characters` | auth service | `character_id uuid pk`, `account_id uuid` → `accounts`, `name text`, `name_normalized citext unique`, `chargen_option_id text`, `created_at`, `deleted_at` |
| `auth.name_reservations` | auth service | `name_normalized citext pk`, `account_id uuid`, `reserved_until timestamptz` |
| `shard.character_state` | shard `charstore` | `character_id uuid pk`, `zone_id text`, `position_x/y/z real`, `heading real`, `level int`, `experience bigint`, `health int`, `save_seq bigint`, `saved_at timestamptz` |
| `shard.character_inventory` | `charstore`, for `inventory` | `character_id uuid`, `slot int`, `item_id text`, `quantity int`; pk `(character_id, slot)` |
| `shard.character_quests` | `charstore`, for `quests` | `character_id uuid`, `quest_id text`, `state text`, `objectives jsonb`, `updated_at`; pk `(character_id, quest_id)` |
| `public.goose_db_version` | goose | The migration ledger |

Ownership is enforced twice, in code and in the database:

- **`internal/charstore` is the only package that may write `shard.*`**, and
  `internal/auth` the only one that may write `auth.*`. Enforced by the import-graph
  check in ADR 0033: no other module may import `pgx` at all.
- Migration `00001` creates PostgreSQL roles `sarnaut_auth` and `sarnaut_shard`,
  each granted only on its own schema, and enables `citext`. The shard's DSN uses
  `sarnaut_shard`, so a stray query against `auth.accounts` fails with a permission
  error rather than working.

`inventory` and `quests` do not talk to the database themselves. They hand
`charstore` a plain-Go snapshot of their state and take one back on load. That is
what keeps them testable without a container, and it is why the table rows say
"`charstore`, for `inventory`" rather than naming `inventory` as the owner.

### 5. Save cadence: explicit checkpoints, not write-behind

`charstore` exposes `Save(ctx, characterID, snapshot) error`, and it is called at
exactly these points, all of them outside the tick loop:

1. **Character creation**, materializing state from the chargen table (ADR 0032),
   before the first spawn is granted.
2. **Zone entry**, after the ticket redeems and state loads, stamping `zone_id`.
3. **Every 60 seconds per connected character**, from a `charstore` ticker
   goroutine, skipped if `save_seq` has not advanced since the last save.
4. **Immediately after any irreversible gain**: loot picked up, item consumed,
   quest reward granted, quest state transition, level-up.
5. **On clean logout.** In M2 that is the client closing the connection —
   [ADR 0026](0026-wire-message-envelope.md)'s message set has no logout case — so it
   is the same session-layer save as §8's disconnect path, not a second one.
6. **On shard shutdown**, for every connected character, before the process exits.

Each save is **one transaction** covering `character_state`, `character_inventory`
and `character_quests` together. Never three transactions, never a partial write.

### 6. Crash-consistency expectations

What M2 promises, in the form a player would state it: *inventory and quest state
never disagree with each other, and you never lose an item or a completed
objective — you can lose up to a minute of walking and progress.*

Concretely:

- Because every save is one transaction, inventory and quests are always mutually
  consistent as of some past instant. There is no reachable state where a quest is
  recorded complete but its reward item is absent, or where a loot flag is set and
  the item is not in the bag.
- Irreversible gains are committed **before** the client is told they happened. A
  crash in that window replays as "you already have it" on next login, never as a
  loss. The reverse ordering — acknowledge, then save — is the one that produces
  the bug report we cannot answer.
- Position, unspent experience and partial objective counters are reversible and
  may roll back up to 60 seconds. That is the accepted cost.
- `save_seq` increments per character per save. A save whose `save_seq` is not
  greater than the stored value is rejected, so a slow write from a dying session
  cannot clobber a newer write from a reconnect that already happened.

### 7. The 30 Hz tick performs no database I/O

`world.Zone.step` and `world.Zone.publishSnapshot` — driven from `Zone.Run` on a
`TickInterval` ticker whose default is `time.Second/30` — must not reach
PostgreSQL, Valkey or NATS, directly or through any transitive call. `step` holds
`zone.mu` for its whole body; a 5 ms query there stalls movement for every player
in the zone, not just the one being saved. `charstore` always works from a copied
snapshot, on its own goroutine, never inside a zone lock.

Enforced mechanically, two ways:

- The ADR 0033 import-graph check forbids `internal/world` from importing
  `internal/infra`, `internal/charstore`, `github.com/jackc/pgx/...`,
  `github.com/redis/...` or `github.com/nats-io/...`. An I/O call cannot be written
  there without failing `make lint`.
- `TestZoneStepBudget` in `internal/world` runs 10,000 `step()` calls over a
  populated zone and fails if p99 exceeds 1 ms.

### 8. Character save must not hang off `world.Zone.Leave`

This is the trap this ADR exists to close.

`session.Server.handle` registers `defer zone.Leave(entityID)` immediately after
`zone.Join()`, and `handle` returns on *every* error path: a failed snapshot write,
a decode error, a dead datagram read, a client that vanished mid-move. Those are
not edge cases — they are the ordinary way a session ends. `Zone.Leave` takes
`zone.mu`, deletes the session sink and the entity, and returns nothing.

Attaching the save to `Leave` would put a database round trip under the zone mutex,
run it on paths where the session has already failed and the process may be
shutting down, and give the failure nowhere to go: `Leave` has no error return and
its caller is a `defer`. The disconnects most likely to lose state — write failures
and crashes — are exactly the ones where that save would be attempted under the
worst conditions and its failure silently discarded.

Therefore:

- **`Zone.Leave` stays a pure in-memory eviction.** No I/O, no error return, no
  persistence hook, ever. It is a memory-management call and nothing else.
- Persistence on disconnect is driven from the session layer **before** `Leave`
  runs, from a snapshot taken while the entity still exists. The zone exposes that as
  one read — `Zone.SnapshotCharacter(entityID)`, copying the entity under a single
  acquisition of `zone.mu` — so the session layer never reads entity fields piecemeal
  and never persists a torn mix of two ticks (`specs/protocol/session.md` rule 5.7.5).
- That save uses a context derived from the shard's lifetime context, **not** the
  connection context, with its own 5 s timeout. A cancelled connection must not
  cancel the save of the state it was holding.
- If it fails, `charstore` appends the snapshot to a bounded on-disk retry queue and
  retries with backoff, logging `character_id` and alerting. It does not swallow.
- The 60-second checkpoint is the backstop. Even if the disconnect-time save is
  lost outright, the ceiling on damage is one minute of reversible progress, and
  zero irreversible gains, because those were committed when they happened.

## Alternatives rejected

- **`github.com/golang-migrate/migrate/v4 v4.19.1`**. The other obvious choice, and
  widely deployed. Rejected on its `schema_migrations.dirty` flag: a failed
  migration sets it, and clearing it requires a human running `force` with a version
  number. This project runs unattended between hard stops (ADR 0025), so a dirty
  flag converts a bad migration into a halted pipeline only a person can restart.
  Its separate up/down file pair also doubles the review surface for one change.
- **`CREATE TABLE IF NOT EXISTS` DDL at boot**, no tool. Fine for two tables.
  Rejected because it has no ledger, no ordering and no down path, so by the tenth
  table the dev database and the schema in the repo have quietly diverged and
  nothing reports it.
- **Write-behind with a dirty set flushed on an interval.** This is the standard
  MMO answer and it is correct at scale. Rejected for M2 because at this size the
  throughput problem it solves does not exist, while the failure mode it introduces
  does: a crash mid-flush leaves some characters with the item and the loot flag,
  some with neither, and there is no reconciliation code to sort it out. Revisit
  when save volume rather than correctness is the binding constraint — the
  `charstore.Save` seam is where that change lands.
- **A separate database per service** instead of two schemas in one. The cleaner
  long-term split. Rejected for M2 because it doubles the dev container footprint
  and the connection configuration to buy an isolation that separate roles already
  provide, and moving a schema to its own database later is a DSN change plus a
  dump/restore.

## Consequences

- Two new packages own all SQL: `internal/charstore` (`shard.*`) and
  `internal/auth` (`auth.*`). `internal/infra` keeps building the pool and stops
  being interesting.
- `go.mod` gains one direct requirement: `github.com/pressly/goose/v3 v3.27.3`.
- CI gains a job that starts `postgres:18` as a service, then runs `goose up`,
  `goose down-to 0`, `goose up` against it. Every `Down` section is proven rather
  than assumed, which is the only way reversibility stays true.
- PostgreSQL becomes required for shard and auth startup. `infra.openPostgres`
  currently returns `nil` when the DSN is empty; for those two services it must
  fail instead.
- Backup, restore and point-in-time recovery are deliberately out of scope until
  there is something to lose. Production deployment is a hard stop (ADR 0025).
- Existing tests that construct a `world.Zone` keep working untouched, because
  nothing in `internal/world` gains a database dependency. That is the point of §7.

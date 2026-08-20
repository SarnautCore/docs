# ADR 0032 — Character creation: one League melee option, driven from a pack table

**Status**: Accepted (2026-08-20)

## Context

ADR 0003 scopes M2 to "character creation (one race/class)" and a spawn on a
starting isle. Nothing in the server has a character today: `world.Zone.Join`
hands every connecting player the same `PlayerSpawn` read from `WorldConfig`, and
`internal/config` defaults the shard's zone to `InstLeague1` with content slug
`inst-league1` — which is also the only zone under `data/classic/zones/` with
extracted spawns, quests and routes.

Two things get decided here that are easy to get wrong cheaply: *which* one option,
and *where the option's facts live*. The second matters more. Every starting item,
stat and coordinate that ends up in Go source or C# source is a server rebuild the
day a second class exists.

## Decision

### 1. The one playable option is `chargen.league.warrior`

A League-side melee starter, spawning in `inst-league1`.

It is the only combination the rest of M2 can actually serve. The zone data exists
and nothing else does. A melee starter satisfies "kill one mob with one ability"
(ADR 0003) with no resource bar, no cast timer, no projectile travel, no pet AI
and no line-of-sight rules — every one of which is a system M2 would otherwise
have to ship to make the demo honest. The Empire side and every caster or pet
option are M3, and are expected to arrive as data plus the systems they need, not
as a second special case.

Per ADR 0007 the identifier is the canonical English one; display names are
`loc_ref` lookups, never literals.

### 2. Options, loadout and spawn come from a compiled pack table

A new document type, `chargen`, authored as YAML in the `data` repo, validated by a
JSON Schema in `data-schemas`, compiled into the runtime pack by `sarnaut-pack`
([ADR 0029](0029-runtime-pack-format.md)) — the same path every other document type
takes under ADR 0006. It compiles to a `chargen` table in the ruleset-global pack
(`<packs-root>/classic/-/tables/chargen.sptbl`), with a row type declared in
`data-schemas/proto/sarnaut/content/v1/`.

```
data-schemas/schemas/chargen.schema.json
data/classic/chargen/chargen.league.warrior.yaml
```

**Neither the client nor the server may hold a starting item, a starting stat, a
starting ability or a spawn coordinate in code.** A `chargen` document carries:

| Field | Meaning |
|---|---|
| `schema_version`, `id` | `id` matching `^chargen\.[a-z0-9]+(?:-[a-z0-9]+)*(?:\.[a-z0-9]+(?:-[a-z0-9]+)*)+$`, mirroring the existing `item.schema.json` id pattern |
| `race`, `class`, `faction` | Canonical ids |
| `enabled` | Boolean. This is how "one option in M2" is expressed |
| `loc_ref` | Display name and description keys (ADR 0007) |
| `visual_ref` | The model and material every character of this option renders with |
| `spawn` | `zone_id`, `position {x, y, z}`, `heading` |
| `starting_level` | Integer |
| `starting_stats` | List of `{stat, value}`, matching the item schema's stat shape |
| `starting_loadout` | List of `{item_id, quantity, slot}`; `slot` is an equipment slot or `bag` |
| `starting_abilities` | List of ability ids |
| `starting_quests` | Quest ids granted at first spawn |

Every other option ships with `enabled: false` rather than being absent from the
tree, so adding the second playable option is a one-line data change that CI has
already been validating. The option list the client renders is the set of `enabled`
documents; the client asks the server for it and never derives it.

The shard reads the table through `internal/pack`, the pack reader that replaces
`internal/content`'s YAML walk (ADR 0029). `world.ZoneConfig.PlayerSpawn`
stops being where a player's spawn comes from and survives only as a fallback for
debug and unowned entities.

### 3. Name validation, uniqueness and reservation

**Shape.** 3 to 16 characters. First character an uppercase ASCII letter. Remaining
characters lowercase ASCII letters, or a single apostrophe or hyphen that is
neither first nor last and is not adjacent to another apostrophe or hyphen. No
digits, no spaces, no non-ASCII in M2.

Two checks, in this order:

```go
var namePattern = regexp.MustCompile(`^[A-Z][a-z]*(?:['-][a-z]+)*$`)

// valid == len(name) >= 3 && len(name) <= 16 && namePattern.MatchString(name)
```

The length is a separate `len()` check rather than a `{2,15}` quantifier, and the
punctuation rule is expressed by requiring `[a-z]+` after every `'` or `-` rather
than by a lookahead. Go's `regexp` is RE2 and **has no lookahead assertions**, so
the obvious one-regexp phrasing does not compile; this phrasing enforces the same
rule and does. Because the pattern is ASCII-only, `len()` in bytes is also the
character count for anything that matches.

The second quantifier is `[a-z]*`, not `[a-z]+`, on purpose: `[a-z]+` would require
a lowercase letter between the initial capital and the first punctuation mark, and
so would reject `O'brien`. That is the one case worth stating in the ADR, because
it is the one a reimplementation gets wrong.

Unit-tested against an explicit accept/reject table. Rejected: `Ab` (too short —
passes the pattern, fails the length check), `A'` and `Ann'` (trailing
punctuation), `Ann--e` (adjacent punctuation), `-anne` (leading punctuation),
`ANNE` (interior uppercase), `Ann3` (digit), `Ann e` (space),
`Averyveryverylongname` (too long), and the Cyrillic homoglyph `Аnne`, which must
be rejected by the ASCII class rather than accidentally accepted. Accepted: `Abc`,
`Anne`, `O'brien`, `Jean-luc`.

**Normalization.** Strip `'` and `-`, then lowercase. `O'brien`, `Obrien` and
`Ob-rien` therefore collide, which is the intent: impersonation by punctuation is
the cheapest attack on a name system. The typed form is stored in
`auth.characters.name` for display; the normalized form in
`auth.characters.name_normalized` (`citext`) carries the unique index (ADR 0031).

**Blocklist.** A `chargen.name_blocklist` pack document of normalized substrings,
checked before uniqueness so a blocked name never gets reserved. M2 ships it
containing only impersonation prefixes — `gm`, `admin`, `sarnaut` — because real
moderation policy is not an M2 problem and a half-hearted profanity list is worse
than an honest empty one.

**Uniqueness.** Global across the shard. Not per account, not per faction. It is
enforced by the unique index and by nothing else: creation does a plain
`INSERT INTO auth.characters` and maps SQLSTATE `23505` to `NAME_TAKEN`. There is
no "check whether the name is free, then insert" path anywhere, because that is a
race by construction and the losing player gets a 500 instead of a clear answer.

**Reservation.** A courtesy for the creation form, not an authority. `POST
/v1/characters/name-checks` inserts into `auth.name_reservations` with
`reserved_until = now() + interval '5 minutes'` and the caller's `account_id`, so a
player typing into the form is not beaten to the name mid-keystroke. The subsequent
insert consumes and deletes the reservation if it belongs to the same account;
expired rows are ignored on read and swept lazily. If the reservation table and the
unique index ever disagree, the index wins.

**Deletion.** `auth.characters.deleted_at` soft-deletes. A deleted character's name
stays taken indefinitely in M2. Releasing names needs a grace period, a rename
story and an impersonation policy, and none of those are written.

### 4. Appearance and customization: M2 has none

This is a decision, not an omission.

There is no customization data model, no appearance columns on any table, and no
appearance fields on the wire. Every `chargen.league.warrior` character renders
with the model and material named in the option's `visual_ref`, identical for every
player.

The reason is that appearance is blocked on the art pipeline, not on the server:
character rigs, morph targets and material variants are converter work still in
flight. A schema invented now would be guessing at what that pipeline produces, and
would be migrated away before a single player used it.

When appearance does land it arrives additively — a `chargen.appearance` pack
document describing the axes, and a `shard.character_appearance` table owned by
`charstore` — with **no change to `auth.characters`**. Appearance is shard state
(it is display data that changes in-game via barbers and dyes), not account
identity, and putting it on the identity table now would put it in the wrong
service permanently.

### 5. Creation flow

1. Client `POST /v1/characters` to auth with the account session token, the name and
   a `chargen_option_id` (ADR 0030).
2. Auth validates in order: name shape → blocklist → the option exists and is
   `enabled` → insert, taking `NAME_TAKEN` from the unique index.
3. Auth returns `character_id`. The character now has **identity but no state**.
4. On the first `ticket.redeem` for that `character_id`, `charstore` finds no
   `shard.character_state` row and materializes one from the chargen table: spawn,
   level, stats, `starting_loadout` into `shard.character_inventory`,
   `starting_quests` into `shard.character_quests` — in the single transaction ADR
   0031 requires, committed **before** the spawn is granted.
5. Materialization is idempotent on `character_id`, so a redeem that races or
   retries produces one character with one set of starting gear, not two.

Splitting identity from state this way is what lets auth own account data and the
shard own game data without a distributed transaction between them: if step 4 never
happens, the player has a character that has never logged in, which is exactly what
the database says.

## Alternatives rejected

- **Two options, one per faction**, so faction selection is exercised early.
  Rejected: it doubles the chargen data, the starting loadout, the spawn point and
  the starter quest chain for a milestone whose entire purpose is proving one path
  end to end. Faction selection is a UI list and a data field; nothing about it gets
  easier by doing it twice at the point where the systems underneath are least
  stable.
- **A raceless, classless "test avatar"** with hardcoded stats. Faster to the demo.
  Rejected because M2 would then pass without the chargen table ever being loaded,
  and the table is the artifact that has to work. A milestone that green-lights the
  thing it did not test is worse than no milestone.
- **Spawn point from `WorldConfig`, loadout from a Go slice** — roughly where the
  code already is, and about fifteen minutes of work. Rejected because it
  guarantees the second class costs a server rebuild and a config change in
  lockstep, and because the compiler's cross-reference check (ADR 0006) cannot
  validate item ids that live in Go source.
- **A nullable `appearance jsonb` column now**, "since it costs nothing". Rejected:
  an unschemad nullable JSON column is where undocumented shapes accumulate, and
  the migration to remove one that three code paths have started writing is not
  free.

## Consequences

- `data-schemas` gains `chargen.schema.json` plus a hand-authored demo document, and
  the data compiler's cross-reference pass must resolve `starting_loadout` item ids,
  `starting_quests` quest ids, `starting_abilities` ability ids and `spawn.zone_id`
  — otherwise a typo in chargen data becomes a player who spawns naked in the void.
- The name regexp and the normalization function live in exactly one Go package,
  used by auth. The client may mirror them as a pre-submit hint; the server's answer
  is the one that counts, and the client must render `NAME_TAKEN` and
  `NAME_INVALID` from the server rather than assuming its check was sufficient.
- Restricting M2 names to ASCII is a real limitation with a real cost for a
  Russian-speaking playerbase, and it is temporary. Lifting it means NFKC
  normalization, confusable-script detection across Cyrillic and Latin, and a
  blocklist that works on normalized Unicode. That is M3 work and gets its own ADR;
  it is deferred because getting it wrong quietly (two visually identical names) is
  worse than deferring it loudly.
- Character deletion, rename and re-customization are all out of M2 scope and all
  interact with the name policy above. Each needs a decision before it ships.

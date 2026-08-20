# Spec 0002 — Loot (M2 vertical slice)

**Status**: Draft
**Milestone**: M2
**Owner**: mechanics
**Last reviewed**: 2026-08-20

## 1. Scope

Covers how a corpse turns into items and money: loot-table tree evaluation over
`And` / `Or` / `SingleItem` / `Money` nodes, the seeded RNG discipline that makes a
roll reproducible, kill credit and loot ownership, and splitting a rolled quantity
into inventory stacks.

Two different things are specified here and it matters which is which:

- **The tree shape and its chance semantics are derived from reference data.** The
  node set, the parallel `chances` array, and the min/max count fields are the shape
  found at
  `REF:World/LootTables/<Creature>/<Creature>(NN).xdb`, and §8 cites the structural
  evidence for how `And` and `Or` differ.
- **The evaluation policy is a curated SarnautCore decision.** Draw order, draw
  budget, seed derivation, count rounding, ownership window, and stack packing are
  invented. They are chosen to be deterministic and testable, not to reproduce retail
  drop rates.

Out of scope: retail loot fidelity. The 4,234 real loot trees are content that M2
does not consume, and the retail modifier chain that scales them (`MobKind.lootMod`,
`MobWorld.extra.lootDropModifier`) is queued behind a later spec per ADR 0003. M2
rolls one curated fixture table.

Also out of scope: the inventory container itself (owned by a later inventory spec;
this spec calls into it at rule 5.6), quest item credit
(see [`quests.md`](quests.md) §5.4), and vendor value.

## 2. Vocabulary

| Term | Meaning |
|---|---|
| **Loot table** | A tree with exactly one root node, loaded from one content record. |
| **Node** | One of `And`, `Or`, `SingleItem`, `Money`. The first two have children; the last two are leaves. |
| **Entry** | A child of a container node. Entries are ordered; the order is the file order. |
| **Chance** | `chances[i]`, the value positionally paired with `entries[i]`. Its meaning depends on the parent node's type (rules 5.3, 5.4). |
| **Stream** | A named, ordered source of `float64` values in `[0, 1)`. |
| **Draw** | One value taken from a stream. Draws are counted; the count is part of the contract. |
| **Roll** | One complete evaluation of one tree against one stream. |
| **Drop** | The result of a roll: a money amount and a list of `(item_id, count)` pairs. |

## 3. Constants

Prefixes as in [`combat.md`](combat.md) §3: `REF:` is the classic reference root,
`DATA:` is the SarnautCore data repository, `SERVER:` is the server repository.

| Constant | Value | Unit | Provenance |
|---|---|---|---|
| `SEED_HASH` | BLAKE3-256 | — | curated SarnautCore decision — already the project's content hash (`_source.blake3` in every extracted YAML) |
| `SEED_BYTES` | 16 | bytes | curated SarnautCore decision — the first 16 bytes of the hash become the two PCG64 words |
| `LOOT_OWNERSHIP_S` | 30 | seconds | curated SarnautCore decision — deliberately equal to `combat.md` `CORPSE_TIMER_S`, so ownership lasts exactly the corpse's lifetime |
| `MAX_TREE_DEPTH` | 8 | container levels | curated SarnautCore decision — a load-time guard; the deepest tree observed in reference data is 2 (§8) |
| `MAX_DRAWS_PER_ROLL` | 4096 | draws | curated SarnautCore decision — a runaway guard, not a design limit |

Per-item values read from content, not hard-coded:

| Field | Provenance |
|---|---|
| `stack_limit` | `DATA:` item YAML, field `stack_limit`; extracted from `<stackLimit>` in the source `ItemResource`. Example: `REF:Items/Mechanics/TrashHoof.xdb` has `<stackLimit>20</stackLimit>`. |

## 4. Model

```
LootTable
  id            string        // content id of the record
  root          Node
  max_depth     uint8         // computed at load, validated against MAX_TREE_DEPTH

Node = And | Or | SingleItem | Money

And
  entries       []Node
  chances       []float64     // len(chances) == len(entries), enforced at load

Or
  entries       []Node
  chances       []float64     // len(chances) == len(entries), enforced at load

SingleItem
  item_id       string        // resolved content id
  min_number    int32
  max_number    int32         // >= min_number, enforced at load

Money
  min_number    int32
  max_number    int32         // >= min_number, enforced at load

Stream
  NextFloat64() float64       // in [0, 1)
  Draws() int                 // monotonically increasing count

Drop
  money         int64
  items         []ItemGrant   // in draw order, duplicates not merged

ItemGrant
  item_id       string
  count         int32

Stack
  item_id       string
  count         int32         // 1 <= count <= item.stack_limit
```

`Money` carries no item reference. It is a leaf whose only fields are the two count
bounds, and it credits the character's purse rather than a bag slot (rule 5.6.1).

## 5. Rules

### 5.1 When a roll happens

1. A roll is performed **once**, at corpse creation (`combat.md` rule 5.9.4) — not
   when a player opens the corpse. The drop is fixed before it is observable, so
   re-opening a corpse cannot re-roll it and a disconnect mid-loot cannot change what
   is there.
2. The `Drop` is stored on the corpse. It is destroyed with the corpse at
   `despawn_tick`.

### 5.2 Stream discipline

1. Every roll uses a fresh stream seeded per corpse. Streams are never shared between
   corpses and never reused.
2. The seed is
   `SEED_HASH(world_seed ‖ zone_id ‖ spawn_slot_id ‖ death_server_tick ‖ killer_character_id)`,
   where `‖` is length-prefixed concatenation of the UTF-8 or little-endian binary
   encoding of each field. Length prefixing is required: without it,
   `("ab", "c")` and `("a", "bc")` hash to the same seed.
3. The first `SEED_BYTES` bytes of the digest are read as two little-endian `uint64`
   words `(lo, hi)` and become the PCG64 state (`math/rand/v2.NewPCG(lo, hi)` in Go).
4. `world_seed` is per-shard-instance configuration, not per-process-start. A shard
   restart must not change the seed a given corpse would produce, or the rule in 5.1.1
   buys nothing across restarts.
5. **The seed is logged.** Before the first draw, the server logs at info level:
   the corpse `entity_id`, the loot table `id`, and `loot_seed` as the full 32-hex
   digest prefix. Logging before the roll, not after, means a panic mid-roll still
   leaves behind the value needed to reproduce it.
6. After the roll, the server logs the draw count. A drop report plus its seed plus
   its draw count is enough to replay the roll offline with no server state.
7. The evaluator takes a `Stream` interface, not a seed. Production passes a
   PCG64-backed stream; tests pass a scripted stream (§6.1). Nothing in rules 5.3–5.5
   knows which one it has.

### 5.3 Evaluating an `And` node

An `And` node's `chances` are **independent probabilities**, one per entry. Zero, some,
or all entries may contribute.

1. For `i` from `0` to `len(entries) - 1`, in order:
   1. Draw `u`.
   2. If `u < chances[i]`, evaluate `entries[i]` (recursively, per 5.3–5.5) and append
      its results to the drop.
   3. If `u >= chances[i]`, skip `entries[i]`. **A skipped subtree consumes no draws.**
2. `chances[i] >= 1.0` therefore always passes, and `chances[i] <= 0.0` never does.
   A draw is still taken in both cases, so the draw budget does not depend on the
   chance values.
3. An `And` node itself consumes exactly `len(entries)` draws plus whatever its
   surviving subtrees consume.

### 5.4 Evaluating an `Or` node

An `Or` node's `chances` are a **probability distribution**. At most one entry is
selected.

1. Draw `u` exactly once.
2. Walk a running total `c = 0`. For `i` from `0` upward: `c += chances[i]`; if
   `u < c`, select `entries[i]` and stop.
3. If the walk completes without selecting — which happens when the chances sum to
   less than `1.0` — select nothing and contribute nothing. This is the guard, not an
   error: it lets a table express "nothing dropped" without a null entry.
4. Evaluate the selected entry recursively and append its results.
5. An `Or` node consumes exactly `1` draw plus whatever the selected subtree consumes.
   Unselected subtrees consume none.

### 5.5 Evaluating leaves

1. **`SingleItem`**: draw `u`, then
   `count = min_number + floor(u * (max_number - min_number + 1))`,
   clamped to `max_number`. Append `ItemGrant{item_id, count}`.
2. **`Money`**: draw `u`, then the same formula, and add the result to `drop.money`.
3. A leaf draws **even when `min_number == max_number`**. The draw is wasted in that
   case, and that is the point: the draw budget for a tree depends only on its shape,
   so tightening a count range in content does not shift every subsequent draw and
   silently change every other drop in the table.
4. The clamp in 5.5.1 is defensive. `u < 1.0` makes it unreachable for a correct
   stream; it exists so that a stream implementation that can return `1.0` produces a
   wrong count rather than an out-of-range one.
5. Duplicate `item_id`s across grants are **not** merged. Merging happens at
   inventory insertion (rule 5.6), where stack limits are known.

### 5.6 Awarding the drop

The client asks with `ClientMessage.loot_take` on the reliable channel and is answered
with `ServerMessage.loot_result` plus `ServerMessage.inventory_update`, or with
`ServerMessage.error` ([ADR 0026](../../adr/0026-wire-message-envelope.md)). The award
is committed to storage **before** either answer is written
([`../protocol/session.md`](../protocol/session.md) rule 5.7.4), so a crash in that
window replays as "you already have it" rather than as a lost item.

1. `drop.money` credits the looting character's purse directly. Money occupies no bag
   slot and has no stack limit.
2. Item grants are packed into stacks per rule 5.7, then inserted into the character's
   bag.
3. The insertion is **all or nothing**. If the bag cannot hold every stack, nothing is
   inserted, no money is credited, the corpse keeps the full drop, and the server
   returns `ErrBagFull`. Partial looting — take what fits, leave the rest — is a
   deliberate non-goal for M2 because it doubles the state machine for one slice.
4. On success the drop is cleared from the corpse. The corpse remains until its
   despawn tick (`combat.md` rule 5.9.5) but is now empty.

### 5.7 Stack splitting

1. Look up `stack_limit` for `item_id`. It is per-item content, never a constant.
2. `full = floor(count / stack_limit)`, `remainder = count mod stack_limit`.
3. Emit `full` stacks of `stack_limit`, then one stack of `remainder` if
   `remainder > 0`.
4. Required free slots is therefore `full + (1 if remainder > 0 else 0)`.
5. Merging into a partially filled existing stack of the same item is allowed and
   reduces the required slot count. M2 computes the requirement **after** accounting
   for merges, so a player with a half-full stack is not told the bag is full when it
   is not.
6. `stack_limit` of `1` degenerates correctly: `count` slots, no remainder stack.

### 5.8 Kill credit and loot ownership

1. Kill credit is decided by `combat.md` rule 5.9.2 — the character whose damage
   brought the mob to zero HP — and is copied onto the corpse at creation. This spec
   does not re-derive it.
2. Only the credited character may open the corpse. Any other character receives
   `ErrNotYourLoot`.
3. Ownership lasts `LOOT_OWNERSHIP_S`, which §3 sets equal to the corpse timer. The
   corpse therefore always despawns at the same instant ownership would lapse, and the
   free-for-all state is unreachable in M2. It is not implemented. When group play
   lands and ownership must genuinely expire before despawn, that becomes a new rule
   5.8.4 rather than a change to 5.8.3.
4. A character who disconnects and reconnects keeps ownership: ownership is keyed on
   `character_id`, not on the session or the world `entity_id`, both of which are
   destroyed by `zone.Leave` (`../protocol/session.md` §5.6).
5. Groups, raids, round-robin, need-before-greed, and master looter are all out of
   scope. M2 is a solo slice.

## 6. Worked examples

### 6.1 Rolling a three-level nested tree with a fixed stream

The tree below is `loot.fixture.m2-nested`, a **curated SarnautCore test fixture**.
Every number in it — chances, min/max counts, and the scripted draws — is part of the
fixture, not a spec constant and not a value taken from reference data. It exists
because reference data contains no tree this deep: 4,225 of the 4,234 reference tables
have a flat `And` root, 9 have an `Or` root, and exactly one nests at all (§8). A
depth-3 case has to be constructed to test the recursion.

**Fixture tree** (entry indices in brackets; `chances[i]` pairs with `entries[i]`):

```
[root]  And                                   chances = [1.00, 0.50, 0.25]
  [0]     Money            min=2  max=4
  [1]     Or                                  chances = [0.30, 0.70]
    [1.0]   SingleItem  trash-hoof   min=1 max=3
    [1.1]   And                             chances = [1.00, 0.40]
      [1.1.0]  SingleItem  heal-elixir  min=1 max=1
      [1.1.1]  Money                    min=10 max=10
  [2]     SingleItem  trash-hoof   min=1  max=1
```

Container depth is 3: `root` → `[1]` → `[1.1]`.

**Scripted stream**, consumed strictly in order:

```
u = [0.10, 0.90, 0.42, 0.55, 0.05, 0.33, 0.80, 0.99]
```

**Evaluation trace.** Each row is one draw. "Rule" is the rule that consumed it.

| # | `u` | Rule | Test | Outcome |
|---|---|---|---|---|
| 1 | 0.10 | 5.3.1 | `0.10 < 1.00` | pass → descend into `[0]` |
| 2 | 0.90 | 5.5.2 | `2 + floor(0.90 * (4 - 2 + 1)) = 2 + floor(2.7) = 4` | **money += 4** |
| 3 | 0.42 | 5.3.1 | `0.42 < 0.50` | pass → descend into `[1]` |
| 4 | 0.55 | 5.4.2 | `c = 0.30`, `0.55 < 0.30` is false; `c = 1.00`, `0.55 < 1.00` is true | select `[1.1]` (the nested `And`) |
| 5 | 0.05 | 5.3.1 | `0.05 < 1.00` | pass → descend into `[1.1.0]` |
| 6 | 0.33 | 5.5.1 | `1 + floor(0.33 * (1 - 1 + 1)) = 1 + floor(0.33) = 1` | **grant `heal-elixir` ×1** |
| 7 | 0.80 | 5.3.1 | `0.80 < 0.40` is false | skip `[1.1.1]`; its `Money` leaf draws nothing (5.3.1.3) |
| 8 | 0.99 | 5.3.1 | `0.99 < 0.25` is false | skip `[2]`; no leaf draw |

Note that `[1.0]` was never reached, so its `SingleItem` consumed no draw — rule 5.4.5.

**Expected output**, exactly:

```
drop.money  = 4
drop.items  = [ { item_id: heal-elixir, count: 1 } ]
stream.Draws() = 8
```

This is the unit test. Assert all three: the money, the item list, and the draw count.
The draw count is the assertion that catches an evaluator which silently changed its
draw budget — the money and item list alone would pass for several wrong
implementations.

**The seed-pinned companion test.** The scripted-stream test above pins behaviour;
it does not pin the PCG64 wiring in rule 5.2. A second test constructs the real
stream from a fixed tuple —
`world_seed = "sarnautcore-m2-test"`, `zone_id = "inst-league1"`,
`spawn_slot_id = "spawn.inst-league1.placement.000-020.1-2-server-objects.4"`,
`death_server_tick = 150`, `killer_character_id = 1` — rolls the same fixture, and
asserts against a golden value. The implementing pull request records that golden
value on first run and freezes it. A later change to `SEED_HASH`, to the field order
in 5.2.2, or to the word extraction in 5.2.3 then fails loudly instead of quietly
reshuffling every drop in the game.

### 6.2 Stack splitting and the bag-full path

An item with `stack_limit` 20 — the value `REF:Items/Mechanics/TrashHoof.xdb` carries
in `<stackLimit>`, and the value the extracted YAML carries as `stack_limit` — granted
in a count of 45:

| Step | Rule | Computation | Value |
|---|---|---|---|
| 1 | 5.7.2 | `floor(45 / 20)` | `full = 2` |
| 2 | 5.7.2 | `45 mod 20` | `remainder = 5` |
| 3 | 5.7.3 | two stacks of 20, one of 5 | `[20, 20, 5]` |
| 4 | 5.7.4 | `2 + 1` | **3 free slots required** |

With 3 or more free slots the insertion succeeds. With 2 free slots, rule 5.6.3
applies: nothing is inserted, no money is credited, the corpse keeps all 45 units plus
the money, and the client gets `ErrBagFull`.

With 2 free slots **and** an existing stack of 12 of the same item, rule 5.7.5 changes
the answer: 8 units top up the existing stack, leaving 37, which is `full = 1`,
`remainder = 17`, requiring 2 slots. The insertion succeeds. A test that only covers
the empty-bag case will not catch an implementation that skips 5.7.5.

## 7. Open questions and placeholders

### 7.1 `Or` chances that do not sum to 1

Rule 5.4.3 treats a short-falling distribution as "nothing drops". Every `Or` table in
reference data sums to exactly `1.0` (§8), so the branch is untested by real content
and its intent is inferred rather than observed. If a real table is ever found that
sums to less than 1, this rule is either confirmed or replaced by normalisation. The
two behaviours differ sharply, so the branch needs its own unit test either way.

### 7.2 The loot modifier chain is deferred

`MobKind` carries `lootMod` (`REF:Mechanics/MobKindTemplates/AE1Player.xdb` has
`0.33`, `Usual1Player.xdb` has `1`), and individual mobs carry
`extra.lootDropModifier` — the M2 zone's `demon-scout1-1` has one of type `NoLoot`
(`DATA:classic/zones/inst-league1/spawns/mobs/demon-scout.demon-scout1-1.yaml`). Where
in rules 5.3–5.5 those multipliers apply — scaling every chance, scaling only the
root, gating the roll entirely — is unestablished. M2 applies none of them. Note the
consequence: a mob marked `NoLoot` in content would still drop loot under this spec,
which is why M2's target mob is not one of them.

### 7.3 Count distribution is uniform by assumption

Rule 5.5.1 spreads counts uniformly over `[min, max]`. Nothing in the data says the
distribution is uniform; the fields only give bounds. Most reference leaves have
`min == max` anyway, which is why this has stayed cheap to leave open.

### 7.4 Partial looting

Rule 5.6.3's all-or-nothing insertion is a simplification with a real cost: a player
with one free slot cannot take the money either. When inventory becomes a real spec,
partial looting and a per-item take path replace 5.6.2–5.6.4.

## 8. Sources

Structural facts about `REF:World/LootTables/` (`REF:` =
`servers-clean/1.1.02.0/game/data/`). These are counts and shapes, not content:

- The directory holds **4,234** `.xdb` files, every one of which has
  `gameMechanics.constructor.schemes.item.LootTableResource` as its root element.
  Files are laid out as `<Creature>/<Creature>(NN).xdb`, e.g. `Boar/Boar(01).xdb`
  through `Boar/Boar(50).xdb`.
- The single `<table>` element under the root carries a `type` attribute naming the
  root node. Across the 4,234 files that attribute is
  `…item.LootTableAnd` in **4,225** files and `…item.LootTableOr` in **9**.
- Leaf node types appear as `type` attributes on `<Item>` elements inside `<entries>`:
  `…item.LootTableSingleItem` occurs **11,271** times and `…item.LootTableMoney`
  **1,474** times.
- `LootTableSingleItem` has `<item href=…>`, `<minNumber>`, `<maxNumber>`.
  `LootTableMoney` has `<minNumber>` and `<maxNumber>` and **no** `<item>` — the basis
  for rule 5.6.1 treating money as purse credit rather than a bag item.
- Container nodes carry `<entries>` and a separate `<chances>` element, each a list of
  `<Item>` children. They are positionally paired; nothing in the file links entry `i`
  to chance `i` other than ordinal position. This is the basis for the load-time
  length check in §4.
- **`And` chances are independent, not a distribution.** `Aviak/Aviak(01).xdb` is an
  `And` of a `Money` leaf and a `SingleItem` leaf with chances `1` and `0.742502`,
  summing to `1.742502`. A distribution cannot exceed 1, so these are per-entry
  probabilities. Rule 5.3.
- **`Or` chances are a distribution.** `RareBox/CannonBox.xdb` is an `Or` with 12
  entries whose chances sum to exactly `1.0`. `RareBox/RubyRareBox.xdb`'s outer `Or`
  has 11 chances that also sum to exactly `1.0`, and its inner `Or` has 14 chances of
  `0.0714` each, summing to `0.9996`. Rule 5.4.
- **Nesting is real but rare.** Of the 9 `Or`-rooted files, exactly one —
  `RareBox/RubyRareBox.xdb` — contains a second `Or` as an entry of the first. That is
  the deepest tree in the reference set, at container depth 2. `MAX_TREE_DEPTH` of 8
  in §3 is headroom, and §6.1's depth-3 fixture is curated for exactly this reason.
- `Boar/Boar(01).xdb` is the canonical minimal case: an `And` with one `SingleItem`
  entry and one chance.

Other reference paths:

- `Items/Mechanics/TrashHoof.xdb` — `<stackLimit>`, the source field behind
  `stack_limit`.
- `Mechanics/MobKindTemplates/AE1Player.xdb`, `Mechanics/MobKindTemplates/Usual1Player.xdb`
  — `lootMod`, for §7.2.

SarnautCore data-repository paths (`DATA:` = `data/`):

- `classic/items/mechanics/ac1bad-drink.yaml` — an extracted item showing the
  `stack_limit` field name and the `_source.blake3` field that §3 cites for `SEED_HASH`.
- `classic/zones/inst-league1/spawns/mobs/demon-scout.demon-scout1-1.yaml` —
  `extra.lootDropModifier` of type `NoLoot`, for §7.2.

Per ADR 0011, nothing here is quoted, paraphrased, or transliterated from decompiled
code. Every claim above is a count, a path, an element or attribute name, or an
arithmetic property of values at a cited path. No localization strings, art, or bulk
data tables appear in this document, and the 4,234 reference trees are described, not
reproduced.

## 9. Change log

| Date | Change |
|---|---|
| 2026-08-20 | Created for M2. |

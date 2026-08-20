# Spec 0003 — Quests (M2 vertical slice)

**Status**: Draft
**Milestone**: M2
**Owner**: mechanics
**Last reviewed**: 2026-08-20

## 1. Scope

Covers one quest, start to finish: the state machine a quest instance moves through,
the objective kinds M2 supports, how a kill turns into objective progress, how
prerequisites and required level are evaluated, and the reward grant — including what
happens when it cannot complete.

The **objective vocabulary and the quest record's field names are derived from
reference data**; §8 cites where each comes from. The **policy** — which of those
kinds M2 implements, the state machine's shape, the atomicity guarantee, and every
number in §3 — is a curated SarnautCore decision.

M2's quest is a **curated overlay definition**, not a retail quest. The reason is in
§7.1: the M2 starting zone contains no plain "kill N of a mob" quest to use. Inventing
one costs nothing legally and creates no spec debt, and it exercises the same code
path a real quest would.

Out of scope: dialogue trees, quest chains beyond a single prerequisite edge, quest
items that drop from mobs (M2's item objective is satisfied by a hand-placed grant),
`startImpacts` / `rewardImpacts` scripting, quest-attached loot tables, and every
non-solo quest type. All queued per ADR 0003.

## 2. Vocabulary

| Term | Meaning |
|---|---|
| **Quest definition** | The immutable content record: objectives, prerequisites, rewards, starter and finisher. |
| **Quest instance** | One character's progress against one definition. Holds the state and the counters. |
| **Objective** | One countable goal. A definition has zero or more, ordered; the order is the file order and is the counter index. |
| **Counter** | An objective's current progress, `0 <= counter <= limit`. |
| **Starter** | The NPC that offers the quest. |
| **Finisher** | The NPC that accepts the turn-in. Not necessarily the starter. |
| **Grant** | The reward transaction: experience, money, and items, applied together or not at all. |

## 3. Constants

Prefixes as in [`combat.md`](combat.md) §3.

| Constant | Value | Unit | Provenance |
|---|---|---|---|
| `QUEST_LOG_CAPACITY` | 25 | quest instances | curated SarnautCore decision |
| `TURN_IN_RANGE_M` | 5.0 | metres | curated SarnautCore decision — shorter than `combat.md` `ABILITY_RANGE_M` so a player must actually walk to the finisher |
| `M2_REQUIRED_LEVEL` | 1 | level | curated SarnautCore decision — the M2 quest is the first thing a character does |
| `M2_KILL_LIMIT` | 3 | kills | curated SarnautCore decision |
| `M2_ITEM_LIMIT` | 1 | items | curated SarnautCore decision |
| `M2_REWARD_EXPERIENCE` | 8 | experience | curated SarnautCore decision |
| `M2_REWARD_MONEY` | 0 | money | curated SarnautCore decision |
| `M2_REWARD_HONOR` | 0 | honor | curated SarnautCore decision |
| `M2_REWARD_ITEM_KINDS` | 2 | distinct items | curated SarnautCore decision — two, not one, so that §6.2's net-slot arithmetic has a non-zero answer and the bag-full path is reachable |
| `M2_REWARD_ITEM_COUNT` | 1 | items per kind | curated SarnautCore decision |

Per-quest and per-item values read from content, not hard-coded: `level`,
`required_level`, `prerequisites[]`, `objectives[].limit`, `rewards.experience`,
`rewards.money`, `rewards.honor`, and each reward item's `stack_limit`. Field names
are the extracted YAML's; §8 gives the paths.

## 4. Model

```
QuestDefinition
  id                string
  level             int32
  required_level    int32          // 0 means "no level gate"
  prerequisites     []Prerequisite
  objectives        []Objective    // may be empty
  starter_id        string         // mob content id
  finisher_id       string         // mob content id; may differ from starter_id
  rewards           Rewards
  can_cancel        bool

Prerequisite
  quest_id          string
  required_status   enum { Finished }   // M2 supports only this value

Objective
  kind              enum { CountKill, CountItem }
  limit             int32
  targets           []string       // content ids; any one of them counts
  internal          bool           // hidden from the quest log
  show_count        bool           // display "n / limit" in the log

Rewards
  experience        int64
  money             int64
  honor             int64
  items             []ItemReward

ItemReward
  item_id           string
  count             int32

QuestInstance
  character_id      uint64
  quest_id          string
  state             QuestState
  counters          []int32        // len == len(definition.objectives)
  accepted_at_tick  uint64
```

## 5. Rules

### 5.1 States

| State | Meaning |
|---|---|
| `unavailable` | The character does not meet the gates in rule 5.3. No instance exists. |
| `offered` | Gates pass and the starter will hand it over. No instance exists yet. |
| `accepted` | The grant of the quest itself is committed. Every counter is 0. |
| `in-progress` | At least one counter is above 0 and at least one is below its limit. |
| `completable` | Every counter has reached its limit. |
| `turned-in` | Terminal. Rewards granted. |
| `abandoned` | Terminal for this instance. The character may re-enter `offered`. |

`accepted` and `in-progress` are separate states rather than one, because they differ
in what the client shows and in what abandoning costs. Collapsing them would make
"has this player touched this quest yet" unanswerable without inspecting counters.

### 5.2 State transition table

| # | From | Event | Guard | To | Side effects |
|---|---|---|---|---|---|
| T1 | `unavailable` | gate re-evaluation (level up, or a prerequisite reaching `turned-in`) | rule 5.3 passes | `offered` | Starter shows the offer marker. |
| T2 | `offered` | gate re-evaluation | rule 5.3 fails | `unavailable` | Offer marker cleared. Only reachable by losing a level; M2 has no level loss, so this edge is specified but unreachable. |
| T3 | `offered` | `QuestAccept` from the starter | within `TURN_IN_RANGE_M` of the starter; quest log below `QUEST_LOG_CAPACITY`; rule 5.3 still passes | `accepted` | Instance created, counters zeroed, `accepted_at_tick` set. |
| T4 | `offered` | `QuestAccept` | any guard in T3 fails | `offered` | Rejected with `ErrQuestUnavailable`, `ErrQuestLogFull`, or `ErrOutOfRange`. No instance created. |
| T5 | `accepted` | objective progress (rule 5.4 or 5.5) | some counter still below its limit | `in-progress` | Counter incremented; progress event sent. |
| T6 | `accepted` | objective progress | every counter now at its limit | `completable` | Counters incremented; completion event sent. |
| T7 | `accepted` | instance created with zero objectives | `len(objectives) == 0` | `completable` | Applied immediately as part of T3, in the same transaction. See rule 5.6. |
| T8 | `in-progress` | objective progress | some counter still below its limit | `in-progress` | Counter incremented. |
| T9 | `in-progress` | objective progress | every counter now at its limit | `completable` | Completion event sent. |
| T10 | `completable` | objective regression (rule 5.5.4) | a counter drops below its limit | `in-progress` | Only `CountItem` objectives can regress. |
| T11 | `completable` | `QuestTurnIn` at the finisher | within `TURN_IN_RANGE_M` of the finisher; grant succeeds (rule 5.7) | `turned-in` | Rewards applied; item objectives consumed; gate re-evaluation runs for every quest listing this one as a prerequisite. |
| T12 | `completable` | `QuestTurnIn` | grant fails (rule 5.7.6) | `completable` | Nothing applied. `ErrBagFull` returned. The instance is unchanged, byte for byte. |
| T13 | `completable` | `QuestTurnIn` | out of range, or the target is not the finisher | `completable` | `ErrOutOfRange` or `ErrWrongNpc`. |
| T14 | `accepted`, `in-progress`, `completable` | `QuestAbandon` | `definition.can_cancel` is true | `abandoned` | Instance destroyed. Item objectives' items removed if the definition says so (rule 5.5.5). |
| T15 | `accepted`, `in-progress`, `completable` | `QuestAbandon` | `can_cancel` is false | unchanged | `ErrQuestCannotBeCancelled`. Retail's starting quests set this flag false, and M2's curated quest follows. |
| T16 | `abandoned` | gate re-evaluation | rule 5.3 passes | `offered` | The quest can be taken again from scratch. |
| T17 | `turned-in` | anything | — | `turned-in` | Terminal. A second `QuestTurnIn` returns `ErrQuestAlreadyComplete`. |

No transition leaves `turned-in`. Quest repetition, dailies, and resets are out of
scope; when they arrive they add a new state rather than reopening this one.

The client-driven triggers are the `ClientMessage` oneof cases of
[ADR 0026](../../adr/0026-wire-message-envelope.md) — `quest_accept` carries
`QuestAccept`, `quest_turn_in` carries `QuestTurnIn` — and both travel on the reliable
channel. **`QuestAbandon` has no case in the M2 message set**: T14 and T15 are
specified and unreachable from a client until one is added, which is an additive change
that does not bump `ProtocolVersion` (ADR 0027). See
[`../protocol/session.md`](../protocol/session.md) §7.6.

### 5.3 Prerequisite and required-level evaluation

Evaluated whenever the character levels up, whenever any quest reaches `turned-in`,
and on demand when a player interacts with a starter. Never on a timer.

1. Fail if an instance for this quest already exists in any state other than
   `abandoned`.
2. Fail if `definition.required_level > 0` and `character.level < required_level`.
   A `required_level` of `0` means no gate; retail's first starting quest uses exactly
   that (§8).
3. For each `prerequisite`: fail unless the character has an instance of
   `prerequisite.quest_id` in state `turned-in`. `required_status` is `Finished` in
   every reference record M2 consumes, and M2 rejects any other value at content-load
   rather than at evaluation time.
4. Prerequisites are conjunctive: all must pass. No reference record in the M2 zone has
   more than one, so a disjunctive form has never been observed and is not supported.
5. There is no upper level bound, no faction gate, no race gate, and no class gate in
   M2, even though reference quest records carry fields that imply all four exist.
   §7.2.
6. Pass ⇒ `offered`. Fail ⇒ `unavailable`.

### 5.4 Kill-credit attribution

1. `combat.md` rule 5.9.3 emits
   `MobKilled{victim_content_id, victim_level, killer_character_id, zone_id, server_tick}`.
   The quest module consumes it; it does not observe combat state directly.
2. Credit goes to `killer_character_id` and to no one else. Group credit, assist
   credit, tap rules, and level-difference cutoffs are all out of scope — M2 is a solo
   slice and `combat.md` rule 5.9.2 already reduced credit to a single character.
3. For each of that character's instances in `accepted` or `in-progress`:
   1. For each objective of kind `CountKill` whose `targets` contains
      `victim_content_id`:
   2. Skip if `counter == limit` — already satisfied, and over-counting would make
      T10's regression check ambiguous.
   3. `counter = min(counter + 1, limit)`.
4. One kill increments at most one counter **per objective**, but a kill may increment
   counters in several different quests. That is intended.
5. Evaluate T5/T6 or T8/T9 once, after all counters for this event have been updated —
   not once per counter. Otherwise a kill that satisfies the last two objectives at
   once would emit two events and briefly report a false intermediate state.
6. The event is processed on the zone's tick, in the same order kills occurred.
   Progress is therefore deterministic given a deterministic kill order, which matters
   for the replay-a-seed workflow in `loot.md` §5.2.

### 5.5 Objective kinds

M2 supports two of the three kinds present in reference data.

1. **`CountKill`** (`quest-count-kill` in extracted YAML). Advanced by rule 5.4.
   `targets` holds content ids; any one of them counts. In reference data the targets
   of the M2 zone's only kill objective are interactive chests rather than mobs, which
   is why M2's own quest is curated — see §7.1. The rule is the same either way: the
   destroyed entity's content id is matched against `targets`.
2. **`CountItem`** (`quest-count-item`). The counter tracks how many of the target item
   the character currently holds, summed across stacks.
3. `CountItem` is recomputed on every inventory change to the tracked item, not
   incremented. This is why it is the only kind that can regress.
4. **Regression**: dropping, selling, or destroying a tracked item lowers the counter
   and can move a `completable` instance back to `in-progress` (T10). A `CountKill`
   counter never decreases.
5. If the definition marks the item objective as removed on abandon — reference records
   carry this as `extra.removeOnAbandon` — T14 destroys the tracked items along with
   the instance.
6. **`quest-count-special` is not supported.** It is by far the most common kind in the
   M2 zone (§8), and it works by having quest script impacts fire against a named
   counter id, which requires the impact system. M2 rejects any definition containing
   one at content-load, loudly, rather than silently loading a quest that can never be
   completed.
7. Objectives marked `internal` are evaluated normally and hidden from the client's
   quest log. `show_count` controls whether the log renders `n / limit` or a plain
   incomplete marker. Neither flag affects rules 5.4–5.6.
8. An objective with `limit == 0` is satisfied on creation. Reference data uses this
   together with `internal: true` for objectives whose completion is driven entirely by
   script. M2 treats such an objective as immediately at its limit, which means a quest
   whose objectives are all `limit == 0` behaves like the zero-objective case in T7.

### 5.6 The zero-objective quest

A definition may have no `objectives` key at all; the first quest in the M2 starting
zone is exactly that shape (§8). Such a quest is accepted and immediately
`completable` — the player's only task is to walk to the finisher.

1. T7 applies inside T3's transaction. The instance is never observably in state
   `accepted` with zero objectives.
2. This is not a special case in the code. `every counter has reached its limit` is
   vacuously true over an empty counter list, so T6's guard already covers it. Rule
   5.6.1 exists to state that the two transitions commit together, which is the part
   that is not automatic.

### 5.7 The reward grant

The turn-in is one transaction. Either everything in it happens or nothing does.

1. Preconditions, checked in this order and all before any mutation: state is
   `completable`; the interacted NPC is `definition.finisher_id`; the distance to it is
   at most `TURN_IN_RANGE_M`.
2. **Compute the inventory requirement before touching anything.** For each
   `ItemReward`, apply `loot.md` rule 5.7 — split `count` by the item's `stack_limit`,
   subtract what merges into existing partial stacks — and sum the required free slots
   across all rewards. Items consumed by `CountItem` objectives free slots and are
   counted in this arithmetic, so a quest that takes 5 of something and gives 1 thing
   back nets a slot free rather than needing one.
3. **Fail fast**: if required free slots exceed available free slots, stop here.
   Nothing has been mutated yet, so there is nothing to roll back. Return `ErrBagFull`.
   T12.
4. Otherwise, in one transaction: consume the `CountItem` objectives' items; insert the
   reward item stacks; credit `rewards.experience`, `rewards.money`, and
   `rewards.honor`; set the instance state to `turned-in`.
5. Commit. Then, outside the transaction, run rule 5.3 for every definition listing
   this quest as a prerequisite, and emit the client's quest-completed event.
6. **Failure mode.** Any failure inside step 4 — a concurrent insert filling the last
   slot between the check in step 2 and the insert in step 4, a storage error, a
   context cancellation — rolls the whole transaction back. The character keeps the
   quest items, gains no experience, no money, no honor, and no reward items, and the
   instance stays `completable`. The player can walk away and try again. There is no
   partial turn-in and no state in which experience was granted but the quest is still
   open.
7. Step 2's check and step 4's insert must run under the same lock on the character's
   inventory, or the race in 5.7.6 is not a rare failure but the normal outcome of a
   double-click. The check being cheap is not a reason to do it outside the lock.
8. Experience is credited as a number. What that number does — level-ups, the level
   curve — is `combat.md` §7.1's open question and is not resolved here.

## 6. Worked example

### 6.1 The M2 quest, end to end

`quest.overlay.m2.slay-earth-elementals` is a **curated SarnautCore overlay
definition**. Every value in the definition below is a §3 constant. The two content
references are cited in §8. The distances appearing in the trace are chosen scenario
inputs, not constants: what matters about them is only which side of
`TURN_IN_RANGE_M` they fall on.

| Field | Value |
|---|---|
| `required_level` | `M2_REQUIRED_LEVEL` = 1 |
| `prerequisites` | none |
| `objectives[0]` | `CountKill`, `limit` = `M2_KILL_LIMIT` = 3, `targets` = [`mob.inst-league1.earth-elemental.earth-elemental2-1`] |
| `objectives[1]` | `CountItem`, `limit` = `M2_ITEM_LIMIT` = 1, `targets` = [`item.quest-items.inst-league1.heal-elixir.heal-elixir-item-resource`] |
| `rewards` | experience `M2_REWARD_EXPERIENCE` = 8, money 0, honor 0, and `M2_REWARD_ITEM_KINDS` = 2 distinct items at `M2_REWARD_ITEM_COUNT` = 1 each |
| `can_cancel` | false |

The mob id is the M2 target mob from `combat.md` §6.1. The quest-item id is the item
already referenced by a real `quest-count-item` objective in the zone (§8).

Trace:

| Step | Event | Transition | Counters | State |
|---|---|---|---|---|
| 1 | Character created at level 1 | T1 | — | `offered` |
| 2 | `QuestAccept` at the starter, 3.2 m away | T3 | `[0, 0]` | `accepted` |
| 3 | First elemental dies (`combat.md` §6.1: six casts, t = 5000 ms) | T5 | `[1, 0]` | `in-progress` |
| 4 | Second elemental dies | T8 | `[2, 0]` | `in-progress` |
| 5 | Heal elixir enters the bag | T8 | `[2, 1]` | `in-progress` |
| 6 | Player sells the elixir | T8, via 5.5.3 | `[2, 0]` | `in-progress` |
| 7 | Elixir reacquired | T8 | `[2, 1]` | `in-progress` |
| 8 | Third elemental dies | T9 | `[3, 1]` | `completable` |
| 9 | Fourth elemental dies | none, via 5.4.3.2 | `[3, 1]` | `completable` |
| 10 | `QuestTurnIn` at the finisher, 12 m away | T13 | `[3, 1]` | `completable` (`ErrOutOfRange`) |
| 11 | `QuestTurnIn` at 4.1 m, bag full | T12 | `[3, 1]` | `completable` (`ErrBagFull`) |
| 12 | Bag cleared, `QuestTurnIn` at 4.1 m | T11 | `[3, 1]` | `turned-in` |

Step 9 is the assertion that `CountKill` clamps. Step 6 is the assertion that
`CountItem` regresses. Step 11 is the assertion that a failed grant leaves the
instance untouched — the test compares the whole instance before and after, not just
the state field.

### 6.2 The bag-full arithmetic at step 11

The reward is `M2_REWARD_ITEM_KINDS` = 2 distinct items, each with `stack_limit` 1 —
the shape every equipment record in `DATA:classic/items/quest-rewards/` has. The quest
also consumes the character's single heal elixir. The bag is full: 0 free slots.

| Step | Rule | Computation | Value |
|---|---|---|---|
| 1 | 5.7.2 via `loot.md` 5.7 | reward A: `floor(1 / 1) = 1` full stack, remainder 0 | 1 slot |
| 2 | 5.7.2 via `loot.md` 5.7 | reward B: same | 1 slot |
| 3 | 5.7.2 | gross requirement | 2 slots |
| 4 | 5.7.2 | the consumed elixir is the character's last one, so its stack disappears | 1 slot freed |
| 5 | 5.7.2 | net requirement `2 - 1` | **1 slot** |
| 6 | 5.7.3 | `1 > 0` available | **`ErrBagFull`**, nothing mutated (T12) |

Two things are being asserted here, and they fail in opposite directions:

- Computing the **gross** requirement (2) instead of the net (1) rejects turn-ins that
  should succeed. Free one slot and this same turn-in must succeed, because the net
  requirement drops to 1 and one slot is available — that is §6.1 step 12.
- Skipping the check and letting the insert fail halfway corrupts the bag: the elixir
  is consumed, reward A is inserted, reward B is not, and the quest is marked
  `turned-in`. Rule 5.7.6 exists to make that outcome unreachable.

The single-reward case is worth a test too, precisely because its net requirement is
zero: one reward item in, one consumed stack out, so a completely full bag still
turns in successfully.

## 7. Open questions and placeholders

### 7.1 Why M2's quest is curated rather than a real one

The M2 starting zone's 21 extracted quest definitions contain 12 objectives: 10 of
kind `quest-count-special`, one `quest-count-item`, and one `quest-count-kill`. The
single kill objective targets a `ChestResource` — an interactive object — with
`limit: 0` and `internal: true`, i.e. it is script-driven, not a kill count. There is
no "kill N mobs" quest in the zone to implement.

Implementing `quest-count-special` first would mean implementing the impact system,
which is a much larger spec and is not on M2's critical path (ADR 0003). So M2 ships a
curated overlay quest that uses the real objective vocabulary with a mob target. When
the impact system lands, the real quests load against the same state machine and the
overlay quest is deleted. Nothing in §5 is retail-shaped in a way that would need
undoing.

### 7.2 Race-, class-, and reputation-conditional rewards

Reference quest records carry a `rewardImpacts` structure whose entries are typed
impacts — buff attach and detach among them — and the surrounding data model has
race, class, honor, and reputation gates. Conditional rewards ("a plate wearer gets
the plate version") are the obvious use. M2 grants a flat reward list with no
conditions. Resolving this requires the impact system and the class/race model, both
queued per ADR 0003.

### 7.3 Quest-attached loot tables

Quest records carry a `lootTable` field, `null` in the M2 zone's first quest. Whether
a quest reward can be rolled from a loot table rather than listed — and if so whether
`loot.md`'s evaluator is reused verbatim — is unestablished. Rule 5.7.2 assumes a
fixed list.

### 7.4 Starter and finisher are not the same NPC

Reference data has quests whose finisher differs from the starter, so §4 models them
separately and T11 checks the finisher specifically. What M2 does not specify is what
happens when the finisher is dead, despawned, or in a different zone at turn-in time.
The M2 quest's finisher is a stationary NPC, so the case does not arise.

### 7.5 Objective ordering as counter identity

`counters[i]` is bound to `objectives[i]` by position. Reordering objectives in content
silently reassigns live counters. A content-hash check on the definition at load, or
stable objective ids, would fix this. M2's single quest never changes, so it is
deferred, but it is a real data-migration hazard the first time a quest is edited in
production.

## 8. Sources

SarnautCore data-repository paths (`DATA:` = `data/`). All 21 quest definitions cited
here live under `classic/zones/inst-league1/quests/`:

- `quest-1-10.yaml` — a definition with **no** `objectives` key at all (rule 5.6), plus
  the field names `level`, `required_level` (value `0`, the no-gate case in rule
  5.3.2), `starter`, `finisher`, `rewards.experience` / `.money` / `.honor`,
  `flags.can_cancel`, `extra.lootTable` (null, §7.3), and `extra.rewardImpacts` (§7.2).
- `quest-2-10.yaml` — `prerequisites[].quest.id` with `status: Finished` (rule 5.3.3);
  a starter and finisher that are **different** NPCs (§7.4); two objectives of kind
  `quest-count-special` with `limit`, `internal`, and `show_count` fields.
- `quest-2-20.yaml` — the zone's only `quest-count-kill` objective, with `limit: 0`,
  `internal: true`, and a `ChestResource` target (§7.1); and the zone's only
  `quest-count-item` objective, with `extra.removeOnAbandon: 'true'` (rule 5.5.5) and
  the heal-elixir item id used in §6.1.
- Across all 21 files, objective kinds occur as: `quest-count-special` ×10,
  `quest-count-kill` ×1, `quest-count-item` ×1. This distribution is the whole
  argument in §7.1.
- `classic/items/quest-rewards/` — every record in this directory is equipment with
  `stack_limit: 1`, the shape §6.2 assumes for the reward item.
- `classic/zones/inst-league1/spawns/mobs/earth-elemental.earth-elemental2-1.yaml` —
  the mob content id used as §6.1's kill target.

Reference-root paths (`REF:` = `servers-clean/1.1.02.0/game/data/`):

- `World/Quests/InstLeague1/Quest_1_10/Quest_1_10.xdb` and its siblings — the
  `gameMechanics.constructor.schemes.quest.QuestResource` records the `DATA:` files
  above were extracted from. Cited for provenance of the field names only.

Sibling specs: [`combat.md`](combat.md) rules 5.9.2 and 5.9.3 for kill credit,
[`loot.md`](loot.md) rule 5.7 for stack splitting.

Per ADR 0011, nothing here is quoted, paraphrased, or transliterated from decompiled
code. Every claim is a path, a field or kind name, a count of records, or a value at a
cited path. No localization strings, art, or bulk data tables appear in this document.

## 9. Change log

| Date | Change |
|---|---|
| 2026-08-20 | Created for M2. |

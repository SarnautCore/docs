# Spec 0001 — Combat (M2 vertical slice)

**Status**: Draft
**Milestone**: M2
**Owner**: mechanics
**Last reviewed**: 2026-08-20

## 1. Scope

Covers the minimum combat loop the M2 vertical slice needs: entity health and level,
one instant-cast damaging ability, target selection and range validation, the global
cooldown, the damage formula, aggro/chase/leash, and death → corpse → respawn.

Out of scope, deliberately:

- **Retail fidelity.** The rules here are invented. They are not a reconstruction of
  Allods Online combat math. Real retail math (`MobKind` multiplier chains, spell
  effect graphs, stat conversion, resistances, crit, hit tables) is queued behind
  later specs per ADR 0003, which allows only critical-path work per milestone.
- **Mob damage output.** M2's target mob aggros, chases, and dies. It does not hit
  back. Two-sided combat, threat tables with more than one attacker, and player death
  are the next spec.
- **Loot.** Owned by [`loot.md`](loot.md); this spec hands off at rule 5.9.4.
- **Quest kill credit.** Owned by [`quests.md`](quests.md); this spec emits the event
  it consumes (rule 5.9.3).
- **Transport.** Owned by [`../protocol/session.md`](../protocol/session.md).

Because the rules are invented rather than ported, they carry no legal exposure and
no spec debt: replacing them later is a rewrite of §3 and §5 of one document, not an
unpicking of derived work. Where a real retail value was available and cheap to use,
§3 uses it and cites it; everywhere else §3 says **curated SarnautCore decision**.

## 2. Vocabulary

| Term | Meaning |
|---|---|
| **Combatant** | A world entity with `level`, `max_hp`, `current_hp`. Players and mobs both. |
| **Caster** | The combatant initiating an ability use. In M2, always a player. |
| **Target** | The combatant an ability resolves against. |
| **GCD** | Global cooldown. One timer per caster, shared by all abilities. |
| **Anchor** | A mob's spawn position, used as the origin for the leash radius. |
| **Threat** | Cumulative damage a mob has taken from a given character. Decides the mob's chase target. |
| **Kill credit** | The character attributed with the killing blow. Input to loot and quests. |
| **Corpse** | The non-combatant, lootable remains of a dead mob. |

## 3. Constants

Naming the source root prefixes used below:

- `REF:` — the classic reference root, `servers-clean/1.1.02.0/game/data/` (ADR 0009).
- `DATA:` — the SarnautCore `data` repository root.
- `SERVER:` — the SarnautCore `server` repository root.

| Constant | Value | Unit | Provenance |
|---|---|---|---|
| `HP_BASE` | 100 | hit points | curated SarnautCore decision — the level curve is not in content and is owned by this project, settled in §7.1 |
| `HP_PER_LEVEL` | 20 | hit points | curated SarnautCore decision — see §7.1 |
| `ARMOR_BASE` | 10 | armor | curated SarnautCore decision |
| `ARMOR_PER_LEVEL` | 20 | armor | curated SarnautCore decision |
| `ATTACK_POWER_BASE` | 12 | attack power | curated SarnautCore decision |
| `ATTACK_POWER_PER_LEVEL` | 2 | attack power | curated SarnautCore decision |
| `ABILITY_BASE_DAMAGE` | 18 | damage | curated SarnautCore decision |
| `ATTACK_POWER_COEFF` | 0.5 | damage per attack power | curated SarnautCore decision |
| `ARMOR_SOFTCAP_BASE` | 100 | armor | curated SarnautCore decision |
| `ARMOR_SOFTCAP_PER_LEVEL` | 50 | armor | curated SarnautCore decision |
| `GLOBAL_COOLDOWN_MS` | 1000 | milliseconds | `REF:Mechanics/GameRoot/SpellRoot.xdb` — `<globalCooldown>` |
| `TICK_INTERVAL_MS` | 33.333333 | milliseconds | `SERVER:config.example.yaml` — `world.tick_interval` |
| `ABILITY_RANGE_M` | 10.0 | metres | curated SarnautCore decision — chosen to match the modal `<radius>` value observed across `REF:Mechanics/Spells/Creatures/`, but the retail unit is unverified (§7.2) |
| `RANGE_TOLERANCE_M` | 0.5 | metres | curated SarnautCore decision — absorbs client/server position drift over one snapshot interval |
| `AGGRO_RADIUS_M` | 12.0 | metres | curated SarnautCore decision — see §7.2 |
| `LEASH_RADIUS_M` | 40.0 | metres | curated SarnautCore decision |
| `CHASE_SPEED_MULTIPLIER` | 1.0 | ratio | curated SarnautCore decision — the mob chases at its own `walk_speed` |
| `CORPSE_TIMER_S` | 30 | seconds | curated SarnautCore decision |
| `RESPAWN_DELAY_MIN_MS` | 10000 | milliseconds | `REF:Maps/Inst_LeagueStart/SpawnTables/InstLeague1/AirElemental.(MobSpawnTable).xdb` — `spawnTime` of type `TimeRange`, `<range><min>` |
| `RESPAWN_DELAY_MAX_MS` | 14000 | milliseconds | same file — `<range><max>` |

Per-entity values read from content, not hard-coded:

| Field | M2 target mob's value | Provenance |
|---|---|---|
| `level_min` / `level_max` | 2 / 2 | `DATA:classic/zones/inst-league1/spawns/mobs/earth-elemental.earth-elemental2-1.yaml` |
| `walk_speed` | 2.0 | same file |
| `hp_mod` | absent → defaults to 1.0 | the referenced `MobKind` carries no `hpMod`; its prototype does (§7.1) |

## 4. Model

```
Combatant
  entity_id        uint64
  level            uint8        // >= 1
  max_hp           int32        // derived, rule 5.1.1
  current_hp       int32        // 0 <= current_hp <= max_hp
  armor            int32        // derived, rule 5.1.2
  attack_power     int32        // derived, rule 5.1.3
  gcd_ready_tick   uint64       // server tick at which the next ability may start
  alive            bool

MobState
  anchor           Vec3         // spawn position
  home_zone        string
  ai_state         enum { Idle, Aggro, Returning, Dead }
  threat           map[character_id]int64
  walk_speed       float32      // from content
  aggro_target     uint64       // 0 when none

Corpse
  entity_id        uint64       // reuses the dead mob's id for the loot handshake
  despawn_tick     uint64
  kill_credit      character_id
  spawn_slot_id    string       // which placement respawns this mob

AbilityUse
  caster_id        uint64
  target_id        uint64
  client_tick      uint64       // advisory; the server uses its own tick
```

`Vec3` is the existing world vector; `Z` is vertical (`SERVER:internal/world/world.go`).

## 5. Rules

Position, distance, and time are all evaluated on the server's authoritative state at
the tick the ability is processed. Client-supplied positions and ticks are advisory.

### 5.1 Derived stats

1. `max_hp(level, hp_mod) = round_half_up((HP_BASE + HP_PER_LEVEL * (level - 1)) * hp_mod)`.
   `hp_mod` defaults to `1.0` when the content record omits it.
2. `armor(level) = ARMOR_BASE + ARMOR_PER_LEVEL * (level - 1)`.
3. `attack_power(level) = ATTACK_POWER_BASE + ATTACK_POWER_PER_LEVEL * (level - 1)`.
4. Stats are computed once at spawn and cached on the combatant. M2 has no buffs, no
   gear deltas, and no level-ups mid-combat, so nothing recomputes them.
5. A mob's level is drawn once at spawn, uniformly from `[level_min, level_max]`
   inclusive, from the zone's spawn stream (see `loot.md` §5.2 for stream discipline).
   The M2 target mob has `level_min == level_max == 2`, so the draw is degenerate but
   is still performed, to keep the stream position stable if the content changes.

### 5.2 Target selection

1. The client sends `ClientMessage{ability_use: AbilityUse{caster_id, target_id}}` on
   the **reliable** channel — combat is never a datagram
   ([ADR 0026](../../adr/0026-wire-message-envelope.md)). `caster_id` is ignored; the
   server uses the session's own `entity_id`
   ([`../protocol/session.md`](../protocol/session.md) rule 5.2.6).
2. Reject with `ErrNoTarget` if `target_id == 0` or names no entity in the caster's zone.
3. Reject with `ErrInvalidTarget` if the target is not a `Combatant` — corpses, static
   props, and the caster itself are all rejected here. M2 has no self-targeted or
   friendly abilities.
4. Reject with `ErrTargetDead` if `target.alive == false`.
5. Reject with `ErrInvalidTarget` if the target's faction is not hostile to the caster.
   M2 resolves this from the mob's `faction` reference; `Wild` is hostile to players,
   `Neutral` and `NeutralFriendly` are not.

### 5.3 Range validation

1. `d = euclidean_distance(caster.position, target.position)`, using all three axes.
2. Reject with `ErrOutOfRange` if `d > ABILITY_RANGE_M + RANGE_TOLERANCE_M`.
3. There is no line-of-sight check in M2. Collision and terrain queries do not exist
   yet (`SERVER:internal/world/world.go` defers Z resolution), so a LoS check would be
   built on a surface that is not there. Recorded in §7.3.

### 5.4 Global cooldown

1. Reject with `ErrOnCooldown` if `now_tick < caster.gcd_ready_tick`.
2. On a use that passes 5.2–5.4, set
   `caster.gcd_ready_tick = now_tick + ceil(GLOBAL_COOLDOWN_MS / TICK_INTERVAL_MS)`.
   With the constants in §3 that is `ceil(1000 / 33.333333) = 30` ticks.
3. The GCD is set **before** damage resolution, so an ability that kills its target
   still consumes the cooldown.
4. The ability is instant: there is no cast bar, no cast-time timer, and therefore no
   interrupt or movement-cancel path in M2.
5. There is no per-ability cooldown in M2. `GLOBAL_COOLDOWN_MS` is the only gate.

### 5.5 Damage formula

Named inputs: `attack_power` (caster, rule 5.1.3), `target_armor` (target, rule 5.1.2),
`caster_level`.

1. `raw = ABILITY_BASE_DAMAGE + ATTACK_POWER_COEFF * attack_power`
2. `softcap = ARMOR_SOFTCAP_BASE + ARMOR_SOFTCAP_PER_LEVEL * caster_level`
3. `mitigation = target_armor / (target_armor + softcap)`
4. `mitigated = raw * (1 - mitigation)`
5. `damage = round_half_up(mitigated)`, then clamped to a minimum of `1`.
6. Steps 1–4 are computed in `float64` and rounded exactly once, at step 5. Rounding
   intermediates is a defect: it makes the worked example in §6 unreproducible.
7. `mitigation` is strictly below `1` for any non-negative `target_armor` because
   `softcap > 0`, so step 4 needs no clamp of its own.
8. There is no critical strike, no variance band, and no resistance in M2. The same
   caster hitting the same target always deals the same damage. This is what makes
   rule 6.1's "exact number of casts" assertion possible, and it is a curated
   simplification, not an observation about retail.

### 5.6 Damage application

1. `target.current_hp = max(0, target.current_hp - damage)`.
2. Add `damage` to `mob.threat[credited_character]`, where the credited character is
   the caster's character (rule 5.9.2). Threat accumulates even on the killing blow.
3. If the mob's `ai_state` is `Idle`, transition it to `Aggro` with
   `aggro_target = caster.entity_id`, regardless of distance. Being hit always pulls.
4. If `target.current_hp == 0`, go to rule 5.9.

### 5.7 Aggro

Evaluated for every `Idle` mob once per tick.

1. For each player in the mob's zone, `d = euclidean_distance(mob.position, player.position)`.
2. If `d <= AGGRO_RADIUS_M`, set `ai_state = Aggro` and `aggro_target = player.entity_id`,
   and stop scanning.
3. Ties — two players inside the radius on the same tick — resolve to the lower
   `entity_id`. Entities are already iterated in ascending id order when snapshots are
   built (`SERVER:internal/world/world.go`), so reusing that order costs nothing and
   makes the tie deterministic.
4. Aggro is not evaluated for mobs in `Aggro`, `Returning`, or `Dead`.

### 5.8 Chase and leash

Evaluated for every `Aggro` mob once per tick.

1. If `aggro_target` no longer exists in the zone — the player disconnected, and
   `zone.Leave` removed the entity — go to 5.8.5.
2. `leash_d = euclidean_distance(mob.position, mob.anchor)`. If `leash_d > LEASH_RADIUS_M`,
   go to 5.8.5.
3. Otherwise step the mob toward `aggro_target.position` by
   `walk_speed * CHASE_SPEED_MULTIPLIER * (TICK_INTERVAL_MS / 1000)` metres, stopping
   at `ABILITY_RANGE_M` from the target so the mob does not stack on top of the player.
4. Return.
5. **Leash break.** Set `ai_state = Returning`, clear `threat`, clear `aggro_target`.
6. A `Returning` mob steps toward `anchor` at the same speed as 5.8.3 and ignores both
   aggro (5.7) and incoming damage-induced aggro (5.6.3). It is untargetable while
   returning; ability uses against it are rejected by 5.2.3.
7. On arriving within `RANGE_TOLERANCE_M` of `anchor`, set `current_hp = max_hp` and
   `ai_state = Idle`.
8. The M2 target mob's content record has `extra.leashData.globalLeash: 'false'` and
   `localLeash: 'false'`. M2 does not interpret those fields; it applies
   `LEASH_RADIUS_M` uniformly. Recorded in §7.4.

### 5.9 Death, corpse, respawn

1. Set `alive = false`, `ai_state = Dead`, clear `aggro_target`, and stop replicating
   the mob as a combatant.
2. **Kill credit** goes to the character whose damage brought `current_hp` to `0` —
   the caster of the final `AbilityUse`, not the highest-threat character. M2 is
   single-attacker, so the two rules coincide; picking the killing blow is the one that
   needs no tie-break and is therefore the one specified.
3. Emit `MobKilled{victim_content_id, victim_level, killer_character_id, zone_id, server_tick}`.
   `quests.md` §5.4 consumes this. `MobKilled` is the **server-internal** event; what
   the client receives is `ServerMessage.death_event` on the reliable channel, which is
   the projection of it that ADR 0026 defines. The two are deliberately separate: the
   internal event carries `victim_content_id`, which is content identity the client has
   no business inferring kill credit from.
4. Create a `Corpse` with `despawn_tick = now_tick + ceil(CORPSE_TIMER_S * 1000 / TICK_INTERVAL_MS)`
   and `kill_credit` from 5.9.2. `loot.md` §5.1 takes over from here; the loot roll
   happens at corpse creation, not at loot time, so that the drop is fixed before the
   player opens it.
5. At `despawn_tick`, remove the corpse whether or not it was looted. Unlooted items are
   discarded, not mailed. Curated simplification.
6. Schedule the respawn at `despawn_tick + delay`, where `delay` is drawn uniformly
   from `[RESPAWN_DELAY_MIN_MS, RESPAWN_DELAY_MAX_MS]` and converted to ticks with
   `round_half_up`. The draw uses the zone spawn stream.
7. Respawn recreates the mob at `anchor` with `ai_state = Idle`, `current_hp = max_hp`,
   and a freshly drawn level per rule 5.1.5.
8. Some retail mobs carry an explicit despawn ability reference in their content record
   — the M2 zone's `demon-scout1-1` lists a `CorpseQuickDespawn` ability — which says
   corpse lifetime is per-mob data in retail, not one global constant. M2 uses the
   global constant anyway. Recorded in §7.5.

## 6. Worked example

### 6.1 One player, one cast, and the exact kill count

**Setup.** Every input below is a §3 constant, a content value cited in §3, or a
chosen scenario input marked as such. Nothing is a new constant.

- Caster: a level 1 player.
- Target: `mob.inst-league1.earth-elemental.earth-elemental2-1`, `level_min: 2`,
  `level_max: 2`, `walk_speed: 2.0`, no `hp_mod` on its `MobKind`.
- Scenario input: the player stands 6.0 m from the mob when the first cast goes out.
  Any distance inside `ABILITY_RANGE_M + RANGE_TOLERANCE_M` gives the same result.

**Derived stats** (rules 5.1.1–5.1.3):

| Quantity | Computation | Value |
|---|---|---|
| Mob level | uniform draw over `[2, 2]` | 2 |
| Mob `max_hp` | `(100 + 20 * (2 - 1)) * 1.0` | **120** |
| Mob `armor` | `10 + 20 * (2 - 1)` | **30** |
| Player `attack_power` | `12 + 2 * (1 - 1)` | **12** |

**Range check** (rule 5.3): `6.0 <= 10.0 + 0.5`. Passes.

**Damage** (rule 5.5), computed once and reused for every cast because 5.5.8 makes the
ability deterministic:

| Step | Computation | Value |
|---|---|---|
| 5.5.1 | `18 + 0.5 * 12` | `raw = 24` |
| 5.5.2 | `100 + 50 * 1` | `softcap = 150` |
| 5.5.3 | `30 / (30 + 150)` | `mitigation = 0.1666…` (exactly `1/6`) |
| 5.5.4 | `24 * (1 - 1/6)` | `mitigated = 20.0` |
| 5.5.5 | `round_half_up(20.0)`, clamped to `>= 1` | **`damage = 20`** |

**Casts to kill**: `ceil(120 / 20) = ` **6**.

**HP after each cast**: 100, 80, 60, 40, 20, 0.

**Timing** (rule 5.4.2): the GCD is 30 ticks at `TICK_INTERVAL_MS = 33.333333`, i.e.
exactly 1000 ms. With the first cast at `t = 0`, casts land at 0, 1000, 2000, 3000,
4000, and 5000 ms. The mob dies at **t = 5000 ms**, on server tick 150 relative to the
first cast.

**Aftermath**: corpse `despawn_tick` is `150 + ceil(30 * 1000 / 33.333333) = 150 + 900 = 1050`
(t = 35 000 ms). Respawn is scheduled between t = 45 000 ms and t = 49 000 ms.

This example is the acceptance test for the M2 combat loop: assert `damage == 20`,
assert the sixth cast and only the sixth cast emits `MobKilled`, and assert the GCD
rejects a seventh cast issued before tick 180.

### 6.2 Range rejection at the boundary

Same setup, player at 10.4 m: `10.4 <= 10.5`, passes. Player at 10.6 m: rejected with
`ErrOutOfRange`, and — per rule 5.4.2, which only runs after 5.3 — the GCD is **not**
consumed. A rejected ability costs the player nothing.

## 7. Open questions and placeholders

### 7.1 The level-to-base-HP/DPS curve is a curated decision

**Settled 2026-08-20.** This section used to be an open question. The search is now
finished and the answer is negative: the curve is not in the content tree, and
SarnautCore owns it.

What is observable: ordinary world mobs derive their health and damage from a
`MobKind` resource that carries only **multipliers** — `hpMod`, `dpsMod`, `expMod`,
`manaMod`, `lootMod` (`REF:Mechanics/MobKindTemplates/Usual1Player.xdb` and its
siblings; instance kinds such as
`REF:Mechanics/Creatures/AirElemental/AirElementalStartInstKind.(MobKind).xdb`
inherit them through a `Prototype` reference). The multiplicand — the per-level base
HP and base DPS those mods scale — does not appear anywhere under the data root.

Where the extraction pass looked, and what it found:

| Candidate | Result |
|---|---|
| `REF:Mechanics/GameRoot/ExperienceTable.xdb` | 50-row table, but it is XP-to-level for **players**, not mob base stats |
| `REF:Mechanics/GameRoot/StatGainTable.xdb` | 51-row table of player primary/secondary stat gain per level |
| `REF:Mechanics/GameRoot/MobRoot.xdb` | a warm-up buff reference and a melee range of 5; no curve |
| `REF:System/Corks/MobKind.xdb` | the null-object `MobKind`; carries `hpMod`/`dpsMod` like any other, no base |
| `REF:Mechanics/Rules/`, `REF:Mechanics/Variables/` | timers, contest windows and per-spell variables only |
| `REF:Types/types.xml` | a class registry; it names `MobKind` but declares no field values |
| Tree-wide grep for `baseHP`, `baseDps`, `hpPerLevel`, `healthPerLevel`, `levelTable` | hits only under `REF:Mechanics/Astral/Mobs/`, plus two unrelated buff files |

The only place absolute values appear is astral mobs, whose `AstralMobWorld`
resources carry `<baseHP>`, `<baseDPS>`, and `<baseAggroRange>` directly
(`REF:Mechanics/Astral/Mobs/Single/Assassin.xdb`), and those are a separate mob
system that bypasses the curve entirely.

**Decision.** Outcome 2 of the three previously listed: the curve is a **curated
SarnautCore constant**, not a cited one. `HP_BASE` and `HP_PER_LEVEL` in §3 are that
constant in its current two-parameter form. It graduates to a curated constant table
shipped in `DATA:` — with a schema in `data-schemas` and a loader change — when
combat needs more resolution than a straight line; until then the line stays, and
nothing in the pipeline pretends otherwise.

**How the gap is recorded in data.** `sarnaut-extract mobkinds` emits the resolved
multiplier chain with `_source.prototype_chain`, and every mob kind carries
`extra.level_curve_gap` naming this section. A reader of an extracted mob kind cannot
mistake the multipliers for absolute stats, and nothing downstream has to guess where
the missing multiplicand went.

Blast radius of a change: `HP_BASE` and `HP_PER_LEVEL` feed rule 5.1.1 only. Every
worked number in §6 changes; no rule text changes.

Until the multiplier chain is actually applied, it stays **explicitly deferred** per
ADR 0003. Rule 5.1.1 accepts an `hp_mod` argument so the seam exists, and M2 always
passes `1.0`.

### 7.2 Distance units are unverified

`AGGRO_RADIUS_M` is curated because the M2 target mob's content record carries no
aggro field at all. A sibling mob in the same zone does — `demon-scout1-1` has
`extra.aggroRadius: '35'` (`DATA:classic/zones/inst-league1/spawns/mobs/demon-scout.demon-scout1-1.yaml`)
— but 35 in the same units as a `walk_speed` of 1.5 would be an aggro radius roughly
23 seconds of walking wide, which suggests the field is not in metres, or not in the
same metres as position coordinates. The `<baseAggroRange>` of 500 in
`REF:Mechanics/Astral/Mobs/Single/Assassin.xdb` points the same way. Settle this by measuring a known distance in a converted zone against the
same zone's placement coordinates, then convert every distance constant in §3 at once.

### 7.3 No line of sight

Rule 5.3.3 skips LoS because terrain height and collision are unresolved
(`SERVER:internal/world/world.go` carries an open TODO on Z movement). Once collision
lands, LoS becomes rule 5.3.3 proper and §6.2 gains a case.

### 7.4 Leash flags are read but not interpreted

`extra.leashData.globalLeash` and `localLeash` survive extraction into the mob YAML.
Both are `false` on the M2 target. Their semantics — and whether "no leash" means
infinite chase or no chase — are unestablished, so rule 5.8.8 ignores them.

### 7.5 Corpse lifetime is per-mob in retail

See rule 5.9.8. Making `CORPSE_TIMER_S` per-mob requires deciding how an ability
reference on a mob record turns into a duration, which is the spell-effect system, and
that is out of scope until a spells spec exists.

## 8. Sources

Reference-root paths (`REF:` = `servers-clean/1.1.02.0/game/data/`):

- `Mechanics/GameRoot/SpellRoot.xdb` — `<globalCooldown>`.
- `Mechanics/MobKindTemplates/Usual1Player.xdb`, `Mechanics/MobKindTemplates/AE1Player.xdb`
  — the `hpMod` / `dpsMod` / `expMod` / `manaMod` / `lootMod` field set on `MobKind`
  prototypes, and the absence of any absolute HP or DPS field.
- `Mechanics/Creatures/AirElemental/AirElementalStartInstKind.(MobKind).xdb` — a
  non-prototype `MobKind` that reaches its mods through a `Prototype` reference.
- `Mechanics/Astral/Mobs/Single/Assassin.xdb` — `AstralMobWorld` carrying explicit
  `<baseHP>`, `<baseDPS>`, `<baseAggroRange>`, unlike world mobs.
- `Maps/Inst_LeagueStart/SpawnTables/InstLeague1/AirElemental.(MobSpawnTable).xdb` —
  `spawnTime` of type `TimeRange` with `<min>` / `<max>`. The same directory's other
  spawn tables use `TimeOnce`, `TimeCommon`, and `TimeNever`, which is why respawn is
  spawn-table data and not a mob property.
- `Mechanics/Spells/Creatures/` — the distribution of `<radius>` values used for the
  §3 note on `ABILITY_RANGE_M`.

SarnautCore data-repository paths (`DATA:` = `data/`):

- `classic/zones/inst-league1/spawns/mobs/earth-elemental.earth-elemental2-1.yaml` —
  the M2 target mob: `level_min`, `level_max`, `walk_speed`, `faction`, `extra.leashData`.
- `classic/zones/inst-league1/spawns/mobs/demon-scout.demon-scout1-1.yaml` —
  `extra.aggroRadius`, `extra.lootDropModifier`, and the `CorpseQuickDespawn` ability
  reference.

SarnautCore server-repository paths (`SERVER:` = `server/`):

- `config.example.yaml` — `world.tick_interval`, `world.snapshot_interval`.
- `internal/world/world.go` — ascending-`entity_id` iteration order, the open TODO on
  Z-axis movement, and `Zone.Leave` semantics referenced by rule 5.8.1.

Observable behaviour: none of the numbers above were taken from running the retail
client or server. Everything cited is a field value or a structural property of a file
at the path given.

Per ADR 0011, nothing in this document is quoted, paraphrased, or transliterated from
decompiled code, and no localization strings, art, or bulk data tables appear here.

## 9. Change log

| Date | Change |
|---|---|
| 2026-08-20 | Created for M2. |
| 2026-08-20 | §7.1 settled: the level curve is absent from the source tree and is a curated SarnautCore constant. Search record added. |

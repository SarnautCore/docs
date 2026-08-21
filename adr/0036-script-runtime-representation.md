# ADR 0036 — One recursive representation for the script runtime

**Status**: Accepted (2026-08-20)

## Context

M3 must carry extracted impact, predicate, effect, and addressee-finder trees
through authored YAML, `sarnaut.content.v1` pack rows, and `internal/script` without
flattening or translating them at a repository boundary. Quest rewards, triggers,
script zones, and spells all contain the same recursive shapes. Choosing a separate
shape for any caller would create a second evaluator and make nested behavior depend
on which subsystem invoked it.

The tutorial corpus is too broad for a build-time opcode allow-list. The following
census was reproduced from the classic 1.1.02.0 source on 2026-08-20. The full scope
is the union of these selectors. Counts are XDB documents, with each document counted
once.

| Selector below `game/data` | Documents |
|---|---:|
| `World/Quests/InstLeague1/**/*.xdb` | 108 |
| `Maps/Inst_LeagueStart/**/*.xdb` | 231 |
| `Characters/**/InstLeague1/**/*.xdb` | 32 |
| `Creatures/**/InstLeague1/**/*.xdb` | 23 |
| `Items/**/InstLeague1/**/*.xdb` | 176 |
| **Union** | **570** |

| Node family | Distinct types in the 570-document scope | Uses in that scope | Distinct types in `World/Quests/InstLeague1` | Uses in that quest tree |
|---|---:|---:|---:|---:|
| `gameMechanics.elements.impacts.*` | 65 | 905 | 60 | 634 |
| `gameMechanics.elements.effects.*` | 18 | 81 | 18 | 78 |
| `gameMechanics.elements.predicates.*` | 11 | 187 | 9 | 159 |
| `gameMechanics.elements.addresseeFinders.*` | 3 | 15 | 3 | 14 |

`ImpactsDeferred` is the most common impact at 144 uses. Other shapes that prevent a
flat representation include `ImpactIfTarget` at 62 uses, `ImpactIfCaster` at 9,
`PredicateCharacterClass` at 58, and `PredicateCharacterRace` at 42.

This PowerShell command reproduces the selection and both census columns. It reads
the source tree and writes nothing.

```powershell
$root = 'E:\allods\servers-clean\1.1.02.0\game\data'
$groups = [ordered]@{
  'World/Quests/InstLeague1' = @(
    Get-ChildItem -LiteralPath "$root\World\Quests\InstLeague1" -Filter *.xdb -File -Recurse
  )
  'Maps/Inst_LeagueStart' = @(
    Get-ChildItem -LiteralPath "$root\Maps\Inst_LeagueStart" -Filter *.xdb -File -Recurse
  )
  'Characters/**/InstLeague1' = @(
    Get-ChildItem -LiteralPath "$root\Characters" -Filter *.xdb -File -Recurse |
      Where-Object FullName -Match '\\InstLeague1(\\|$)'
  )
  'Creatures/**/InstLeague1' = @(
    Get-ChildItem -LiteralPath "$root\Creatures" -Filter *.xdb -File -Recurse |
      Where-Object FullName -Match '\\InstLeague1(\\|$)'
  )
  'Items/**/InstLeague1' = @(
    Get-ChildItem -LiteralPath "$root\Items" -Filter *.xdb -File -Recurse |
      Where-Object FullName -Match '\\InstLeague1(\\|$)'
  )
}
$files = @($groups.Values | ForEach-Object { $_ }) | Sort-Object FullName -Unique
$pattern = 'type\s*=\s*"gameMechanics\.elements\.(?<family>impacts|effects|predicates|addresseeFinders)\.(?<name>[A-Za-z0-9_]+)"'
$rows = foreach ($file in $files) {
  foreach ($match in [regex]::Matches((Get-Content -Raw -LiteralPath $file.FullName), $pattern)) {
    [pscustomobject]@{
      Family = $match.Groups['family'].Value
      Name = $match.Groups['name'].Value
      Path = $file.FullName
    }
  }
}
$groups.GetEnumerator() | ForEach-Object {
  [pscustomobject]@{ Selector = $_.Key; Documents = $_.Value.Count }
} | Format-Table
'TOTAL documents={0}' -f $files.Count
$questRoot = "$root\World\Quests\InstLeague1\"
foreach ($family in 'impacts','effects','predicates','addresseeFinders') {
  $all = @($rows | Where-Object Family -eq $family)
  $quest = @($all | Where-Object Path -Like "$questRoot*")
  [pscustomobject]@{
    Family = $family
    ScopeTypes = @($all.Name | Sort-Object -Unique).Count
    ScopeUses = $all.Count
    QuestTypes = @($quest.Name | Sort-Object -Unique).Count
    QuestUses = $quest.Count
  }
}
```

## Decision

### One ordered recursive node

The one representation is `sarnaut.content.v1.ScriptNode`. Authored YAML has the
same fields, the extractor constructs generated `ScriptNode` messages after its
transient XML read, pack rows encode those messages, and `internal/pack` exposes a
Go type alias to the decoded message. `internal/script` evaluates that alias
directly. There is no extractor-only impact model, runtime AST, or conversion into
Go structs per opcode.

The protobuf shape is normative. Names below describe fields; field numbers are set
when the content schema lands.

```proto
message ScriptNode {
  string node_key = 1;
  string family = 2;
  string opcode = 3;
  CoverageTier tier = 4;
  repeated ScriptField fields = 5;
}

message ScriptField {
  string name = 1;
  ScriptValue value = 2;
}

message ScriptValue {
  oneof value {
    sint64 integer = 1;
    Decimal decimal = 2;
    bool boolean = 3;
    string text = 4;
    ContentRef reference = 5;
    uint64 duration_ms = 6;
    ScriptNode node = 7;
    ScriptValueList list = 8;
  }
}

message ScriptValueList {
  repeated ScriptValue values = 1;
}
```

`Decimal` is an exact signed mantissa plus a base-10 scale. `ContentRef` contains a
canonical content id and row type, never a source href. `family` and `opcode` are
strings so an unknown value survives extraction and pack round trips. The `tier`
field controls whether it executes.

Fields are sorted bytewise by name, field names are unique within a node, and lists
retain source order. `node_key` is the owning content-row id followed by the field
names and list ordinals on the path to the node. This makes it stable across builds
that contain the same content. The pack validator rejects duplicate fields, an
invalid value union, duplicate node keys, excessive nesting, and an excessive node
count. It does not reject an unfamiliar opcode.

The following schema-shaped examples are illustrative and contain no source values.
They show the required nesting without introducing another representation.

`ImpactsDeferred` wraps an arbitrary ordered impact list:

```yaml
family: impact
opcode: ImpactsDeferred
fields:
  - {name: delay, value: {duration_ms: $delay}}
  - name: impacts
    value:
      list:
        - {node: {family: impact, opcode: ImpactClientData, fields: [...]}}
        - {node: {family: impact, opcode: ImpactIncreaseQuestCount, fields: [...]}}
```

`ImpactIfTarget` and `ImpactIfCaster`, found 62 and 9 times respectively, use the
same predicate-and-branch shape. Changing the opcode selects which invocation role
the predicate reads.

```yaml
family: impact
opcode: ImpactIfTarget # or ImpactIfCaster
fields:
  - {name: predicate, value: {node: {family: predicate, opcode: PredicateHasItem, fields: [...]}}}
  - name: impactsIf
    value:
      list:
        - {node: {family: impact, opcode: ImpactDestroyItem, fields: [...]}}
```

`ScaledPhysicalDamage` retains its on-hit children, including `MarkedImpact`, in
order:

```yaml
family: impact
opcode: ScaledPhysicalDamage
fields:
  - name: impactsOnHitTarget
    value:
      list:
        - {node: {family: impact, opcode: MarkedImpact, fields: [...]}}
  - {name: minDamage, value: {decimal: $minimum}}
  - {name: maxDamage, value: {decimal: $maximum}}
```

A `Switch` effect retains both lifecycle branches. `impactsOn` runs once when the
effect attaches. `impactsOff` runs once when it detaches or expires.

```yaml
family: effect
opcode: Switch
fields:
  - name: impactsOn
    value:
      list:
        - {node: {family: impact, opcode: ImpactsDeferred, fields: [...]}}
  - name: impactsOff
    value:
      list:
        - {node: {family: impact, opcode: ImpactIncreaseQuestCount, fields: [...]}}
```

The quest-2-20 reward cascade remains an ordered list of roughly 20
`ImpactIfTarget` nodes. Each branch retains the `PredicateCharacterRace` and
`PredicateCharacterClass` children of its `PredicateAnd`, followed by its reward.
The evaluator visits every branch in list order. It does not turn the cascade into
a map or stop after the first true branch.

```yaml
value:
  list:
    - node:
        family: impact
        opcode: ImpactIfTarget
        fields:
          - name: predicate
            value:
              node:
                family: predicate
                opcode: PredicateAnd
                fields:
                  - name: predicates
                    value:
                      list:
                        - {node: {family: predicate, opcode: PredicateCharacterRace, fields: [...]}}
                        - {node: {family: predicate, opcode: PredicateCharacterClass, fields: [...]}}
          - name: impactsIf
            value:
              list:
                - {node: {family: impact, opcode: ImpactGiveItem, fields: [...]}}
    - {node: {family: impact, opcode: ImpactIfTarget, fields: [...]}}
```

### Coverage has three tiers

Every node carries exactly one tier:

1. `implemented` means `internal/script` validates the node's fields and executes
   it.
2. `inert-and-counted` means the node parses, remains in the pack, records a census
   hit, and has no effect. This tier is allowed only when omission cannot alter
   authoritative M3 state or bypass a safety check.
3. `refused` means evaluation returns an error naming the content row, node key, and
   opcode before any child executes. Unknown opcodes and unsupported nodes that can
   affect authoritative state default to this tier.

The pack compiler has no opcode allow-list. It validates structure and preserves
all three tiers. `sarnaut-pack check` reports counts for all nodes, accepts
inert-and-counted nodes, and fails a pack containing a refused node. At runtime, a
refused node also fails on reach. A refused node reached by
`scripts/m3-tutorial-driver` is a hard CI failure, even if a caller catches the
runtime error.

M3's implemented set is fixed to the following node types:

| Family | Implemented in M3 |
|---|---|
| Impact and control | `BuffAttacher`, `BuffDetacher`, `DeviceDie`, `DeviceImpactsDeferred`, `GoThroughPath`, `ImpactAddExperience`, `ImpactClientData`, `ImpactClientDataCoords`, `ImpactDestroyItem`, `ImpactDeviceDisintergrate`, `ImpactDeviceSetVisualState`, `ImpactFindPermanentDevice`, `ImpactFindSingleDevice`, `ImpactFindSingleMob`, `ImpactGiveItem`, `ImpactIfCaster`, `ImpactIfTarget`, `ImpactIncreaseQuestCount`, `ImpactMobChat`, `ImpactRenewQuestMarks`, `ImpactScriptZoneSetDisabled`, `ImpactScriptZoneVariableSummand`, `ImpactStopTalk`, `ImpactsDeferred`, `ImpactsToInterlocutor`, `MarkedImpact`, `ResetSpawnTable`, `ReturningImpact`, `ScaledPhysicalDamage`, `SpawnSingleDevice`, `SpawnTableObjects` |
| Predicate | `PredicateAnd`, `PredicateCharacterClass`, `PredicateCharacterRace`, `PredicateHasItem`, `PredicateIsDead`, `PredicateMobWorld`, `PredicateQuestStatus`, `PredicateVariableValueGreatThan` |
| Effect | `CombatStateTrigger`, `CreatureNoticedTrigger`, `EquipTrigger`, `HealthTrigger`, `LifeGuard`, `Switch` |
| Addressee finder | `AddresseeFinderSelf`, `AddresseeFinderTarget`, `AddresseeFinderSingleMob` |
| Scaler | `TrivialScaler` |

Periodic `ImpactsOverTime` and `EntityImpactsOverTime` remain
inert-and-counted in M3. Non-trivial scalers and every state-changing type outside
the table are refused. Other presentation-only types in the 570-document corpus are
inert-and-counted. Moving any type between tiers changes the checked-in
`server/internal/script/testdata/inst-league1-tier-counts.json` table. That table has
rows sorted by family and opcode with `tier`, `nodes`, and `uses`; CI diffs it
against the extractor and pack report. Coverage can widen only through that visible
review diff.

### One evaluator and one host contract

`internal/script.Evaluate` accepts a `pack.ScriptNode`, an invocation frame, and a
host. The frame names the stable evaluation id, pack id, event, zone, source,
caster, target, and interlocutor. The four callers are:

1. quest start and reward impacts;
2. trigger activation;
3. script-zone entry, leave, and variable paths;
4. spell caster conditions, caster impacts, and target impacts.

All four use the same entry point and the same host interface:

```go
type Host interface {
	Now() time.Time
	Query(context.Context, Query) (Value, error)
	Resolve(context.Context, ResolveRequest) ([]gametypes.EntityID, error)
	Apply(context.Context, Command) error
	Enqueue(context.Context, Deferred) error
}
```

`Query`, `ResolveRequest`, and `Command` are typed unions owned by
`internal/script`; the host never receives an opcode and never decides a coverage
tier. `Resolve` returns entity ids in bytewise id order. Every `Command` carries an
execution key so a replay is idempotent. Caller adapters may import the domain
modules they bridge. `internal/script` imports only `internal/gametypes` and
`internal/pack`, plus the Go standard library.

### Deterministic probability and ordering

Probability never reads a process-global random generator or wall-clock entropy.
The extractor stores probability as an exact decimal. For each activation the
evaluator hashes the pack id, evaluation id, node key, and activation ordinal with
SHA-256. It interprets the first 64 bits as an unsigned integer and compares that
integer with the exact probability threshold using integer arithmetic. Zero always
misses and one always hits. Re-evaluating the same activation after a restart makes
the same choice. `RandomImpact` uses the same hash input and selects from the stored
branch order. Activation ordinals follow depth-first node order and sorted resolved
entity ids, never map iteration order.

`ImpactsDeferred` schedules its children in stored list order. A child's due time is
the host's integer millisecond time at the moment its enclosing deferred node runs,
plus that node's delay. A nested delay is therefore relative to execution of its
parent, not the parent's original enqueue time. The queue orders entries by
`(due_at_ms, enqueue_seq)`, where `enqueue_seq` is a persisted, monotonic sequence
allocated by the queue scope. Zero-delay work still enters the queue. On restart,
overdue entries run immediately in that same order; their delays never restart.

This matters in the tutorial because a 60000 ms buff contains deferred delays of
43500, 5000, 3000, and 58000 ms. A restart between those beats must not drop or
reorder the remaining work.

The persisted deferred queue has one row per scheduled child with this shape:

| Field | Purpose |
|---|---|
| `queue_id` | Stable primary key and base of the idempotency key. |
| `scope_kind`, `scope_id` | Character or zone queue that owns the monotonic sequence. |
| `due_at_ms`, `enqueue_seq` | Total dequeue order. |
| `pack_id`, `node_key`, `node_bytes` | Exact deterministic `ScriptNode` plus the pack identity used to resolve references. |
| `evaluation_id`, `activation_ordinal` | Probability replay inputs. |
| `zone_id`, `source_id`, `caster_id`, `target_id`, `interlocutor_id` | Invocation roles restored after restart. |
| `attempts`, `last_error` | Retry accounting and diagnostics. |

`node_bytes` is the deterministic protobuf encoding, not a second queue-specific
impact shape. Dequeue is at least once: the host deletes the row only after all
commands succeed. A crash after a command but before deletion replays the row, and
the command's `(queue_id, node_key)` execution key suppresses duplicate durable
effects.

### ADR 0020 and module containment

[ADR 0020](0020-server-scripting-gopher-lua.md) remains accepted, but this decision
narrows its boundary. Extracted impacts, predicates, effects, and finders never
become Lua and Lua cannot register or override their opcodes. Lua is reserved for
hand-authored exceptional encounter choreography. Its sandbox calls a small
simulation API; it does not receive `ScriptNode`, decode content rows, or become a
fifth caller of this evaluator. The data-driven core named by ADR 0020 is the
runtime decided here.

`LifeGuard` is the pre-commitment that tests the module boundary. If honoring it
would require `internal/combat` to import `internal/script`, the seam is wrong and
the change stops. The damage clamp moves behind the existing combat hook, with a
host adapter above both packages invoking that hook. A depguard exemption is never
an acceptable remedy.

### Alternatives rejected

**A closed protobuf `oneof` and a Go type per opcode.** This gives each known type a
pleasant API, but makes extraction of the next unknown type a schema edit and code
generation event. It also cannot round-trip the unimplemented majority without
pretending they are supported. At the current census it would bind pack validity to
an enumerated list of more than 65 impacts plus the other families.

**An untyped JSON, YAML, or `map<string, Value>` property tree.** This preserves
unknown names but loses exact decimals, content-reference identity, ordered
children, and structural validation. Deferred replay would then depend on a second
runtime decoder. A repeated typed field/value union preserves the same openness
without giving up determinism.

## Consequences

- The extractor, content schema, pack reader, interpreter, and deferred queue share
  one recursive data contract. New opcodes do not require a representation change.
- Supporting a type is a tier change plus a handler and tests. It is not a pack
  format change, and the per-tier count diff exposes the coverage increase.
- The generic node shape moves semantic validation into opcode handlers. Golden
  traces over the discovered trigger corpus and the M3 driver are therefore binding
  checks, not optional examples.
- Persisting full deterministic node bytes costs more space than persisting a
  pointer. M3 accepts that cost to make restart behavior independent of source files
  and to avoid silently executing a different node after a pack change.
- Domain modules remain ignorant of the interpreter. Host adapters translate typed
  queries and commands at the existing module seams.

## Amendments

### 2026-08-21 — Census scope, node families, and per-opcode scaler tiering

The representation, the three tiers, the four callers, the host interface, the
determinism rules, the persisted deferred queue, and the ADR 0020 boundary are
unchanged. What changes is scope and tiering. The M3 impact-interpreter survey
followed the reference graph out of the 21 tutorial quests and found three places
where the tables above stop short, each of which blocks an M3 exit criterion on its
own.

**The census scope gains two selectors.** The five selectors describe the four
tutorial trees correctly, but the set is not closed under the reference graph of the
quests. Three count-special objectives, `Quest_1_30/CountId_1`, `Quest_2_10/CountId_1`
and `Quest_2_10/CountId_2`, have no `ImpactIncreaseQuestCount` anywhere in the 570
documents. Their incrementers are `RatKiller`, `HumanKiller` and `Human2Killer` under
`Mechanics/Spells/QuestSpells/IL_QuestSpells/`, reached from the quests' own
`startImpacts` through `ImpactFindSpawnTable` and `ImpactAttachTrigger`. Quest 1-20
additionally attaches `IL_QuestSpells/PaladinAsks.(BuffResource).xdb` and
`Mechanics/Spells/ItemSpells/StartInstPotions/Buff01.(BuffResource).xdb` from its start
and reward impacts.

| Added selector below `game/data` | Documents |
|---|---:|
| `Mechanics/Spells/QuestSpells/IL_QuestSpells/**/*.xdb` | 186 |
| `Mechanics/Spells/ItemSpells/StartInstPotions/**/*.xdb` | as discovered |

The union is therefore no longer fixed at 570 documents, and the family counts of 65
impacts, 18 effects, 11 predicates, and 3 addressee finders hold only for the
five-selector scope they were measured over. Each count is read with its scope
attached. The bounded transitive closure from the 21 quest documents, following hrefs
into script-bearing root elements, reaches 228 documents. The trigger corpus is 13
documents, not 10. What binds is the discovered set with zero dangling references; the
extractor reports what it finds. Left uncorrected, the M3 zero-skip gate cannot pass at
any coverage tier, because three objectives have no incrementer to reach.

**The representation has seven node families, not five.** `family` is a normative
`ScriptNode` field, so the legal values are part of this decision. The tier table rows
impact, predicate, effect, addressee finder, and scaler. Two more carry M3 weight:

- `calcer`, from `gameMechanics.elements.calcers.*`;
- `basic`, from `gameMechanics.constructor.basicElements.*`.

`PredicateAnd` is already implemented and already lives in `basicElements`, so the
table already mixes namespaces without saying so. Naming the families makes that
explicit. The consequence is concrete rather than cosmetic: `HealthTrigger` is
implemented, its `healthOn` operand is `FullHealthCalcer` and its `healthOff` operand
is `FloatZero`, and neither operand has a tier. Both default to refused, and a refused
node reached by `scripts/m3-tutorial-driver` is a hard CI failure. As written above,
the first rat death in quest 1-30 fails the build.

**Scalers are tiered per opcode.** The sentence "Non-trivial scalers and every
state-changing type outside the table are refused" is replaced by this rule: a scaler
carries a tier per opcode like every other family, and an untiered scaler is refused.
`Mechanics/Spells/Warrior` and `Mechanics/Spells/AutoAttack` produce every damage
number through `PhysicalScaler`, `PhysicalRangedScaler`, and `WeaponSpeedScaler`, and
`ScaledPhysicalWeaponDamage` is the auto-attack itself. Under the blanket rule the
first swing reaches a refused node.

**There is a fourth addressee finder.** Three finders is correct for the
570-document scope and stops being correct once spells land. `AddresseeFinderCaster`
appears in `Mechanics/Spells/Warrior` and joins the implemented tier.

**The M3 implemented tier gains the following types**, all reached by the tutorial
closure or the Warrior kit:

| Family | Added to the implemented tier |
|---|---|
| Trigger binding | `ImpactAttachTrigger`, `TriggerAgentSelf`, `TriggerAgentInterlocutor`, `TriggerAgentSimple`, `TriggerAgentOnTagged` |
| Calcer | `FullHealthCalcer`, `FloatData` |
| Basic | `FloatZero`, `PredicateOr`, `PredicateNot`, `EffectTrigger` |
| Addressee resolution | `ImpactFindSpawnTable`, `ImpactCreaturesAround`, `ImpactDevicesAround`, `AddresseeFinderCaster` |
| Quest counting | `TagMobForKill` |
| Spawn and despawn | `ImpactInstantiating`, `ImpactInstantiatingSimple`, `ReturningInstantiatingImpact`, `SpawnSingleMob`, `ImpactKill`, `ImpactDisintegrate`, `ImpactStopSpawn` |
| World state | `DoorSwitch`, `ImpactTeleportLoc`, `ImpactTurnMob`, `ImpactGoToLocator`, `ImpactMobMorph`, `ImpactSummon`, `ImpactClearTarget`, `ImpactLearnUp`, `AttachAbility` |
| Variables | `ImpactIfScriptZoneVariable`, `PredicateVariableValueLessThan` |
| Aggro | `ImpactActivateAggro`, `ForceAggro` |
| Probability | `ProbabilisticImpact`, `ProbabilisticImpactBinary`, `RandomImpact` |
| Scaler | `PhysicalScaler`, `PhysicalRangedScaler`, `WeaponSpeedScaler`, `LinearEffectScaler`, `LinerMultiplierScaler` |
| Warrior kit | `ScaledPhysicalWeaponDamage`, `ImpactSetTarget`, `ImpactRemoveBuff`, `ImpactRemoveAllBuffsFromGroup`, `ImpactResetCombatAdvantage`, `PredicateEquipped`, `PredicateRemote`, `PredicateHasCombatAdvantage`, `PredicateIsMob`, `EffectLinearStatModifier` |

The determinism rule for `ProbabilisticImpact` and `RandomImpact` is already decided
above; those types were never tiered.

**Movement-lockout effects stay inert-and-counted, as a stated concession.**
`EffectDisableMove`, `EffectDisableRotate`, `EffectDisableEvadeTimeout`,
`AutoAttackDisabler`, and `EffectNoAggro` appear on quest 3-10's `AnimationBuff`, which
pins the player through a scripted beat. Omitting them cannot corrupt authoritative
state, which is the inert tier's admission test, but it does let a player walk out of a
cutscene. M3 accepts that, and the quest mechanics spec records it.

**Quest count-special consumes the interpreter through `Host.Apply`.**
`internal/quests` does not learn what an impact is. `QuestCountSpecial` becomes a
counter keyed by a `QuestCountId` content id, and the quest module exposes exactly one
new operation: increment counter K for character C by N, clamped at the objective's
limit, idempotent under the command's execution key. The evaluator reaches it as a
`Command` through the existing host interface. The adapter lives above both packages in
`internal/session`, where `ZoneBinding.KillSink` already adapts a combat kill into a
quest kill, so the ADR 0033 module row for `internal/quests` is untouched: quests still
import only inventory, pack, and gametypes, and never `internal/script`. The quest row
needs no shape change, because `pack.QuestObjective.CounterKey` already carries the
count id; the only addition is a map from count id to objective index, built once per
definition. Two properties bind the tests, both taken from the data. `Quest_1_20/CountId_1`
has two independent incrementers, `DressTrigger` and the broken-door exploit, against a
limit of 1, and both firing must leave the counter at 1. `RatKiller` increments through
`ReturningImpact`, so credit must land on the killer rather than the dying rat; a test
that attributes it to the addressee passes trivially and proves nothing.

**Spell documents are a root-element family the extractor discovers**, not a
directory: `SpellSingleTarget`, `SpellCasterSelf`, `AbilityResource`, and
`BuffResource`. `SpellCasterSelf` already appears seven times inside the tutorial
scope, and `Items/QuestItems/InstLeague1/HealElixir/Heal.(SpellCasterSelf).xdb` is
itself a count-special driver, so the spell path is on the zero-skip critical path and
not only on the combat one. The Warrior surface is 28 distinct node types over 15
documents. Damage flows from `ScaledPhysicalWeaponDamage` through a scaler to
`MarkedImpact`, and the scaler is where combat's numbers come from. The `LifeGuard`
pre-commitment holds unchanged: the arithmetic stays behind the existing combat hook
with a host adapter above both packages, and if a scaler handler finds itself wanting
`internal/combat`, the seam is wrong and the change stops.

Nothing here changes the pipeline posture. The pack compiler still has no opcode
allow-list; every change above decides which opcodes execute, never which opcodes
encode, and an untiered opcode still round-trips. Every addition lands as a row in
`server/internal/script/testdata/inst-league1-tier-counts.json`, so widening coverage
stays a visible review diff. `pack_id` is unaffected, because it is a function of table
bytes and tiering rides in rows that already exist.

This amendment grants no import edge. `internal/script` still has no legal caller,
`server/.golangci.yml` has no depguard row for it, and adding one remains an
architecture change under ADR 0033 that belongs to the M3 depguard amendment. A
skeleton may exist behind a feature flag with no caller.

### 2026-08-21 — Field-level corrections from the count-special implementation

Implementing the count-special shapes and the auto-attack path against the 1.1
documents corrected several details the survey and the amendment above carried
imprecisely, and surfaced types the tier table should name. The representation, the
tiers, the callers, the host contract and the determinism rules are unchanged; what
follows is field-level fact, each item verified against `Types/types.xml` and the
authored documents.

**`ImpactIncreaseQuestCount`'s delta field is named `value`.** The reflection schema
gives the type exactly two fields: `id`, a required `QuestCountId` reference, and
`value`, an Integer defaulting to 1. The overwhelming majority of the 1486 uses in
the reference tree omit it. The mistake this corrects was silent: an implementation
reading a wrongly guessed field name defaults every increment to 1, which is correct
for every tutorial document and wrong the first time content asks for more.

**`HealthTrigger` has five fields, and `impactsOff` is detach-time cleanup.** The
schema lists `healthOn`, `healthOff`, `impactsOn`, `impactsOff`, and `effects` — a
nested effect list that attaches while the trigger is on. `RatKiller` does not use
`effects`, but `IL_QuestSpells/SummonZombie.(AbilityResource).xdb` does, inside the
tutorial's reach. Of the second impact list the schema says it is called on detach,
if the first was called first, and that its meaning is cleanup. So `impactsOff` is
not an upward crossing of `healthOff`: healing back past the threshold runs nothing,
ending the attachment runs the cleanup. `healthOff` reads as a re-arm threshold, and
a pure crossing model re-arms on its own, so nothing in the tutorial depends on the
difference — `RatKiller`'s `healthOff` is `FloatZero`.

**`WeaponSpeedScaler` carries one optional field, `source`**, of the enum
`gameMechanics.world.attack.AttackSource` with values `Mainhand` and `Ranged`,
selecting which weapon's speed the scaler reads. It is the only field on any of the
three per-opcode scalers; `PhysicalScaler` and `PhysicalRangedScaler` carry none in
any of their 173 and 40 uses.

**Two vocabularies name a weapon slot, reconciled at the host adapter.** Equip
gates (`EquipTrigger.slot`) speak `DressSlot` — `MAINHAND`, `TWOHANDED` — while
damage sources and `WeaponSpeedScaler.source` speak `AttackSource` — `Mainhand`,
`Ranged`. The interpreter passes both through verbatim, as the content spells them,
and the host adapter that owns the equipment model is where the two vocabularies map
onto one set of slots. Folding them into one spelling inside the evaluator would
falsify the round trip back to content.

**`HealthCalcer` exists and is refused.** The calcer family has a fourth member
beside `FullHealthCalcer`, `FloatZero` and `FloatData`. No tutorial document reaches
it, so it takes the family default: refused, loudly, until data demands it.

**`TriggerAgentAddressee` and `TriggerAgentTarget` exist and are not in the
tutorial.** The trigger-agent family is six members, not four. The schema defines
all six over the `TriggerAgentResource` base (`trigger`, `detachesOnDeath`);
`TriggerAgentSimple` adds `mobWorld`, and `TriggerAgentOnTagged` adds `mobWorld` and
an `onSelf` Boolean that no tutorial document sets — `onSelf` true is refused rather
than guessed. The implemented tier holds the four the tutorial reaches; Addressee
and Target wait for data that uses them.

**`ScaledPhysicalWeaponDamage` inherits an unused `scalerTarget` slot.** Its own
fields are `scaler`, `avgDamage` and `source`; the `PhysicalDamage` base contributes
`scalerTarget`, `canBeAvoided`, `cantBeMissed`, `threatMultiplier`, the on-hit
impact lists and `statsConvertor`. The Warrior and auto-attack documents leave
`scalerTarget` empty, so the implementation ignores the slot and the census will say
so the first time content fills it.

**`ImpactSetTarget`'s direction is inferred, and stays an open question.** The node
appears in `Mechanics/Spells/Warrior` carrying an `AddresseeFinderCaster`, and the
implementation reads it as "point the addressee at the found entity". No second
independent use in reach confirms which side is the pointer and which the pointee;
the reading is recorded here as inference, not fact, and the first document that
distinguishes the two decides it.

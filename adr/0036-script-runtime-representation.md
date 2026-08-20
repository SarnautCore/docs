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

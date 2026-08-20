# ADR 0035 — M3: the playable League tutorial, its exit gate, and the binding cut ladder

**Status**: Accepted (2026-08-20)

## Context

ADR 0003 defines milestones M0 through M2 only. M2 is finished: one zone, one class,
one ability, one quest, one mob that does not fight back, and `scripts/m2-slice-driver`
replaying the loop headless in CI. Two literal ADR 0003 queue pointers, plus
several equivalent deferrals, no longer identify a milestone that will resolve them.
Four of the eight modules ADR 0033 named still do not exist, and its depguard
allow-list carries 3 rules where §2 specifies 9.

The owner has decided what M3 is: **a fully playable League tutorial journey** — the
complete `InstLeague1` questline, real Warrior spells from extracted data, combat that
fights back, characters standing on the ground, death and respawn, and the connective
UI. The Lightwood levelling spine is extracted and converted already and is explicitly
deferred to M4.

### What the tutorial actually contains

Every number below was re-derived from an owned 1.1.02.0 `game/data` tree over the
four tutorial areas (`World/Quests/InstLeague1`, `Maps/Inst_LeagueStart`, and
the `InstLeague1` subtrees of `Characters`, `Creatures` and `Items`), 570 documents in
total. They replace the estimates that circulated during option analysis, several of
which were wrong by 3x or more, and they are reproducible by root-element census
rather than by filename.

| Surface | Count | Note |
|---|---|---|
| `QuestResource` | 21 | the questline |
| Extracted objectives | 12 | 10 `quest-count-special`, 1 `quest-count-item`, 1 `quest-count-kill` |
| Quests servable today | 12 of 21 | 9 are skipped by `content.skip_unsupported_quests` |
| `TriggerResource` | 10 | by root element across the full scope; the quest tree has 9 roots but only 7 suffixed filenames |
| `QuestCountId` | 11 | |
| `ScriptZone` | 32 | placed 20 times as `ScriptZoneElement` |
| `VariableResource` | 3 | `DemonsCount_Current`, `DemonsCount_Total`, `ElementalCount_Current` |
| `BuffResource` | 25 | |
| `ChestResource` / `DoorResource` | 6 / 4 | |
| `TextMessage` / `ClientData` | 19 / 19 | |
| Warrior spell/ability document sets | 7 | the child directories under `Mechanics/Spells/Warrior` |
| Distinct `elements.impacts.*` types | 65 | 60 in the quest tree alone |
| Distinct `elements.effects.*` types | 18 | |
| Distinct `elements.predicates.*` types | 11 | |
| Distinct `elements.addresseeFinders.*` types | 3 | |

The following PowerShell command produced the table on 2026-08-20. It reads the
source tree but writes nothing. `SARNAUT_GAME_DATA` names the owned 1.1.02.0
`game/data` directory.

```powershell
$dataRoot = $env:SARNAUT_GAME_DATA
if (-not (Test-Path -LiteralPath $dataRoot)) {
    throw 'Set SARNAUT_GAME_DATA to the 1.1.02.0 game/data directory'
}

$questRoot = Join-Path $dataRoot 'World\Quests\InstLeague1'
$scopeRoots = @($questRoot, (Join-Path $dataRoot 'Maps\Inst_LeagueStart'))
foreach ($top in 'Characters', 'Creatures', 'Items') {
    $scopeRoots += Get-ChildItem -Directory (Join-Path $dataRoot $top) -Recurse -Filter InstLeague1 |
        Select-Object -ExpandProperty FullName
}

$docs = @($scopeRoots | ForEach-Object {
    Get-ChildItem -File $_ -Recurse -Filter *.xdb
} | Sort-Object FullName -Unique | ForEach-Object {
    $text = Get-Content -Raw -LiteralPath $_.FullName
    $root = [regex]::Match(
        $text, '<(?<root>[A-Za-z_][A-Za-z0-9_.-]*)[\s>]'
    ).Groups['root'].Value
    [pscustomobject]@{ File = $_.FullName; Root = $root; Text = $text }
})

function RootCount([string]$suffix) {
    @($docs | Where-Object Root -like "*$suffix").Count
}
function TypeCount([string[]]$texts, [string]$family) {
    @([regex]::Matches(
        ($texts -join "`n"),
        "gameMechanics\.elements\.$family\.[A-Za-z0-9_]+"
    ) | ForEach-Object Value | Sort-Object -Unique).Count
}

$allText = $docs.Text -join "`n"
$questTreeDocs = @($docs | Where-Object File -like "$questRoot\*")
$questDocs = @($docs | Where-Object Root -like '*.QuestResource')
$skipped = @($questDocs | Where-Object {
    $_.Text -match 'type="gameMechanics\.elements\.quest\.QuestCountSpecial"'
}).Count
$objectives = [regex]::Matches(
    ($questDocs.Text -join "`n"),
    'type="gameMechanics\.elements\.quest\.(?<kind>QuestCountSpecial|QuestCountItem|QuestCountKill)"'
)

[ordered]@{
    Documents = $docs.Count
    Quests = $questDocs.Count
    Objectives = $objectives.Count
    QuestCountSpecial = @($objectives | Where-Object {
        $_.Groups['kind'].Value -eq 'QuestCountSpecial'
    }).Count
    QuestCountItem = @($objectives | Where-Object {
        $_.Groups['kind'].Value -eq 'QuestCountItem'
    }).Count
    QuestCountKill = @($objectives | Where-Object {
        $_.Groups['kind'].Value -eq 'QuestCountKill'
    }).Count
    ServableToday = $questDocs.Count - $skipped
    SkippedToday = $skipped
    TriggerResource = RootCount 'TriggerResource'
    QuestTreeTriggerRoots = @($questTreeDocs | Where-Object Root -like '*.TriggerResource').Count
    QuestTreeTriggerSuffixes = @($questTreeDocs | Where-Object {
        $_.File -like '*(TriggerResource).xdb'
    }).Count
    QuestCountId = RootCount 'QuestCountId'
    ScriptZone = RootCount 'ScriptZone'
    ScriptZonePlacements = [regex]::Matches(
        $allText, 'gameMechanics\.map\.scriptZone\.ScriptZoneElement'
    ).Count
    VariableResource = RootCount 'VariableResource'
    BuffResource = RootCount 'BuffResource'
    ChestResource = RootCount 'ChestResource'
    DoorResource = RootCount 'DoorResource'
    TextMessage = RootCount 'TextMessage'
    ClientData = RootCount 'ClientData'
    WarriorSpellAbilitySets = @(Get-ChildItem -Directory -LiteralPath (
        Join-Path $dataRoot 'Mechanics\Spells\Warrior'
    )).Count
    ImpactTypes = TypeCount $docs.Text 'impacts'
    QuestTreeImpactTypes = TypeCount $questTreeDocs.Text 'impacts'
    ImpactIncreaseQuestCountUses = [regex]::Matches(
        $allText,
        'type="gameMechanics\.elements\.impacts\.ImpactIncreaseQuestCount"'
    ).Count
    EffectTypes = TypeCount $docs.Text 'effects'
    PredicateTypes = TypeCount $docs.Text 'predicates'
    AddresseeFinderTypes = TypeCount $docs.Text 'addresseeFinders'
} | Format-Table -AutoSize
```

Two of those rows are load-bearing and were previously stated wrongly:

**The trigger corpus is discovered by type, not by name.** `Quest_3_20/TriggerAvatar.xdb`
and `Quest_3_20/TriggerTarget.xdb` are `TriggerResource` documents whose filenames carry
no `(TriggerResource)` suffix. Inside the quest tree, a filename-glob extractor finds
7 and silently misses the two documents that drive quest 3-20's counters. The tenth
trigger is the suffixed `Characters/HumMobs/Instances/InstLeague1/GibberSummon`
document. `sarnaut-extract` must dispatch on the
XML root element for every resource kind in this milestone.

**The servable baseline is 12, not 9.** Nine quests are *skipped*. Any regression gate
phrased against "the 9 servable quests" would cover three quarters of what it should.

### Why the interpreter is the milestone

Ten of the twelve extracted objectives are `quest-count-special`. Their counters are
moved only by `ImpactIncreaseQuestCount`, which appears 9 times, always inside a
`TriggerResource`, a `ScriptZone` or a quest's `startImpacts`/`rewardImpacts`. The same
node grammar is the spell system: `Mechanics/Spells/Warrior/AimedShot/Spell01.xdb` is
`casterConditions` (predicates) plus `targetImpacts`, and a `ScriptZone` is `filter`
plus `conditionsIn` (predicates) plus impacts. **One evaluator, four callers.** If the
representation is not decided in an ADR before the spell runtime exists, it gets shaped
implicitly by whatever the first caller needed, and M4 pays to generalise it.

But the surface is a script engine, not a lookup table: 65 impact types, not the ~50
that option analysis assumed. An enumerated build-time allow-list that hard-fails
`sarnaut-pack check` would force ADR 0036 to enumerate 65+ opcodes in week one or break
every pack build. The coverage policy must be tiered instead, and the *tutorial's reach*
must be the gate rather than the type census.

### The debts M3 must pay, and where they go

- ADR 0033 §3 bills the gateway deviation at "M3, or the first time a second shard or a
  second zone exists". M3 pays it.
- ADR 0033 §2 specifies a 9-row depguard allow-list; 3 rules exist. **And no row admits
  an interpreter**: `quests` may import only `{inventory, pack, gametypes}` and `world`
  only `{interest, combat, gametypes}`, so `internal/script` has no legal caller today.
  ADR 0033 line 120 calls an allow-list change "an architecture change". Adding the
  `script` row is therefore an amendment landed *with* the allow-list, not a row the
  interpreter task asserts for itself in passing.
- `internal/interest` still lives inside `world` carrying `TODO(SAR-19)`.
- `data/overlays/` sits in the private repo, so the project's own authored content is
  unpublishable.

### The coordinate frame is broken, and it blocks terrain

`Maps/Inst_LeagueStart/000_020/1_2_ServerObjects.xdb` holds placements at y 134–255;
the adjacent `1_3_ServerObjects.xdb` holds a single placement at y=0.82. Tile pitch is
256, so a global frame would put that object just past 255.47, not at 0.82. Coordinates
in `PatchObjects` are **patch-local**. Reconstructing globally — sector `000_020`
contributing 20x256=5120, tile `_2` contributing 512, plus local 241 — gives y≈5873 and
x≈317, against `quest-2-20.yaml`'s `returnLocation` of (300.10, **5814.42**, 0.00): the
same tile. Quest locations are already global; `sarnaut-extract` is emitting placements
without applying the sector/tile origin.

This is not a scale-factor problem and cannot be fixed by calibration alone. Any terrain
task that asserts "extracted spawn placements land within tolerance of sampled ground"
will fail against two different frames, and the symptom — characters sunk into or
floating above the floor — would surface weeks later and be expensive to diagnose. The
frame fix lands in the extractor, with a round-trip regression test, **before** anything
samples terrain against a placement.

Separately, the served version stores the tutorial map as
`clients/1.1.02.0/data/Packs/Inst_LeagueStart.Client.pak`; that tree contains only
`Packs/`, `Editor/`, `Mods/` and `Types/`, with no unpacked `Maps/`. Unpacked
`*_terrainDump.bin` files exist only in 2.0.04.49 and later trees, whose map appears
re-authored. `ao-godot-converter` has no pak reader. Converting 2.0 geometry and serving
it against the 1.1 zone is the same sunk-characters failure by another route.

### Stale specs that this milestone disproves

`quests.md` line 226 states "`quest-count-special` is not supported" and line 178 that a
disjunctive objective form "is not supported". `combat.md` line 22 defers mob damage
output, multi-attacker threat and player death to "the next spec", and its §3 range
constants (`ABILITY_RANGE_M` 10.0, `AGGRO_RADIUS_M` 12.0, `LEASH_RADIUS_M` 40.0) are
curated guesses against an unverified distance unit. There is no spells spec at all.
M3 implements all of it, so M3 writes the specs first rather than leaving three
falsified clauses standing behind a shipped milestone.

## Decision

### 1. M3 is the complete InstLeague1 League tutorial

**In scope.** All 21 extracted `InstLeague1` quests servable with
`content.skip_unsupported_quests` reporting **zero skips**, including all 10
`quest-count-special` objectives driven by the real extracted trigger documents. A
bounded impact/trigger/effect interpreter (`internal/script`) as the single evaluator
for quest impacts, triggers, script zones and spells. Real Warrior spells and abilities
from extracted documents. Two-sided combat with threat, leash, player death, respawn and
resurrection sickness. Server-authoritative terrain height so characters stand on the
ground. Buffs, devices and NPC movement to the extent the tutorial reaches them. Race,
class and stats from real documents, retiring the curated chargen overlay. The gateway
interposed with a zone-transfer-capable session model. The connective client UI.

**Out of scope.** The Lightwood levelling spine (extracted and converted; promoted in
M4). Empire and every class beyond what chargen needs. Party, guild, mail, auction,
trade, crafting, professions, PvP, reputation. Astral. Client prediction. The Lua addon
host. Launcher, installer and website. Retail combat-math fidelity.

### 2. The exit gate is `scripts/m3-tutorial-driver`

M3 is done when `scripts/m3-tutorial-driver` completes the full tutorial headless in CI
with per-step assertions and a checked-in run-report baseline: connect through the
gateway, create a League Warrior resolved against extracted `ClassRaceCombination` and
`Classes/Warrior.xdb`, accept and turn in all 21 quests, cast a real extracted Warrior
spell, interact with an `ElixirChest` device, observe a scripted `GoThroughPath` NPC
walk, die to a mob and respawn, and cross a level — with `sarnaut-quest-census`
reporting 21/21 servable and zero refused-tier script nodes reached. Alongside:
`go test -race ./...`, `make lint`, the C# and Rust suites, and migrations up/down/up
all green.

No acceptance criterion anywhere in M3 may be human-judged. "A human can play it" is the
*purpose*; the *gate* is the driver.

Three curated artifacts are **deleted** by the gate, not kept alongside:
`ability.melee.novice-cleave.yaml`, the curated level curve, and
`chargen.league.warrior.yaml`. `race.kania` and `class.warrior` resolve against real
documents or M3 is not done.

### 3. `sarnaut-quest-census` is the weekly progress metric

One number, in CI, from wave one: quests servable out of 21, with a committed baseline
recording today's true **12 of 21, 9 skipped**, a per-quest reason column, and JSON
output. Widening interpreter coverage is a reviewable diff in a checked-in per-tier
count table, not a quiet addition.

### 4. The interpreter is mandated, and it is tiered

`internal/script` is the single evaluator for impacts, predicates, effects and addressee
finders, with four callers: quest start/reward impacts, triggers, script zones, and
spells. It is specified in **ADR 0036 before any code that would shape it implicitly**.

Its coverage policy is three tiers, not an enumerated allow-list:

1. **implemented** — executes;
2. **inert-and-counted** — parsed, recorded, no effect, visible in the census;
3. **refused** — loud failure.

A refused-tier node reached by `m3-tutorial-driver` is a hard CI failure. That is the
mechanical gate, replacing a build-time opcode allow-list which at 65 impact types would
either break every pack build or be rubber-stamped.

`internal/script` imports only `gametypes` and `pack`. If honouring the tutorial's
`LifeGuard` effect requires `internal/combat` to import `internal/script`, **the seam is
wrong**: the remedy is to move the damage clamp behind an existing combat hook, never a
depguard exemption, which would quietly undo ADR 0033's containment guarantee at exactly
the moment the gateway task depends on it meaning something.

### 5. Sequencing constraints that are decisions, not preferences

- The full 9-row ADR 0033 §2 depguard allow-list, `internal/gametypes`, the
  `account`→`auth` and `store`→`charstore` renames, the `internal/interest` extraction,
  and the **ADR 0033 §2 amendment admitting `internal/script` with its caller edges**
  land together in one task holding an **exclusive lock on the server repo**, in wave
  one, before any other server work and long before the gateway.
- The gateway is last of the system work. Session-verb additions freeze once the last
  verb-adding task merges. Its acceptance is ADR 0033's own containment test, made
  mechanical: `git diff --stat` shows zero changes across the eight simulation modules.
- The coordinate-frame fix lands in `sarnaut-extract` with a round-trip regression test
  before any task samples terrain against a placement.
- The interpreter runs against the **12 already-servable quests as a pure no-change
  regression** — identical completion outcomes, rewards, state transitions and emitted
  events versus the pre-change baseline — before it is allowed to change any quest
  outcome. This is the cheapest available proof that ADR 0036's representation is right,
  and it lands before anything depends on its shape.
- One migration wave, designed from the specs up front, including the **persisted
  deferred-impact queue**: `Quest_3_10/TimerBuff` carries a 60000 ms duration and
  `ImpactsDeferred` delays of 43500, 5000, 3000 and 58000 ms, so without persistence a
  restart or disconnect silently drops a scripted beat mid-quest.
- Specs precede their implementations: `quests.md` v2, `combat.md` v2 and a new
  `spells.md` land before the code that falsifies their current text.

### 6. The cut ladder is binding

Cut **in this order**, top first. A task may not be cut out of order without amending
this ADR.

1. **English locale overlay** — fall back to `ru` plus canonical slug. The fallback chain
   stays implemented and tested either way; gaps render visibly rather than blank.
2. **Paper Harbor shipped profile and its CI fixture** — keep ADR 0037 and the
   public-artifact provenance scan. The legal posture survives; only the showable
   zero-IP demo is lost. (Not "defer Paper Harbor entirely": the split and the scan are
   owner-required and are what actually protect the project.)
3. **Voice and audio playback for `ImpactClientData`** — hint text panels only.
4. **Periodic `ImpactsOverTime` / `EntityImpactsOverTime`** — buffs keep attach, detach,
   duration and expiry; the periodic driver goes behind its disable flag. The periodic
   tier in the tutorial is ambient elf chatter.
5. **Zone transfer out of InstLeague1** — cut to a refusing stub; keep gateway
   interposition, the session split and the routing table. ADR 0033's bill is still paid;
   only the second destination defers to M4.
6. **Static-mesh collision** — drop to heightfield ground sampling plus authored blocking
   volumes around the corridors that actually gate the tutorial path.
7. **`ScriptZone` coverage** — from all 32 zones to only those the 21 quests require.
   Ambient scripted atmosphere is lost; quest-driving scripts are kept.
8. **Client feedback polish** — cut the cast bar and threat / target-of-target display;
   keep quest counters, scripted chat cues and the death/respawn UI.
9. **Warrior content breadth** — from all 7 extracted spell/ability document sets to what
   `Classes/Warrior.xdb` grants at level 1. The interpreter path is unchanged.
10. **Terrain walkability refusal** — keep ground snap, lose slope and walkability
    refusal. Characters stand on the ground; they can walk up cliffs.
11. **`GoThroughPath` fidelity** — NPCs teleport to the path endpoint instead of walking
    it, behind the flag that task already builds. Last resort: this is the rung that most
    visibly makes the tutorial worse, and cutting it means the milestone shipped a
    scripted scene that does not play.

**NEVER CUT.** These are not on the ladder at any depth:

- the interpreter core, its pure no-change regression gate, and `quest-count-special`
  becoming servable;
- all 21 quests servable with `content.skip_unsupported_quests` reporting zero skips;
- the nine ADR 0033 §2 depguard module rows, plus the `script` row amendment;
- ADR 0033's gateway containment test (zero diff across the eight simulation modules);
- the coordinate-frame fix and its round-trip regression test;
- `scripts/m3-tutorial-driver` as the exit gate.

If the ladder is exhausted and M3 still overruns, the milestone slips. It does not
re-scope into the NEVER CUT list.

## Consequences

**The interpreter buys a stretch of invisible work.** Six tasks — ADR 0036, the content
proto rows, the script-graph extractor, the pack carriage, the interpreter core, and the
pure regression — land no player-visible content, and the regression is *deliberately* a
no-change proof. The mitigation is structural, not optimistic: the terrain track and the
two-sided-combat track are dependency-independent of all six and must be scheduled
concurrently. If agent-pool pressure forces serialisation, serialise the interpreter and
keep the visible tracks moving. `sarnaut-quest-census` gives the owner a number every
week regardless.

**ADR 0036's representation choice is the riskiest one-time decision in the project.** If
the chosen node representation proves wrong under the nesting cases, six tasks rework.
The golden fixture in the content-proto task is the smallest end-to-end proof and the
golden call traces over all 10 trigger documents are the second gate; a representation
problem should surface there, not in the spell runtime.

**Wave one takes an exclusive lock on the server repo.** The rename task touches nearly
every file. No other server work may be in flight during it, and the extractor and census
tasks are deliberately scoped out of `server/` so they can run concurrently.

**The gateway is a merge tail by construction.** It depends on every task that adds a
session verb so that splitting `internal/session` is a one-time cut against a settled
dispatch table. The direct-to-shard path stays behind a config toggle until the driver is
green on both, and is deleted in the same PR that proves it.

**Some things this ADR mandates are unbudgeted.** NPC movement has never existed in this
project — no pathfinding, no route consumption. Authored blocking volumes, if cut-ladder
rung 6 fires, land on the content lane rather than the server lane. Both are named on the
ladder for that reason.

**Creating the public `content` repo is owner-gated.** No agent can create it. The move is
staged on a branch behind a single blocking checklist item; if repo creation lags, cut
ladder rungs 1 and 2 are the first casualties, which is why they are rungs 1 and 2.

**Supersedes and amends.** This ADR extends ADR 0003's milestone definitions past M2 and
resolves every stale ADR 0003 queue pointer in the specs. It triggers an amendment to
ADR 0033 §2 (the `internal/script` row and its caller edges) and pays ADR 0033 §3's
gateway bill. It is the parent of ADRs 0036 (script runtime), 0037 (public/private
content split), 0038 (coordinate frame and distance calibration) and 0039 (gateway and
zone-transfer session model).

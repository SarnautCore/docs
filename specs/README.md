# Specifications

Behavioral specifications distilled from studying the retail game — file formats,
protocol semantics, mechanics formulas, system state machines — plus the rules
SarnautCore invents where retail behaviour is not on the current milestone's critical
path.

Per [ADR 0011](../adr/0011-clean-room-reimplementation-rule.md), these documents carry
**facts** (semantics, layouts, formulas, constants) extracted from reference material.
Implementations in `server`/`client` are written from these specs, never from
decompiled code directly. Specs must contain no MY.GAMES content (no extracted
strings, art, or data tables).

## The two rules

**1. Every mechanic is implemented from a spec section, and every pull request says
which one.** A pull request that adds or changes game mechanics, protocol behaviour,
or a data format must name the spec and the numbered rule it implements — for example
`implements specs/mechanics/loot.md §5.4` — in its description, and in a comment on
the function that owns the behaviour. If no section covers it, the spec is written
first, in the same pull request or an earlier one. "I read the reference and wrote the
code" is not an accepted path; that is exactly the derivation ADR 0011 forbids.

Reviewers check the citation against the spec. A citation that does not match what the
code does is a review blocker, whichever of the two is wrong.

**2. Every number is attributed.** A constant in a spec is either cited to a concrete
source data-file path and field, or carries the literal label **curated SarnautCore
decision**. There is no third category. An unlabelled magic number does not ship, in a
spec or in the code that implements it.

## Writing a spec

Copy [`TEMPLATE.md`](TEMPLATE.md). It fixes the section order, the constants table with
its mandatory provenance column, the numbered-pseudocode convention that rule 1 above
depends on, and the limits on what a Sources section may contain.

Numbering is stable once a spec is accepted. Amend a rule in place and note it in the
change log; do not renumber, because pull requests, code comments, and tests cite the
numbers.

## Curated versus derived

Some specs — the M2 mechanics ones especially — describe rules SarnautCore invented
rather than rules recovered from retail. That is deliberate, not a shortfall. Invented
rules carry no legal exposure and no spec debt: replacing one later rewrites two
sections of one document. Real retail math is queued behind later specs per
[ADR 0003](../adr/0003-milestones-m0-m1-m2.md), which allows only critical-path work
per milestone. Each spec says in its §1 which of its rules are which, and its §7 lists
what would settle the open ones.

## Index

### `mechanics/` — combat math, stats, progression, economy systems

| Spec | Milestone | Status |
|---|---|---|
| [`mechanics/combat.md`](mechanics/combat.md) | M2 | Draft |
| [`mechanics/loot.md`](mechanics/loot.md) | M2 | Draft |
| [`mechanics/quests.md`](mechanics/quests.md) | M2 | Draft |

### `protocol/` — session, replication, and message semantics

| Spec | Milestone | Status |
|---|---|---|
| [`protocol/session.md`](protocol/session.md) | M2 | Draft |

### Areas not yet started

- `formats/` — file/container formats (complementing the converter repos' docs)
- `world/` — maps, spawning, AI/patrol behavior

# Spec NNNN — <Title>

**Status**: Draft | Accepted | Superseded by <spec>
**Milestone**: M0 | M1 | M2 | later
**Owner**: <name>
**Last reviewed**: YYYY-MM-DD

Copy this file to start a new spec. Delete the guidance comments as you fill each
section in. Every section below is mandatory; write "None" rather than removing a
heading, so a reader can tell the difference between "nothing here" and "the author
forgot".

## 1. Scope

What this spec covers and, explicitly, what it does not. Name the sibling specs
that own the excluded parts so an implementer knows where to look next.

State up front whether the rules in this spec are **derived from reference data**
or are **curated SarnautCore decisions**. A spec may be both, but every individual
rule must be attributable to one or the other, and §3 is where that attribution
lives.

## 2. Vocabulary

Terms this spec uses in a narrow sense, defined once. Prefer the project's canonical
English IDs (ADR 0007). If a term already has a definition in another spec, link to
it instead of redefining it.

## 3. Constants

Every number this spec relies on appears in this table and nowhere else. The body
of the spec refers to constants by name. This is a hard rule: a bare numeral in the
pseudocode or prose is a defect, except for arithmetic that a worked example derives
from constants already in this table.

Each row's **Provenance** is exactly one of:

- a repository-relative or reference-root-relative **data-file path** plus the field
  the value came from, or
- the literal words **curated SarnautCore decision**, optionally followed by a short
  rationale.

There is no third option. An unattributed number does not ship.

| Constant | Value | Unit | Provenance |
|---|---|---|---|
| `EXAMPLE_LIMIT` | 20 | items | `Items/Mechanics/Example.xdb` — `<stackLimit>` |
| `EXAMPLE_WINDOW` | 30 | seconds | curated SarnautCore decision — placeholder until §7 is resolved |

## 4. Model

The nouns: entities, records, and their fields, with types. Give the shape an
implementer needs before they can read §5. Field names here are the names the Go
implementation should use, so choose them deliberately.

## 5. Rules

Numbered pseudocode. The numbering is load-bearing: pull requests cite it (see
`README.md`), and later amendments reference it. Rules are written so an implementer
can transcribe them without going back to the reference material.

Conventions:

- Steps are numbered `5.1`, `5.2`, … Sub-steps nest as `5.2.1`.
- Loops, branches, and early returns are explicit. "Handle the error case" is not a
  step; "return `ErrBagFull` and roll back the transaction" is.
- Every source of randomness names the stream it draws from and the order it draws
  in, so that a fixed stream produces a fixed result.
- Rounding is named at the point it happens (`round_half_up`, `floor`, `ceil`).

## 6. Worked examples

At least one example with concrete inputs and an exact expected output, chosen so
that it can become a unit test verbatim. Show the arithmetic step by step, not just
the answer — the intermediate values are what a failing test needs to localise the
bug.

If the mechanic is random, the example fixes the random stream explicitly and states
the draw order, so the expected output is a single value rather than a distribution.

## 7. Open questions and placeholders

Facts that are not yet established, with what would settle each one. A placeholder
section is not a failure; an undocumented guess is. Where a constant in §3 is a
stand-in, say what evidence would replace it and what the blast radius of the change
would be.

## 8. Sources

Data-file paths under the reference roots (ADR 0009) and observable in-game
behaviour, only. Per ADR 0011:

- Cite **paths** and the **field or element names** at those paths.
- Do **not** quote, paraphrase, or transliterate decompiled code.
- Do **not** copy game data — no localization strings, no art, no bulk tables. A
  handful of individual field values that a formula depends on belongs in §3 with
  its path; a transcribed table does not belong here at all.
- Structural observations ("the root element is X in N of M files") are facts about
  the data's shape and are fine.

## 9. Change log

| Date | Change |
|---|---|
| YYYY-MM-DD | Created. |

# ADR 0006 — YAML authored data, JSON-Schema validated, compiled runtime packs

**Status**: Accepted (2026-08-20)

## Context

The gameplay is data-driven; the question was YAML vs JSON. Humans hand-tune game
design data and need comments and reviewable diffs; servers need fast, strict loads.

## Decision

- **YAML is the authored source of truth** for all game-design data.
- Every document type is **validated by JSON Schema** (schemas live in `data-schemas`).
- A **data compiler** in `tools` compiles YAML into a compact runtime format
  (JSON or binary packs) that server and client load. Humans never edit the compiled
  form; runtime never parses YAML.
- The compile step also runs cross-reference integrity checks
  (quest → NPC → spawn → map).

## Consequences

- A schema + compiler must exist before meaningful data authoring starts (M0 scope).
- CI validates every data change against schemas and referential integrity.

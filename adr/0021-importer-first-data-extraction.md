# ADR 0021 — Importer-first data extraction with overlay curation

**Status**: Accepted (2026-08-20)

## Context

The canonical catalogue is far too large to hand-author (tens of thousands of
items, thousands of quests/spawn tables). But extracted data needs human fixes,
balance, and English IDs — which re-extraction must never destroy.

## Decision

- **Extractors generate the canonical YAML** from the source XDB trees, every file
  carrying provenance (source path/xpointer, extractor version).
- **Human curation is an overlay**: fixes and additions live as patch layers applied
  on top of generated data by the data compiler; re-running extractors regenerates
  the base layer without touching curation.

## Consequences

- The data compiler (ADR 0006) implements layered merging with conflict reporting.
- Deterministic extraction (stable ordering, stable slugs per ADR 0007) is a
  correctness requirement, not a nicety.

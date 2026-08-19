# ADR 0007 — English canonical IDs; localization as data

**Status**: Accepted (2026-08-20)

## Context

The original game data is Russian-first with English localization layered on. A
mixed-language catalogue of identifiers would be unreadable and unmigratable later.

## Decision

- All canonical identifiers in data, code, and schemas are **English slugs**
  (e.g. `zone.lightwood`, `npc.gibberling_scout`).
- Display strings are **localization data** (`ru`, `en` to start), extracted from
  the clients' localization packs and keyed by canonical ID.

## Consequences

- Extractors must generate stable English slugs during import (with provenance to
  the original resource paths), and slug generation must be deterministic so
  re-extraction doesn't churn IDs.
- Classic 1.1 display text lives in the 1.1 *client* LocPacks (absent from the 1.1
  server tree), so the extraction pipeline needs the client loc-pack reader early.

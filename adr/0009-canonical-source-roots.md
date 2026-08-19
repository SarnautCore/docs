# ADR 0009 — Canonical source roots

**Status**: Accepted (2026-08-20)

## Context

Nine-plus reference trees exist locally (clients 1.1/2.0/3.0/4.0/14.0/15.0/17.0,
server trees 1.1/3.0/4.0/7.0/17.0). Surveying showed the 17.0 client is a strict
superset of 14.0 (same tree grammar, ~half the archives bit-identical), and that
the extracted server trees carry the loose XDB data the pipeline consumes.

## Decision

Four canonical bulk sources:

- **Classic**: the 1.1 server tree's game data (plus its protocol research and
  decompiled reference, per ADR 0010/0011), with the **1.1 client** used for
  localization packs, UI Lua, and client-side assets.
- **Modern**: the **7.0 server tree** (mechanics catalogue + display text) and the
  **17.0.01.49 client** (assets, resource-type spill, exploded localization).

All other trees (2.0, 3.0, 4.0, 14.0, 15.0) are secondary: consulted on demand for
gap-filling, never bulk-processed. 14.0 is skipped entirely in favor of 17.0.

## Consequences

- Converters/extractors point at these four roots; profiles for secondary trees
  remain functional but unused by default.
- All reference trees are read-only (charter sources policy).

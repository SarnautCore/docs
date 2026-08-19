# ADR 0002 — Classic 1.1 as the content spine; mechanics as a superset

**Status**: Accepted (2026-08-20)

## Context

The project targets both classic and modern eras. Content (zones, quests, NPCs,
itemization) diverges hard between them; mechanics (systems) largely accumulate.
A canonical catalogue needs one spine, or it becomes two half-maintained games.

## Decision

- **Classic 1.1 is the spine**: world, quests, zones, itemization from the 1.1 era.
- **Mechanics are a superset**: systems from later versions (e.g. battle mounts on
  top of mounts) are implemented as additive, data-driven modules. Systems live in
  code; whether a given era enables them lives in data.
- **Later-era content ships as content packs** per zone/system, importable alongside
  the classic spine, not merged into it.

## Consequences

- The 1.1 server data tree is the primary mechanics/content catalogue; 7.0 is the
  primary source for later-era systems and display text.
- The data schema must support era/pack tagging from day one.
- Modern (17.0) assets are converted and stored on equal footing with classic ones,
  even though classic content is the default world.

# Specifications

Behavioral specifications distilled from studying the retail game — file formats,
protocol semantics, mechanics formulas, system state machines.

Per [ADR 0011](../adr/0011-clean-room-reimplementation-rule.md), these documents
carry **facts** (semantics, layouts, formulas, constants) extracted from reference
material. Implementations in `server`/`client` are written from these specs, never
from decompiled code directly. Specs must contain no MY.GAMES content (no extracted
strings, art, or data tables).

Structure grows as specs land; expected areas:

- `formats/` — file/container formats (complementing the converter repos' docs)
- `protocol/` — session, replication, and message semantics
- `mechanics/` — combat math, stats, progression, economy systems
- `world/` — maps, spawning, AI/patrol behavior

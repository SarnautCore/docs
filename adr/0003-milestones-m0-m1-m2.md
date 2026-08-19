# ADR 0003 — Milestones: M0 Pipelines → M1 Zone Walkabout → M2 Vertical Slice

**Status**: Accepted (2026-08-20)

## Context

"Recreate the entire game" needs a first concrete target that forces every pipeline
(asset conversion, data extraction, netcode, client rendering, server sim) to exist
end-to-end, staged so each stage is demoable.

## Decision

- **M0 — Pipelines & Foundations**: toolchain installed; local infra (Compose stack,
  Perforce); content-addressed asset store; bulk conversion of classic 1.1 and modern
  17.0 assets; first XDB→YAML extractors proving the data schema; icon upscaling pilot.
  Exit: a Godot project where converted assets are browsable in an asset viewer.
- **M1 — Zone Walkabout**: QUIC+protobuf netcode online; one real converted zone
  streamed and walkable; NPCs placed from real spawn data.
- **M2 — Vertical Slice**: login → character creation (one race/class) → spawn on a
  starting isle → move → kill one mob with one ability → loot → complete one quest.

## Consequences

- Work is planned and tracked against these milestones (Linear projects mirror them).
- Anything not on the critical path of the current milestone is queued, not started.

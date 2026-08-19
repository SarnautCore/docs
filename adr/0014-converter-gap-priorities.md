# ADR 0014 — Converter gap priorities

**Status**: Accepted (2026-08-20)

## Context

The converter audit found specific gaps. The M0→M2 milestones need some closed
early; others can wait or be skipped.

## Decision

Priority order:

1. **Layered terrain materials** (splat/lightmap composition) — blocks a
   good-looking zone walkabout (M1).
2. **De-hardcode the version profiles** — converters read roots from configuration
   and integrate with the asset store instead of `E:\allods\...` constants.
3. **LOD > 0 native export.**
4. **Blender animation export** (glTF animation samplers).
5. **FMOD proprietary codec decode — skipped**; lossless passthrough is sufficient.

## Consequences

- Items 1–2 are M0/M1 scope; 3–4 are scheduled on demand; 5 is a recorded non-goal
  unless audio work surfaces a need.

# ADR 0018 — Client: C#-first; C++ GDExtension only for proven hot paths

**Status**: Accepted (2026-08-20)

## Context

Godot supports C# and C++ GDExtensions. The asset converter already emits a
C#-based Godot project. Premature C++ splits development across two toolchains
without measured need.

## Decision

- **C# for all gameplay, UI glue, and netcode.**
- **C++ GDExtension only for profiler-proven hot paths** (candidates: asset
  streaming/decompression, terrain paging) — decided by measurement, never upfront.
- In-game UI rendered with Godot Controls; UI scripting surface per ADR 0019.

## Consequences

- The client repo starts with zero C++; adding a GDExtension requires a profile
  trace justifying it.

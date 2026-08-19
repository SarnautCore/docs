# ADR 0008 — Latest-stable toolchain policy, eager upgrades

**Status**: Accepted (2026-08-20)

## Context

The stack targets modern versions (Godot 4.x + C#/GDExtension, Blender 5.x,
PostgreSQL 18, Go, .NET). Riding dev/beta snapshots breaks GDExtension ABI
mid-cycle; freezing on old versions rots a multi-year project.

## Decision

Pin each repo to the **latest stable release** of its toolchain, recorded in that
repo's toolchain/version files, and **upgrade eagerly** when new stables land.
Dev snapshots may be trialed in branches, never on main.

## Consequences

- Every repo carries explicit version pins (e.g. `go.mod` toolchain, `global.json`,
  Godot version in project settings, rust-toolchain.toml).
- Upgrades are routine chores, done promptly and verified by CI, not deferred.

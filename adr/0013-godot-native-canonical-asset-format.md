# ADR 0013 — Godot-native output is the canonical game-ready form; glTF is the DCC bridge

**Status**: Superseded (2026-08-21) by
[ADR 0040](0040-materialized-native-content-architecture.md)

Godot-native output remains the game-ready form, but it is now plain Godot scenes,
meshes, and textures with no custom C# loaders, produced by an offline bake and
committed as product content. `ao-godot-converter` is private internal machinery
rather than a product component.

## Context

`ao-godot-converter` (~11k lines, validated per-version) already emits a complete
Godot project: textures, meshes, skeletons, animations, character dress scenes,
terrain, particles, UI, audio, localization. `ao-blender-converter` emits per-asset
glTF `.glb` for DCC work. Making glTF the canonical interchange would discard the
specialized handling (skinning palette quirks, dress pass, terrain) for purity.

## Decision

- `ao-godot-converter`'s **Godot-native output** (`.tres`/`.tscn`/PNG/skmesh with
  its C# loaders) is the **canonical game-ready form** stored in the asset store.
- **glTF via `ao-blender-converter` exists solely as the DCC bridge** for artists
  doing Blender work; round-tripping artist work back into the game goes through
  standard Godot import.

## Consequences

- Converter improvements land in `ao-godot-converter` first (see ADR 0014).
- The Blender converter's gaps (no animation export) are prioritized only as artist
  workflows demand.

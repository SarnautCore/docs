# ADR 0004 — Per-domain repository topology

**Status**: Accepted (2026-08-20)

## Context

Client (Godot/C#), server (Go), web (Next.js), launcher (Tauri), and a Rust asset
pipeline have disjoint toolchains and CI needs; a single monorepo fights every tool.
Many fine-grained repos fragment ownership. The `data` repo may need a different
visibility than the code (see ADR 0001).

## Decision

Eleven repositories in the **SarnautCore** GitHub organization, bare names, local
checkouts flat under `E:\SarnautCore\<repo>`:

`server` · `client` · `tools` · `ao-godot-converter` · `ao-blender-converter`
(both transferred in) · `data` (**private**) · `data-schemas` · `web` · `launcher`
· `infra` · `docs`.

Non-repo working areas: `E:\SarnautCore\assets\` (Perforce workspace for the asset
store) and `E:\SarnautCore\.cache\` (extraction scratch).

All repos are public from day one except `data`.

## Consequences

- Cross-repo contracts (protobuf definitions, schemas) need a declared home and
  versioning discipline; wire contracts live in `server`, data schemas in
  `data-schemas`.
- Art/binary assets never live in git; they live in the Perforce depot (ADR 0022).

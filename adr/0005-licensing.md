# ADR 0005 — Licensing

**Status**: Accepted (2026-08-20)

## Context

The nightmare scenario for a fan project is a closed commercial fork (e.g. a
pay-to-win private server built on this code). AGPL-3.0 is the standard defense:
anyone running a modified public server must publish their modifications.
Tools and converters benefit from maximum adoption instead.

## Decision

| Scope | License |
|---|---|
| `server`, `client`, `data-schemas`, `web` | **AGPL-3.0** |
| `tools`, `ao-godot-converter`, `ao-blender-converter`, `launcher`, `infra` | **Apache-2.0** |
| `docs` | **CC-BY-4.0** |
| `data` | private, no license granted |

## Consequences

- Contributions to AGPL repos are accepted under AGPL; no CLA for now (solo project).
- Code may move from an Apache repo into an AGPL repo, not the other way around,
  unless the author (us) relicenses it.
- License or legal-posture changes are a hard stop requiring the owner's explicit
  decision (ADR 0025).

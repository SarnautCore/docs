# ADR 0001 — Clean-room, bring-your-own-client posture

**Status**: Accepted (2026-08-20)

## Context

Allods Online's assets and game data are owned by MY.GAMES. Fan projects that
redistribute converted proprietary assets are the classic cease-and-desist target.
Projects that ship only code and ingest the user's own copy (OpenMW, devilutionX,
OpenRCT2) have survived for decades.

Options considered: (a) clean-room kit that never publishes assets; (b) publish
converted asset packs via CDN; (c) code public, assets via private channels.

## Decision

**(a) Clean-room kit.** All code — server, client, converters, tools — is public
open source. Converted assets and extracted game data are **never published**.
The kit ingests a client installation the user already owns; the launcher/converter
builds asset packs locally. Public releases contain zero MY.GAMES IP.

Locally, the project keeps a full converted asset store (Perforce depot) for
development; the constraint applies to distribution, not to the working setup.

## Consequences

- The `data` repository (extracted/derived content) is private; a public
  `data-schemas` repo carries schemas and a hand-made demo dataset.
- Release engineering must build a "local bake" path: user's client → converter →
  playable data, fully automated.
- CDN plans apply to our own binaries and tools only.

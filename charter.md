# SarnautCore Charter

## What this is

A fan-driven, non-commercial, open-source recreation of *Allods Online* as a kit:
a Godot (C#) client, a Go game server, and the data/asset pipeline connecting them
to a client installation the user already owns. The project aims to preserve and
recreate both the **classic** (1.1-era) and **modern** (17.0-era) incarnations of
the game, with classic 1.1 as the canonical content spine.

## Goals

- A faithful, playable recreation of classic Allods Online gameplay, world, and content.
- A mechanics engine that is a **superset** across game eras (e.g. mounts *and* battle
  mounts), implemented data-first: systems in code, content in data.
- A fully documented asset and data pipeline so anyone with a legal client copy can
  build the complete game locally.
- Openly documented findings: file formats, protocol semantics, and mechanics behavior
  published as specifications (facts), enabling future preservation work.

## Non-goals

- **No commercial use.** Nothing in this project is sold, monetized, or donation-gated.
- **No asset redistribution.** No MY.GAMES art, audio, maps, text, or game-design data is
  ever published — not in repos, releases, CDN, or artifacts. The `data` repository, which
  holds extracted/derived content for development, is private.
- **No production deployment yet.** Until explicitly decided otherwise, nothing is deployed
  to public-facing infrastructure.
- Not an emulator of the retail server binary, and not a continuation of any private-server
  codebase.

## Sources policy

- The reference trees under `E:\allods\` (retail clients, server distributions, prior
  research) are **read-only**. Tools read from them; nothing ever writes to them.
- Compiled retail server code (Java bytecode) and any prior reimplementation code are used
  **only to extract facts** — formulas, constants, state machines, packet and data semantics.
  Those facts are written into the specifications in this repository, and our implementations
  are built **from the specs**. Literal translation of decompiled code into project code is
  prohibited (see [adr/0011](adr/0011-clean-room-reimplementation-rule.md)).
- Retail data catalogs (quests, spawns, loot, routes, stats) are extracted into the private
  `data` repository with provenance, and never published.

## Operating agreements

- Decisions are recorded as ADRs in this repository; an ADR is the authoritative record.
- Continuous autonomous operation is bounded by the hard stops in
  [adr/0025](adr/0025-continuous-operation-hard-stops.md).

# SarnautCore — Documentation

SarnautCore is a **fan-driven, non-commercial, open-source recreation kit** for the MMORPG
*Allods Online*: a new game client built on Godot (C#) and a new game server written in Go,
driven by an openly documented, data-first content pipeline.

## Clean-room, bring-your-own-client

This project **does not distribute any Allods Online assets or game data**. All art, audio,
maps, text, and game-design data remain the property of MY.GAMES and are never published by
this project. The kit is a set of tools and engines: it **ingests a client installation you
already own**, converts its assets locally on your machine, and runs the game from that
locally built data. Public releases contain zero MY.GAMES intellectual property.

See [charter.md](charter.md) for the full goals, non-goals, and sources policy, and
[adr/0001](adr/0001-clean-room-byo-client-posture.md) for the founding decision.

## Repository map (github.com/SarnautCore)

| Repo | Purpose | License |
|---|---|---|
| [server](https://github.com/SarnautCore/server) | Game server (Go): modular shard, auth, gateway | AGPL-3.0 |
| [client](https://github.com/SarnautCore/client) | Game client (Godot, C#) | AGPL-3.0 |
| [tools](https://github.com/SarnautCore/tools) | Asset store, extraction & conversion pipeline | Apache-2.0 |
| [ao-godot-converter](https://github.com/SarnautCore/ao-godot-converter) | Allods resources → Godot project converter (Rust) | Apache-2.0 |
| [ao-blender-converter](https://github.com/SarnautCore/ao-blender-converter) | Allods geometry/textures → glTF for DCC work (Rust) | Apache-2.0 |
| [data](https://github.com/SarnautCore/data) | Extracted & curated game-design data | **private** |
| [data-schemas](https://github.com/SarnautCore/data-schemas) | JSON Schemas + hand-made demo dataset | AGPL-3.0 |
| [web](https://github.com/SarnautCore/web) | Web/admin/GM tools (TypeScript, Next.js) | AGPL-3.0 |
| [launcher](https://github.com/SarnautCore/launcher) | Game launcher (Tauri 2, React/TypeScript) | Apache-2.0 |
| [infra](https://github.com/SarnautCore/infra) | Compose, Kubernetes/Agones, CI | Apache-2.0 |
| [docs](https://github.com/SarnautCore/docs) | This repository: charter, ADRs, specs | CC-BY-4.0 |

## Milestones

- **M0 — Pipelines & Foundations**: toolchain, local infrastructure, content-addressed asset
  store, bulk conversion of classic (1.1) and modern (17.0) assets, first data extractors,
  a Godot asset viewer browsing converted assets.
- **M1 — Zone Walkabout**: netcode online (QUIC + protobuf), a real converted zone streamed
  and walkable with characters, NPCs placed from real spawn data.
- **M2 — Vertical Slice**: login → character creation (one race/class) → spawn on a starting
  isle → move → kill one mob with one ability → loot → complete one quest.

## Contents of this repo

- [charter.md](charter.md) — goals, non-goals, sources policy.
- [adr/](adr/) — Architecture Decision Records, the project's constitution.
- [specs/](specs/) — behavioral specifications distilled from reverse-engineering research.

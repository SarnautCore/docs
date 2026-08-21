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
- [specs/world/m2-demo-runbook.md](specs/world/m2-demo-runbook.md) — developer-machine Godot walkthrough for the complete M2 slice.

### Architecture Decision Records

The full index is in [adr/README.md](adr/README.md). ADRs 0001-0025 set the project up;
0026-0033 are the M2 foundation:

| # | Decision |
|---|---|
| [0026](adr/0026-wire-message-envelope.md) | Wire message envelope, channel assignment, snapshot ownership |
| [0027](adr/0027-proto-contract-and-wire-evolution.md) | Cross-repo proto contract, drift control, wire evolution |
| [0028](adr/0028-world-sim-protobuf-boundary.md) | World sim owns domain types; protobuf stops at the session layer |
| [0029](adr/0029-runtime-pack-format.md) | Compiled runtime pack format and overlay merge semantics |
| [0030](adr/0030-auth-account-service-session-tokens.md) | Auth service: Argon2id, opaque tokens, NATS admission |
| [0031](adr/0031-persistence-and-migrations.md) | Persistence: PostgreSQL, goose migrations, checkpoint saves |
| [0032](adr/0032-character-creation.md) | Character creation from a compiled chargen pack table |
| [0033](adr/0033-m2-module-topology-gateway-deviation.md) | M2 shard module topology via depguard; gateway deviation |

### Specifications

How to write one, and the two rules every mechanics pull request answers to, are in
[specs/README.md](specs/README.md); start a new spec from
[specs/TEMPLATE.md](specs/TEMPLATE.md).

| Spec | Covers | Milestone |
|---|---|---|
| [mechanics/combat.md](specs/mechanics/combat.md) | Derived stats, targeting, GCD, damage, aggro, death and respawn | M2 |
| [mechanics/loot.md](specs/mechanics/loot.md) | Loot-table evaluation, stack splitting, kill credit and ownership | M2 |
| [mechanics/quests.md](specs/mechanics/quests.md) | Quest state machine, objectives, prerequisites, the reward grant | M2 |
| [protocol/session.md](specs/protocol/session.md) | Connect, admission, `EnterZone`, the command loop, save checkpoints | M2 |

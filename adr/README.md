# Architecture Decision Records

The project's constitution. Each ADR records one decision: status, context,
decision, consequences. New decisions get the next number; superseded ADRs are
marked, never deleted.

| # | Decision |
|---|---|
| [0001](0001-clean-room-byo-client-posture.md) | Clean-room, bring-your-own-client posture |
| [0002](0002-classic-11-content-spine.md) | Classic 1.1 content spine; mechanics superset |
| [0003](0003-milestones-m0-m1-m2.md) | Milestones M0 → M1 → M2 |
| [0004](0004-per-domain-repo-topology.md) | Per-domain repository topology |
| [0005](0005-licensing.md) | Licensing |
| [0006](0006-yaml-source-compiled-runtime-data.md) | YAML source → schema-validated → compiled runtime packs |
| [0007](0007-english-canonical-ids-localization-as-data.md) | English canonical IDs; localization as data |
| [0008](0008-latest-stable-toolchain-policy.md) | Latest-stable toolchain policy |
| [0009](0009-canonical-source-roots.md) | Canonical source roots |
| [0010](0010-protocol-posture.md) | Clean protocol now; retail-compat door open |
| [0011](0011-clean-room-reimplementation-rule.md) | Clean-room reimplementation rule |
| [0012](0012-content-addressed-asset-store.md) | Content-addressed asset store |
| [0013](0013-godot-native-canonical-asset-format.md) | Godot-native canonical asset format; glTF as DCC bridge |
| [0014](0014-converter-gap-priorities.md) | Converter gap priorities |
| [0015](0015-icon-upscaling.md) | Icon upscaling as stored variants |
| [0016](0016-server-modular-monolith.md) | Server: modular monolith + auth + gateway |
| [0017](0017-quic-protobuf-transport.md) | QUIC + protobuf behind a transport seam |
| [0018](0018-client-csharp-first.md) | Client C#-first; C++ only for proven hot paths |
| [0019](0019-fresh-lua-addon-api.md) | Fresh AO-inspired Lua addon API |
| [0020](0020-server-scripting-gopher-lua.md) | Server scripting: data-driven + gopher-lua |
| [0021](0021-importer-first-data-extraction.md) | Importer-first extraction with overlay curation |
| [0022](0022-perforce-docker-depot-g.md) | Perforce p4d in Docker, depot on G: |
| [0023](0023-linear-structure.md) | Linear structure |
| [0024](0024-ci-runners.md) | CI runners |
| [0025](0025-continuous-operation-hard-stops.md) | Continuous operation with hard stops |
| [0026](0026-wire-message-envelope.md) | Wire message envelope, channel assignment, snapshot ownership |
| [0027](0027-proto-contract-and-wire-evolution.md) | Cross-repo proto contract, drift control, wire evolution |
| [0028](0028-world-sim-protobuf-boundary.md) | World sim owns domain types; protobuf stops at the session layer |
| [0029](0029-runtime-pack-format.md) | Compiled runtime pack format and overlay merge semantics |
| [0030](0030-auth-account-service-session-tokens.md) | Auth service: Argon2id, opaque tokens, NATS admission |
| [0031](0031-persistence-and-migrations.md) | Persistence: PostgreSQL, goose migrations, checkpoint saves |
| [0032](0032-character-creation.md) | Character creation from a compiled chargen pack table |
| [0033](0033-m2-module-topology-gateway-deviation.md) | M2 shard module topology via depguard; gateway deviation |
| [0034](0034-pinned-patch-free-godot-fork.md) | Pinned patch-free Godot fork |

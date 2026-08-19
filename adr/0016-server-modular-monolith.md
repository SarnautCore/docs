# ADR 0016 — Server: Go modular monolith shard + separate auth + thin gateway

**Status**: Accepted (2026-08-20)

## Context

The retail server was seven Java services; copying that split onto a fresh game sim
would be self-inflicted pain. Microservices from day one multiply deployment and
debugging cost with no load to justify them.

## Decision

- One Go repo. One **`shard`** binary containing world-sim, combat, quests, chat as
  **internal modules with clean interfaces**.
- A separate **auth/account service** (different security domain; needed by
  launcher and web).
- A thin **gateway** terminating transport/sessions.
- **NATS/JetStream** between services; PostgreSQL for persistence, Valkey for
  sessions/cache, ClickHouse for telemetry.
- Further splits (e.g. per-zone processes under Agones) happen along the existing
  module seams only when M2+ load or deployment needs demand it.

## Consequences

- Module boundaries inside the shard are reviewed as if they were service
  boundaries — no cross-module reach-ins.
- The retail seven-service topology is explicitly not a design input.

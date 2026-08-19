# ADR 0022 — Perforce p4d in Docker, depot on G:

**Status**: Accepted (2026-08-20)

## Context

Art/assets need Perforce (binary-heavy, exclusive checkout culture); code lives in
git. The asset store (ADR 0012) is the depot content. G: has the most free space
and keeps depot churn off the working drive.

## Decision

- **Helix Core p4d runs as a Docker container** on the dev machine (free tier:
  5 users / 20 workspaces), server config versioned in `infra`.
- **Depot storage on `G:`**; the local workspace is `E:\SarnautCore\assets\`.

## Consequences

- Backups of the depot volume are an infra task to schedule.
- If the team ever exceeds the free tier, licensing is a hard-stop decision
  (costs money, ADR 0025).

# ADR 0025 — Continuous operation with hard stops

**Status**: Accepted (2026-08-20)

## Context

The project is orchestrated continuously by agents rather than milestone-gated.
Some actions are irreversible, outward-facing, or legally sensitive and must remain
human decisions.

## Decision

Agents proceed autonomously on code, repos in the org, local infrastructure, and
task tracking — **except** the following, which always stop for the owner's
explicit decision:

1. Publishing MY.GAMES-derived content anywhere beyond the private repos/depot.
2. Anything that costs money.
3. Destructive or irreversible operations on the `E:\allods\*` reference trees
   (treated read-only, always).
4. Changing licenses or the legal posture.
5. Deleting repositories or depots.
6. **Production deployment** (nothing public-facing until explicitly decided).

## Consequences

- These stops are restated in agent instructions and reviewed when the operating
  mode changes.

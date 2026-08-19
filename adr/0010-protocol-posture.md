# ADR 0010 — Protocol posture: clean protocol now, retail-compat door kept open

**Status**: Accepted (2026-08-20)

## Context

Extensive prior research reverse-engineered the retail 1.1 wire protocol: a
366-command coverage matrix, retail captures, a generated message catalog, a
self-verifying codec library, and a Wireshark dissector. Building the new Go↔Godot
stack on the retail protocol would be an enormous ongoing burden (legacy framing,
RC4/RSA handshake, replication layer); ignoring the research wastes the project's
strongest potential correctness oracle.

## Decision

- The Go server and Godot client speak a **clean protobuf protocol** (ADR 0017).
- The retail protocol research (catalog, captures, coverage matrix) serves as a
  **behavioral specification and verification oracle**: the retail client/server
  stack is a reference environment we test observations against.
- The Go server's session/replication layer is designed so a **retail-1.1-protocol
  front-end could be added later without surgery** — retail compatibility is a
  possible future milestone, not a current target.

## Consequences

- Gateway/session abstractions must not leak protobuf specifics into the world sim.
- Protocol research artifacts get distilled into `docs/specs/` as needed.

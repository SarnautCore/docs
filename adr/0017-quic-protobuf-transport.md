# ADR 0017 — QUIC transport with protobuf wire format, behind a seam

**Status**: Accepted (2026-08-20)

## Context

Options were QUIC vs custom reliable-UDP. QUIC gives encryption, connection
migration, multiplexed streams, and datagrams for free; quic-go is mature on the
server side and `System.Net.Quic` ships in modern .NET for the Godot C# client.

## Decision

- **QUIC** (quic-go on the server; `System.Net.Quic` in the client).
- **Protobuf** messages end-to-end (also used for internal gRPC).
- Streams carry reliable-ordered channels (chat, inventory, quests); **datagrams**
  carry movement/state where staleness beats retransmission.
- All of it sits behind a small **transport interface** so raw UDP could replace
  QUIC if profiling ever demands it.

## Consequences

- The transport seam is a hard boundary: game code sees channels/messages, never
  QUIC types.
- Certificate handling for local dev (self-signed) is an infra concern from M1.

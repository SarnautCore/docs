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

## Amendments

- **2026-08-20 — datagrams are a Go-side capability, not a shipped client path.**
  `System.Net.Quic` in .NET 10 exposes no datagram API: `QuicConnection` has no
  send or receive datagram surface, so MsQuic's datagram support is unreachable
  from managed code. `client/src/SarnautCore.Network/GameSession.cs` therefore uses
  the **ordered-stream fallback** for movement and snapshots and reports it in
  `GameSession.TransportMode`. The Go `session.Client` still negotiates datagrams
  when `transport.Connection.SupportsUnreliable()` is true, so the datagram path is
  exercised by Go integration tests and by the smoke client only. The channel split
  above stays the design and stays enforced at the message level
  ([ADR 0026](0026-wire-message-envelope.md)); only the carrier differs on the
  shipped client. Revisit when .NET exposes datagrams or when the client moves to a
  native QUIC binding.

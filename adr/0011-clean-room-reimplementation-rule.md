# ADR 0011 — Clean-room reimplementation rule

**Status**: Accepted (2026-08-20)

## Context

The retail 1.1 server is compiled Java (effectively unobfuscated, cleanly
decompilable). A prior partial C# reimplementation of the seven retail services
also exists locally. Studying these is how mechanics fidelity is achieved — but
literal translation of decompiled code would make our code a derivative work.

## Decision

- Decompiled retail code is used **only to extract facts**: formulas, constants,
  state machines, packet and data semantics.
- Facts are written into **specification documents** (`docs/specs/`), and the Go
  implementation is written **from the specs**. No literal translation of decompiled
  method bodies; no carried-over retail identifiers.
- The prior C# reimplementation (`server_bin_csharp`, `Sarnaut.*`) is **ignored
  entirely** — not mined, not referenced.

## Consequences

- Spec writing is a first-class engineering activity; "read the Java, write the
  spec, implement from the spec" is the standard mechanics workflow.
- Specs contain facts and semantics, never MY.GAMES content (strings, data tables).

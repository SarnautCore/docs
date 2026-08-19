# ADR 0020 — Server scripting: data-driven core + gopher-lua for encounters

**Status**: Accepted (2026-08-20)

## Context

Most mechanics are expressible as data (ADR 0006). Boss encounters, scripted
events, and quest edge-cases need choreography data can't express. In a Go server,
LuaJIT means CGO pain; gopher-lua is pure Go and Lua 5.1-compatible — the same
dialect as the client UI scripting.

## Decision

- The mechanics core is **data-driven**; scripting is the exception, not the rule.
- **Embedded Lua 5.1 via gopher-lua** for boss encounters, scripted events, and
  quest edge-cases, with a small, explicit API surface into the sim.

## Consequences

- One scripting dialect across client UI and server encounters.
- Script API is versioned and documented like any other contract; scripts are data
  (live in the `data` repo, era/pack tagged).

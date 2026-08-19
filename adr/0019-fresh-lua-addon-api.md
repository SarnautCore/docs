# ADR 0019 — Fresh AO-inspired Lua addon API

**Status**: Accepted (2026-08-20)

## Context

The original client exposed a Lua addon API (era addons exist: AOConsole,
PowerAuras, ZTarget…). Faithfully recreating that undocumented API is a large
reverse-engineering project of its own.

## Decision

- Build a **fresh, AO-inspired addon API**: same concepts — widgets, events, slash
  commands — with a clean modern design, scripted in **Lua 5.1-compatible**
  (LuaJIT) as planned.
- Original-AO API compatibility is a **possible later shim**, scoped only after the
  fresh API exists.

## Consequences

- The UI layer documents its addon surface from the first widget.
- Era addons are design references (what addon authors needed), not test targets.

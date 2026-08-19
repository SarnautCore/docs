# ADR 0012 — Content-addressed asset store

**Status**: Accepted (2026-08-20)

## Context

Assets are reused across four client/server versions with heavy overlap (~half of
the 14.0↔17.0 archives are bit-identical). Duplicates must not be stored twice, and
re-conversion must be incremental as converters improve.

## Decision

A **content-addressed store** managed by `tools`:

- Every source payload is hashed (**BLAKE3**) and stored once as `blobs/<hash>`.
- **Per-era manifests** map logical paths (`classic/creatures/...`,
  `modern/...`) → blob hash + provenance (source tree, version, relative path).
- **Converted outputs** are stored the same way, keyed by
  *(source hash, converter, converter version, settings)* so improved converters
  invalidate exactly what they change.
- The store — not the raw clients — is what the Perforce depot versions (ADR 0022).

## Consequences

- Dedup across versions is a structural property, not a chore.
- Any pipeline step can answer "where did this byte come from" (provenance) and
  "is this output stale" (key mismatch).

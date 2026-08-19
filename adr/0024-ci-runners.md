# ADR 0024 — CI: GitHub-hosted runners now, self-hosted later for asset/GPU jobs

**Status**: Accepted (2026-08-20)

## Context

GitHub-hosted runners are free for public repos and need zero setup. Bulk asset
conversion and upscaling need the local asset store and GPU, which hosted runners
don't have.

## Decision

- All code repos run CI on **GitHub-hosted runners** from day one.
- A **self-hosted runner on the dev machine** is added later, exclusively for jobs
  that need the asset store or GPU (bulk conversion, upscaling), never for
  untrusted PR code.

## Consequences

- Workflows are written so asset-dependent jobs are cleanly separable from
  code-only jobs.

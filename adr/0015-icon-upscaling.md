# ADR 0015 — Icon upscaling: Real-ESRGAN 4× stored as a variant

**Status**: Accepted (2026-08-20)

## Context

Classic-era icons are low-resolution; modern UI wants crisper art. Upscaling models
mangle some hand-painted icon styles, so results need human curation.

## Decision

- Batch upscale extracted icons with **Real-ESRGAN 4×**, classic 1.1 icons first,
  then other low-res sets, run locally on the GPU.
- Results are stored in the asset store as an **`upscaled` variant next to the
  original**; the original always remains canonical.
- A human-curation checklist gates upscaled icons into actual UI use.
- If Real-ESRGAN disappoints on the painterly style, alternative models (chaiNNer
  model zoo) are trialed; the variant mechanism is model-agnostic.

## Consequences

- The asset store's variant mechanism (ADR 0012) must support named derived
  variants with their own provenance (model, version, settings).

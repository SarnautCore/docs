# ADR 0034 — Pinned patch-free Godot fork

**Status**: Accepted (2026-08-20)

## Context

SarnautCore needs to rebuild the exact engine used by the client without depending
on an upstream binary archive remaining available. A project-owned fork gives us a
stable source location and a place to publish reproducible Windows .NET builds. It
also keeps a lane open if the project later adopts double precision or finds an
engine change that cannot live outside Godot.

That lane creates an easy failure mode: a fork can quietly collect patches until
upstream upgrades become merges instead of routine version changes. Owning the
repository must not change the extension boundary. [ADR 0008](0008-latest-stable-toolchain-policy.md)'s
stable toolchain policy stays in force, and [ADR 0018](0018-client-csharp-first.md)'s
C#-first rule with GDExtension for proven native hot paths stays in force.
[ADR 0012](0012-content-addressed-asset-store.md) covers the content-addressed
asset store and is unaffected by this decision.

## Decision

- Fork `godotengine/godot` to
  [`SarnautCore/godot`](https://github.com/SarnautCore/godot). Keep `master` as the
  upstream-tracking branch, but make `sarnaut-4.7.2` the default branch.
- Pin the engine source on that branch to the upstream `4.7.2-stable` tag. The only
  fork-owned files above the tag are `SARNAUT.md` and the on-demand release
  workflow. There are **no engine patches**.
- Do not add an engine patch without a dedicated accepted ADR that explains why C#
  or GDExtension cannot solve the problem and accepts the upgrade cost. A future
  double-precision build is likewise a separate decision and release lane, not an
  implicit change to this pin.
- Build binaries only through the fork's manually dispatched workflow. Publish the
  Windows .NET editor, export templates and NuGet packages in fork Releases tagged
  `sarnaut-4.7.2-build<N>`. Never commit engine binaries or NuGet packages to Git.
- Upgrade by fast-forwarding the engine source base to a new upstream stable tag,
  replaying only the two fork-maintenance files, renaming the pin branch for the new
  version and making it the default. The diff from the new stable tag must contain
  no engine files.
- Preserve Godot's MIT `LICENSE`, copyright notices and attribution unchanged.

The fork does not replace the normal extension route. If SarnautCore needs native
engine-facing behavior, GDExtension remains the first route considered before an
engine patch.

## Consequences

- A branch SHA and Release tag identify both the engine source and the binaries
  built from it. SarnautCore retains source-availability insurance if upstream
  release assets move or disappear.
- The fork has almost no long-lived divergence, so stable upgrades remain an
  upstream history update plus two maintenance files.
- Full Windows .NET builds are expensive and manual. Pushes do not start them.
- Any future engine patch becomes visible project architecture with a named owner,
  rationale and accepted rebase burden. The default remains patch-free.

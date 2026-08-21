# ADR 0040 — Materialized-native content architecture

**Status**: Accepted (2026-08-21)

**Supersedes**: [ADR 0001](0001-clean-room-byo-client-posture.md),
[ADR 0013](0013-godot-native-canonical-asset-format.md),
[ADR 0014](0014-converter-gap-priorities.md)

**Amends**: [ADR 0002](0002-classic-11-content-spine.md),
[ADR 0004](0004-per-domain-repo-topology.md),
[ADR 0012](0012-content-addressed-asset-store.md),
[ADR 0015](0015-icon-upscaling.md),
[ADR 0021](0021-importer-first-data-extraction.md),
[ADR 0022](0022-perforce-docker-depot-g.md),
[ADR 0035](0035-m3-league-tutorial.md),
[ADR 0037](0037-public-private-content-split.md)

## Context

ADR 0001 set a bring-your-own-client posture: ship code only, have the user's own
installation supply the assets, and let a launcher-driven converter build packs on
the player's machine. ADR 0013 then made the converter's Godot-native output the
canonical game-ready form, and ADR 0014 scheduled converter gap work as product
work. Together these put an asset-conversion layer inside the shipped client.

Two years of that posture in practice, and M2's visual-completeness work in
particular, exposed what the arrangement costs.

The client carries a permanent adapter layer whose only job is to understand
formats that belong to someone else. Every rendering feature has to be expressed
twice: once in engine terms, once in terms of what the adapter can produce from a
source file. Engine facilities that expect ordinary Godot resources — the importer,
lightmap baking, LOD generation, mesh compression, the scene format's own tooling —
are unavailable to content that materializes at runtime through custom loaders.
Load time and memory are spent per session on work whose answer never changes.
Debugging a visual defect means deciding whether the fault is in the source data,
the adapter, or the renderer.

The posture also failed on its own terms. The player still needs an Allods
installation, still needs the bake to succeed on unpredictable hardware, and still
gets output that nobody reviewed. The legal benefit the posture was bought for —
publishing zero MY.GAMES bytes — does not actually require the conversion to happen
on the player's machine. It requires that derived bytes stay out of public
distribution, which is a publishing rule, not a runtime architecture.

Meanwhile the server side already demonstrated the alternative. Its extractor →
YAML → compiled packs pipeline runs offline, produces artifacts the project owns
and versions, and leaves the runtime with a reader for its own pack format and no
knowledge of XDB at all (ADR 0021, ADR 0006, ADR 0029). The server has no converter
in it. Nothing about visual assets makes them a special case.

A parallel question came to a head during the same design session: which source
version is authoritative for what. ADR 0002 made classic 1.1 the content spine and
noted that 17.0 assets are stored on equal footing, but never said what happens
when both versions can render the same thing. M2 shipped under an implicit
"retail-pixel-identity" bar that 1.1's art cannot meet and 17.0's art cannot
satisfy without changing the world.

## Decision

### 1. Content is materialized offline, never converted at runtime

A private bake pipeline transforms Allods 1.1 and 17.0 source data into the
project's own native content: plain Godot scenes, meshes, and textures, plus
compiled packs. That output is committed and shipped as product content.

The runtime contains no conversion code and no awareness of any original format.
It loads Godot scenes and reads compiled packs, which is what the engine and the
pack reader already do. There is no converter component in the product, no
launcher bake step, and no code path that reads an Allods file.

"Plain" is load-bearing. Baked scenes use stock Godot resource types with no custom
loader, so the whole engine toolchain applies to them: importer settings, lightmap
baking, LOD generation, mesh compression, scene inheritance, and the editor itself.
A baked mesh is indistinguishable from one an artist authored.

The bake tooling — `ao-godot-converter` and its supporting scripts — is internal
machinery. It is not a product component, not shipped, and not on any release path.

### 2. Source authority is split, and the split is explicit

**1.1 is authority for what exists.** Mobs, quests, object positions, names, and
behavior come from 1.1 and nothing overrides them. When 17.0 disagrees about what
is in the world, 1.1 wins.

**The entire out-of-game flow and UI is 1:1 to the 1.1 client.** Login, shard
select, character select, character create, in-game UI layouts, and HUD element
positions match 1.1. The art is 1.1's, upscaled. 17.0 contributes nothing to this
surface.

**17.0 supplies exactly this list and nothing more:**

- **Character models.** Model, rig, and animation set are taken as a single unit.
  Splitting them — a 17.0 mesh on a 1.1 rig, or 1.1 animation on a 17.0 skeleton —
  is not permitted. 17.0 units are auto-matched to 1.1 identities, with a curated
  override table for the cases the matcher gets wrong. An identity with no
  acceptable 17.0 match falls back to 1.1 art, upscaled. The player
  character-creation surface is reduced to 1.1's options regardless of what the
  17.0 units can express.
- **Grass.**
- **Lighting.** The 17.0 lighting model is authoritative outdoors. 1.1's baked
  interior ambience is retained until 17.0-equivalent interior data is
  materialized; this is a temporary asymmetry with a known exit.
- **Shadows.**
- **Weather.**
- **Sky.**

Where a zone exists in 17.0, its environmental dressing is authoritative on the 1.1
layout. The layout does not move to accommodate the dressing.

**Upscaling is a standing bake-time step.** Real-ESRGAN 4x runs over 1.1-sourced
art as part of every bake, not as a one-off pilot and not as a stored variant
alongside an original.

### 3. Publishing: public code, private derived bytes

Engine and server code remain public. Every derived byte is private. ADR 0037's
provenance test decides classification and is unchanged; this decision only adds
where the derived bytes live.

- Baked binary content lives in a dedicated Perforce stream, `//content/main`. It
  is separate from `//assets/main`, which holds the input archive. The distinction
  is direction of travel: `//assets/main` is what the bake reads, `//content/main`
  is what the bake writes and the product ships.
- Gameplay data stays in the private `data` repository.
- `ao-godot-converter` becomes a private repository. It was public under ADR 0001's
  "all code is public" rule; that rule was tied to the converter being a product
  component, and it no longer is.

### 4. Curation survives rebakes

Bake output plus a curated overlay, with the same rule the data pipeline has run
under since ADR 0021: a rebake regenerates the base layer and never destroys hand
fixes. Hand-fixed geometry, corrected placements, and adjusted materials are
overlay entries, not edits to baked files.

### 5. Migration is incremental behind the visual gate

The client's 16-probe visual gate is the migration's control. Each class of content
moves independently, and the gate must stay at 16/16 across every step.

Order:

1. **Terrain.**
2. **Statics.**
3. **Characters.**
4. **UI.**

Each conversion-layer class is deleted when its consumer switches to native
content. Deletion is part of the migration step, not a follow-up: a step that
leaves its adapter class alive has not landed.

The UI bake starts in parallel with terrain because it is the largest surface, but
it lands last. It carries the 1:1 bar from section 2, which makes it the step most
likely to need iteration.

## Consequences

### The runtime adapter layer is scheduled for removal

The `Converted*` / `Allods*` adapter families in the client are now dead code with
a schedule. The coupling map at `E:\SarnautCore\.cache\reconstruction\coupling-map.md`
is the working inventory for that removal and is maintained as the migration
proceeds; it is not reproduced here.

Its relevant numbers: the client carries 423 conversion-era references across 45
files, of which 26 files hold substantive coupling — 15 marked for deletion at
roughly 2,800 lines, 8 for rewrite, with 42 further files needing only a repoint or
a rename. The `Converted*` / `Allods*` types themselves are 10 declared types in 10
files totalling 1,812 lines. The `SarnautCore.Network` and `SarnautCore.Gameplay`
assemblies (8 and 14 files) have no coupling at all, and all 30 xUnit tests are
independent of the conversion path, so the removal does not reach the netcode, the
gameplay sim, or the unit-test suite.

Two of the map's structural findings shape section 5's ordering. First,
`ConvertedSceneLoader` is the deepest coupling: its two load entry points are what
every other adapter class calls, so it cannot go until its callers do. Second,
`ZoneLoader.LoadZone` hard-fails when map content is absent and its flat-terrain
fallback derives extents from placement files, so the zone path cannot be deleted
before its replacement exists. That is why terrain is first and why every step is
materialize-then-delete rather than the reverse. The UI end is the opposite case:
the HUD chrome classes are consumed only by `GameplayHudControl`, and every widget
already has a code-built fallback, so the UI adapters can go the moment the baked
theme is ready.

Of the 16 gate probes, 11 test engine capability and survive with new fixtures, 3
are already content-agnostic, and 2 test conversion fidelity and retire with the
layer they cover. The gate therefore stays meaningful across the migration rather
than needing to be rebuilt at the end.

Because the map already groups the adapter classes by the content class that
consumes them, each migration step in section 5 has a known deletion list before it
starts. A step's deletion list going empty is the mechanical signal that the step is
complete.

### The quality bar changes

Retail-pixel-identity is replaced by two bars that can both be met:

- **Structural 1:1 to 1.1.** What exists, where it is, what it is called, how the
  UI is laid out. This is testable and is what the probes check.
- **Best-available visual quality.** 17.0 dressing where 17.0 has it, upscaled 1.1
  art elsewhere. This is a judgment call and is reviewed, not asserted by a test.

A screenshot that differs from retail 1.1 is no longer a defect by itself. A
screenshot that puts an object in the wrong place still is.

### Milestones

M2 stands as achieved. Its exit criteria were about the vertical slice working
end-to-end, and the visual gate closed at 16/16 under the old bar; changing the bar
does not retroactively reopen it.

M3 absorbs the materialization work alongside its League tutorial scope (ADR 0035).
The four migration steps are M3 issues.

### A dangling reference surfaced

Inventorying the client turned up a test, `OriginAppliedManifestProbe`, that cites
"ADR 0038 gate item 5". ADR 0038 is reserved in the index and referenced by ADR
0035, but was never written. That gap is unrelated to this decision and is recorded
here only so it is not lost again; closing it is separate work.

### Costs accepted

- The project now carries baked binary content in version control, with the storage
  and history growth that implies. This is what `//content/main` is sized for.
- A source-data or bake-tool improvement requires a rebake and a content commit,
  where previously it would have reached players through a converter update. Bake
  turnaround becomes a workflow concern.
- Content and code can drift out of sync across commits in a way that a runtime
  converter made impossible. Pack and scene versioning has to catch it.
- The bake pipeline is now single-machine infrastructure that only maintainers can
  run. Nobody outside the project can reproduce the content from source, which is a
  real loss of the transparency ADR 0001 was reaching for. The provenance records
  from ADR 0012 and ADR 0037 are the partial substitute.

# ADR 0038 — World geometry frame and distance units

**Status**: Accepted (2026-08-20)

## Context

SarnautCore must give the server, converter, and Godot client one unambiguous
meaning for a world position and a distance. The 1.1 content exposes two
coordinate frames:

- quest and other zone-level locations are already global;
- placements in `PatchObjects` are local to a map tile.

Treating both as global clusters patch objects near the origin. Treating both as
local adds an origin twice to quest locations. The source names also invite a
wrong axis interpretation: a tile filename is written `R_C`, while the
authoritative evidence shows that its first component contributes source X and
its second component contributes source Y.

Distance constants have a separate ambiguity. Combat currently names values such
as `ABILITY_RANGE_M = 10`, but the `_M` suffix was a convention, not a measured
conversion from content units. A single measured scale is required for every
ability range, spell range, aggro radius, leash radius, tolerance, movement
distance, and geometry position.

Terrain extraction adds a source-version constraint. The served 1.1 client tree
contains `Packs`, `Editor`, `Mods`, and `Types`; `Maps/Inst_LeagueStart` is a
member of `Packs/Inst_LeagueStart.Client.pak`. The converter has no PAK reader.
Later client trees contain unpacked maps but reauthor geometry, so they cannot
calibrate or stand in for 1.1.

The completed terrain research at `.cache/m3/terrain-two-sided-research.md` is
accepted input to this decision. It selects a canonical two-sided
`terrain-regions.sptbl` pack table, with a derived walkable-ground query layer
used by both the Go server and Godot client. This ADR fixes the frame and scale
of that representation; it does not reopen the representation choice.

## Evidence

### Tile wrap and tile pitch

The following PowerShell command reproduces the decisive wrap in the read-only
1.1 server dump:

```powershell
$ref = 'E:\allods\servers-clean\1.1.02.0\game\data\Maps\Inst_LeagueStart\000_020'
'1_2', '1_3' | ForEach-Object {
    $xml = [xml](Get-Content -Raw (Join-Path $ref "${_}_ServerObjects.xdb"))
    $ys = @($xml.SelectNodes('//center') | ForEach-Object { [double]$_.GetAttribute('y') })
    [pscustomobject]@{
        Tile = $_
        Count = $ys.Count
        MinY = '{0:N2}' -f (($ys | Measure-Object -Minimum).Minimum)
        MaxY = '{0:N2}' -f (($ys | Measure-Object -Maximum).Maximum)
    }
} | Format-Table
```

It prints:

```text
Tile Count   MinY   MaxY
---- -----   ----   ----
1_2     89 134.01 255.47
1_3      1   0.82   0.82
```

The second filename component increments exactly where local Y wraps from
approximately 255 to approximately 0. The tile pitch is therefore 256 content
units, and the second component is the source-Y tile index. By the same
ordering, the first component is the source-X tile index. Sector directories use
the same component order: `000_020` is source sector X 0, Y 20, and `020_020` is
source sector X 20, Y 20.

One placement in `Maps/Inst_LeagueStart/000_020/1_2_ServerObjects.xdb` has local
center `(61.1128, 241.054, 56.4219)`. Its global horizontal position is:

```text
X = (0 + 1) * 256 + 61.1128  = 317.1128
Y = (20 + 2) * 256 + 241.054 = 5873.054
```

The quest location `(300.101013, 5814.417969, 0)` lies in that same global tile.
The agreement independently connects patch-local placements to already-global
quest coordinates. It also fixes the sector component order. Reversing `000_020`
to mean sector X 20, Y 0 would reconstruct the placement near `(5437, 753)`,
nowhere near the quest location.

### Distance calibration

The measured calibration pair is a repeated wall run in:

- `Maps/Inst_LeagueStart/000_020/1_2_MapRegion.xdb`;
- `World/Dungeons/InstLeague1/InstLeague1_AI_2X2.(AICollisionMesh).xdb`,
  together with its geometry payload.

This command reproduces the measurement from those source documents:

```powershell
[xml]$region = Get-Content -Raw 'E:\allods\servers-clean\1.1.02.0\game\data\Maps\Inst_LeagueStart\000_020\1_2_MapRegion.xdb'
$walls = @($region.SelectNodes('//Item[StaticObjectTemplate[contains(@href,"InstLeague1_AI_2X2")]]') |
    Where-Object { [math]::Abs([double]$_.Scale.Ratio - 2.56681) -lt 0.00001 })
$steps = for ($i = 1; $i -lt $walls.Count; $i++) {
    $dx = [double]$walls[$i].Position.X - [double]$walls[$i - 1].Position.X
    $dy = [double]$walls[$i].Position.Y - [double]$walls[$i - 1].Position.Y
    [math]::Sqrt($dx * $dx + $dy * $dy)
}
[xml]$mesh = Get-Content -Raw 'E:\allods\servers-clean\1.1.02.0\game\data\World\Dungeons\InstLeague1\InstLeague1_AI_2X2.(AICollisionMesh).xdb'
[pscustomobject]@{
    Count = $walls.Count
    MeanStep = ($steps | Measure-Object -Average).Average
    ScaledWidth = 2 * [double]$mesh.AICollisionMesh.aabb.extents.y * 2.56681
}
```

Nine consecutive wall placements use scale `2.56681`. Their eight horizontal
anchor spacings are `4.75`, `5.03`, `5.00`, `5.06`, `5.03`, `5.00`, `5.08`, and
`4.86` content units, with mean `4.97696`. The collision mesh has horizontal
half-extent `1.00815`, so its scaled full width is:

```text
2 * 1.00815 * 2.56681 = 5.175459
```

The approximately four-percent overlap is expected for a curved barrier run. The
pair proves that placement coordinates and model coordinates use the same linear
unit. `ao-godot-converter/src/geometry.rs` preserves model coordinates, and
`ao-godot-converter/src/godot.rs` applies no linear scale when it emits either
representation. SarnautCore's Godot convention is one world unit per metre. A
0.1 conversion would require scaling both representations and would shrink this
five-unit wall module to about half a metre, which contradicts its source use as
a continuous collision barrier. The measured constant is therefore:

```text
METRES_PER_CONTENT_UNIT = 1.0
```

This is the only linear conversion constant. It applies equally to positions,
terrain samples, collision geometry, movement, ranges, aggro, and leash values.

## Decision

### Canonical server frame

The canonical persisted and network frame is the 1.1 source global frame:

- X and Y are horizontal axes;
- Z is vertical and positive upward;
- positions use metres;
- rotations remain source rotations until converted at a consumer boundary.

For sector directory `SX_SY`, tile filename `R_C`, tile-local center
`(lx, ly, lz)`, and pitch `P = 256` metres:

```text
origin_x = (SX + R) * P
origin_y = (SY + C) * P

server_x = origin_x + lx
server_y = origin_y + ly
server_z = lz
```

`SX`, `SY`, `R`, and `C` are integer indices, not distances. Despite the `R_C`
spelling, `R` contributes X and `C` contributes Y. Already-global records,
including quest locations, bypass this origin calculation.

The Go server stores, simulates, persists, and transmits this canonical frame.
It does not store Godot coordinates.

### Godot boundary mapping

Godot uses Y-up. At the client boundary, convert a canonical server position as
follows:

```text
godot_x = server_x
godot_y = server_z
godot_z = -server_y
```

The inverse mapping is:

```text
server_x = godot_x
server_y = -godot_z
server_z = godot_y
```

This matches the existing object and terrain mapping in
`ao-godot-converter/src/godot.rs`. Conversion happens once at import or protocol
boundaries. Gameplay code must not scatter sign changes or axis swaps.

The protocol does not perform this conversion. Content-pack vectors pass through
`server/internal/pack/pack.go`, the simulation uses X/Y as its horizontal plane,
and `server/internal/session/mapping.go` copies X/Y/Z unchanged in both wire
directions. The wire therefore carries the canonical server frame.

#### Known client and cache defects

The current online client violates this decision. `ZoneNetworkLoop.ToGodot`,
local reconciliation, target projection, and `NetworkEntityVisual.Apply` map a
server vector to `(X, Z, Y)`, while outbound movement sends Godot Z directly as
server Y. This mirrors online entities and movement across the canonical X
axis. `ZoneLoader.ConvertPosition` and the converter use the correct axis
mapping `(X, Z, -Y)`, but that alone does not make their tile-local coordinates
global.

The current converted cache has a second defect. For tile
`000_020/1_2`, `1_2_ServerObjects.xdb.placements.json` still contains
`(61.1128, 241.054, 56.4219)`, and the terrain OBJ has local Godot bounds X
`0..168`, Z `-256..-120`. The generated terrain and map-region scenes have no
root transform. `ZoneLoader` also adds every terrain mesh and converted
placement below untransformed type-wide roots. It neither derives nor applies
the tile's Godot origin `(256, 0, -5632)`. Correct sign conversion therefore
leaves converted terrain and placements thousands of metres from canonical
server entities. No parent-node transform repairs either defect.

#### Required client and cache repair

Before online terrain or visual acceptance, the client must replace these
ad-hoc mappings with one shared boundary module:

```text
server position (x, y, z) -> Godot position (x, z, -y)
Godot ground direction (x, 0, z) -> server direction (x, -z, 0)
```

The same module must serve admission spawn placement, local reconciliation,
remote entity visuals, target projection, and movement intents. Server pack,
simulation, persistence, and protocol mappings remain unchanged. A server
heading measured from canonical +X toward +Y keeps the same numeric yaw in
Godot, matching the converter's current yaw mapping. A known authored placement
must anchor that convention in the regression suite so model-forward offsets do
not become an undocumented second correction.

The authoritative extracted pack applies the sector/tile origin before it
serializes positions or patch coordinates. Those rows are canonical-global and
must never receive another tile origin.

Converted Godot caches may retain tile-local mesh vertices and placement arrays.
If they do, the converter must emit one machine-readable manifest for each tile
with:

- frame identifier `allods-tile-local-v1`;
- sector indices `SX` and `SY`;
- tile indices `R` and `C`;
- canonical `origin_x` and `origin_y` computed by the formula in this ADR.

`ZoneLoader` must reject a local cache without that manifest. It creates one
tile root at `(origin_x, 0, -origin_y)` and parents that tile's terrain,
collision, map-region statics, and authored server placements below it. Online
entities and canonical-global cache rows remain below the global zone root. The
loader must merge transformed terrain bounds, not each mesh's local AABB. It
must also reject a manifest that asks it to add an origin to data marked
canonical-global. Filename parsing, axis reflection, and origin arithmetic must
not be reimplemented separately for terrain, statics, and NPCs.

### Extractor correctness gate

Before terrain samples are accepted, `sarnaut-extract` must correctly classify
every source position as tile-local or already-global and apply the origin
formula exactly once. Its regression suite must include:

1. the `1_2` to `1_3` local-Y wrap shown above;
2. the placement `(61.1128, 241.054, 56.4219)` becoming
   `(317.1128, 5873.054, 56.4219)`;
3. the nearby quest location remaining unchanged;
4. a canonical-to-Godot-to-canonical round trip within the serialized numeric
   tolerance, exercised through the shared client boundary module rather than a
   second copy of the formula;
5. an adjacent sector case proving `000_020` to `020_020` changes X, not Y;
6. rejection of an input that would add a tile origin to an already-global
   record.

Terrain output is invalid until these cases pass. A frame identifier and
`METRES_PER_CONTENT_UNIT` value must be included in extracted pack metadata so
consumers can reject incompatible data.

The client regression suite must also plant one asymmetric canonical position,
preferably the placement from item 2. For `000_020/1_2`, it must prove all of the
following:

1. the tile origin is canonical `(256, 5632, 0)` and Godot
   `(256, 0, -5632)`;
2. cached local terrain bounds X `0..168`, Z `-256..-120` become global Godot
   bounds X `256..424`, Z `-5888..-5752`;
3. the local placement becomes `(317.1128, 56.4219, -5873.054)` through the
   tile-root path;
4. admission spawn, local reconciliation, remote rendering, and target
   projection resolve the same canonical point directly through the global
   boundary path;
5. applying a tile origin to a canonical-global row fails instead of silently
   double-shifting it;
6. a Godot ground direction `(2, 0, -3)` emits wire direction `(2, 3, 0)`, and a
   server +Y movement appears toward Godot -Z.

The regression must include adjacent tiles so a shared boundary vertex lands at
the same global position from either tile. The online probe must compare a
network entity against converted terrain or a static landmark. Comparing only
the controller with another value transformed by the same client path is not
sufficient. The existing `ZoneOnlineProbe` expectation using `(X, Z, Y)` must
be corrected. Until this cross-path regression passes, online scene placement
is a release blocker.

### Distance conversion

Every content distance is multiplied by `METRES_PER_CONTENT_UNIT`, which is
exactly `1.0`. The following table makes the combat and spell implications
explicit:

| Source or server field                                     | Content value | Canonical value |
| ---------------------------------------------------------- | ------------: | --------------: |
| `ABILITY_RANGE_M`                                          |            10 |            10 m |
| `RANGE_TOLERANCE_M`                                        |           0.5 |           0.5 m |
| `AGGRO_RADIUS_M`                                           |            12 |            12 m |
| `LEASH_RADIUS_M`                                           |            40 |            40 m |
| `AimedShot/Spell01.xdb` top-level `<range>`                |            40 |            40 m |
| `AimedShot/Spell01.xdb` nested `PredicateRemote` `<range>` |             8 |             8 m |

The four server constants remain curated gameplay defaults, but their unit is
now measured rather than inferred from their names. All other spell `<range>`
fields and future content radii use the same multiplication. No subsystem may
introduce a separate range, aggro, leash, or terrain scale.

### Authoritative 1.1 terrain source

Terrain and placement geometry must come from the 1.1.02.0 source set, not a 2.0
or later unpacked tree. The conversion input manifest must identify version
`1.1.02.0` and pin the digest of
`clients/1.1.02.0/data/Packs/Inst_LeagueStart.Client.pak`. The pinned file's
SHA-256 is `dfd1f0c3b2236e605acc44ba8b8384eaf5904f2246c5253125cdb6d0b72f45da`.
Extraction fails closed when the version or digest is absent or mismatched.

`ao-godot-converter` will gain a read-only PAK member reader and read the
required 1.1 map members directly from `Inst_LeagueStart.Client.pak`. A
developer's manually unpacked directory is not an authoritative input. Pinning a
later unpacked map is rejected because later clients reauthored this geometry.

### Terrain representation and queries

The accepted terrain-research direction is part of this ADR:

- the canonical zone artifact is `terrain-regions.sptbl` as defined by
  [ADR 0029](0029-runtime-pack-format.md);
- each 256-by-256-metre region preserves exact, independent up-facing and
  down-facing sparse patch data;
- each sparse patch spans 8 metres, giving 32 by 32 patch positions per region
  axis;
- region keys and patch coordinates are expressed in the canonical server frame
  and the 1.0 metres-per-content-unit scale;
- both the Go server and Godot client consume the same canonical row data;
- converted `.tscn` files are caches, not authority;
- a derived walkable-ground query layer selects traversable up-facing surfaces
  using prior Z, masks, and ambiguity reporting.

Stacked floors use the named `PRIOR_Z_THEN_TOP_GROUND` fallback. A query first
chooses a valid walkable up-facing surface nearest the actor's previous
canonical Z. If there is no unambiguous sample, it falls back to the top
walkable ground or heightfield and then applies authored blocking and floor
volumes. The query reports that fallback so callers and diagnostics do not
mistake it for exact two-sided terrain. Down-facing samples remain available for
ceilings and underside collision but are never silently substituted for walkable
ground.

Terrain3D is MIT-licensed and remains a useful reference for region residency,
clipmap LOD, and dynamic collision. We will not fork it. Its core model stores
one height per horizontal vertex in a reusable clipmap mesh. It cannot preserve
the source's independent up/down sparse meshes, topology, material links, and
side-specific lightmaps. A faithful fork would replace the storage, mesher,
shader, and collision assumptions while inheriting the rest of the upstream
codebase.

A Godot engine patch is also rejected under
[ADR 0034](0034-pinned-patch-free-godot-fork.md). Initial client consumption may
use C#; a purpose-built GDExtension is permitted later only when profiling
justifies it and it continues to consume the same pack rows.

## Consequences

- Quest locations, patch placements, terrain, collision, and combat distances
  share one global metre frame.
- The tile and sector formula is testable and eliminates the `R_C` axis
  ambiguity.
- Local Godot caches carry an explicit tile frame and place all content for that
  tile below one translated root. Canonical pack rows remain global.
- Godot axis conversion is isolated to a boundary and is exactly reversible.
- Online placement and movement remain blocked until the client removes its
  positive-Z mirror, applies cache tile origins, and passes the cross-path
  coordinate regression.
- Existing combat defaults retain their numeric values because the measured
  scale is unity.
- Extractor metadata and round-trip tests prevent mixed-frame or mixed-version
  packs from loading.
- Direct PAK reading adds converter work but removes an unverifiable
  manual-unpack step and keeps 1.1 as authority.
- Exact two-sided terrain remains canonical while server and client may build
  purpose-specific walkable query caches.
- Ambiguous stacked-floor queries degrade explicitly through
  `PRIOR_Z_THEN_TOP_GROUND` rather than choosing an arbitrary surface.

## Sources

- `E:\SarnautCore\.cache\m3\terrain-two-sided-research.md`
- `E:\SarnautCore\ao-godot-converter\src\map.rs`
- `E:\SarnautCore\ao-godot-converter\src\geometry.rs`
- `E:\SarnautCore\ao-godot-converter\src\godot.rs`
- `E:\SarnautCore\client\src\zone\ZoneLoader.cs`
- `E:\SarnautCore\client\src\zone\ZoneNetworkLoop.cs`
- `E:\SarnautCore\client\src\zone\NetworkEntityVisual.cs`
- `E:\SarnautCore\client\tests\ZoneOnlineProbe.cs`
- `E:\SarnautCore\assets\converted\classic-1.1\assets\Maps\Inst_LeagueStart\000_020\1_2.terrain.obj`
- `E:\SarnautCore\assets\converted\classic-1.1\assets\Maps\Inst_LeagueStart\000_020\1_2.terrain.tscn`
- `E:\SarnautCore\assets\converted\classic-1.1\assets\Maps\Inst_LeagueStart\000_020\1_2_MapRegion.xdb.scene.tscn`
- `E:\SarnautCore\assets\converted\classic-1.1\assets\Maps\Inst_LeagueStart\000_020\1_2_ServerObjects.xdb.placements.json`
- `E:\SarnautCore\server\internal\pack\pack.go`
- `E:\SarnautCore\server\internal\session\mapping.go`
- `E:\SarnautCore\server\internal\world`
- `E:\allods\servers-clean\1.1.02.0\game\data\Maps\Inst_LeagueStart\000_020\1_2_ServerObjects.xdb`
- `E:\allods\servers-clean\1.1.02.0\game\data\Maps\Inst_LeagueStart\000_020\1_3_ServerObjects.xdb`
- `E:\allods\servers-clean\1.1.02.0\game\data\Maps\Inst_LeagueStart\000_020\1_2_MapRegion.xdb`
- `E:\allods\servers-clean\1.1.02.0\game\data\World\Dungeons\InstLeague1\InstLeague1_AI_2X2.(AICollisionMesh).xdb`
- `E:\allods\servers-clean\1.1.02.0\game\data\Mechanics\Spells\Warrior\AimedShot\Spell01.xdb`
- `E:\allods\clients\1.1.02.0\data\Packs\Inst_LeagueStart.Client.pak`

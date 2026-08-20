# ADR 0029 — Compiled runtime pack format and overlay merge semantics

**Status**: Accepted (2026-08-20)

## Context

[ADR 0006](0006-yaml-source-compiled-runtime-data.md) settled that YAML is the
authored source, that a compiler in `tools` produces a compact runtime form, and that
**runtime never parses YAML** — but it left "JSON or binary packs" open. The shard
still parses YAML at startup: `server/internal/content/loader.go` walks
`<root>/<ruleset>/zones/<zone>/spawns/**` with `gopkg.in/yaml.v3` on every boot.

Three forces close the question now.

**Scale.** The classic item corpus alone is thousands of documents across two dozen
directories under `data/classic/items/`. Re-parsing that per shard boot is the wrong
shape for M2.

**Identity.** [ADR 0027](0027-proto-contract-and-wire-evolution.md) puts a content
digest in the handshake. That requires a compiled artifact whose identity is a pure
function of its content bytes.

**Clean-room hygiene.** The extractor keeps an untyped passthrough map: `pub extra:
BTreeMap<String, Value>` appears on six row types in
`tools/crates/sarnaut-extract/src/model.rs`, filled by `extra_fields` in `xdb.rs`
with every XML attribute the mapper did not claim. Its **keys are verbatim MY.GAMES
type and attribute names** and its values include resource `href` paths. That is
useful while mapping coverage is incomplete, and it must not reach a public artifact
([ADR 0011](0011-clean-room-reimplementation-rule.md)).

## Decision

### Producer and consumer

- **Writer**: a new Rust crate `tools/crates/sarnaut-pack` (workspace member beside
  `sarnaut-assets` and `sarnaut-extract`), binary `sarnaut-pack`, using the existing
  workspace `blake3 = "1.8"` plus `prost = "0.14"` and `prost-build = "0.14"`.
- **Reader**: a new Go package `server/internal/pack`, using
  `github.com/zeebo/blake3 v0.2.4` for digest verification and the repo's existing
  `google.golang.org/protobuf v1.36.12` for row decoding. It replaces
  `internal/content`'s YAML walk.

Neither side imports the other. They interoperate because this ADR specifies the
bytes and because a CI job builds the golden fixture with the Rust writer and reads
it with the Go reader.

### On-disk layout

A pack is a directory, not an archive:

```
<packs-root>/<ruleset>/<zone>/
  manifest.json
  tables/<table>.sptbl
```

`<ruleset>` is `classic` or `modern`. `<zone>` is a zone slug, or `-` for a
ruleset-global pack such as items. `<table>` matches `[a-z0-9-]+`. A directory rather
than one container file so tables can be memory-mapped and diffed individually, and
so the pack digest covers table bytes rather than archive framing.

### manifest.json

Canonical JSON: UTF-8, LF, two-space indent, trailing newline, keys emitted in the
order below.

| Field | Type | Meaning |
|---|---|---|
| `schema_version` | integer | Format version of the pack itself. Currently `1`. A reader rejects any other value outright. |
| `ruleset` | string | `classic` or `modern`. |
| `zone` | string | Zone slug, or `-` for a ruleset-global pack. |
| `pack_id` | string | 64 lowercase hex characters. BLAKE3-256 over table bytes, computed as below. |
| `builder` | object | `{"name":"sarnaut-pack","version":"<crate semver>"}`. |
| `source` | object | `{"repo":"data","commit":"<40 hex>","overlays":["<layer id>", …]}` in applied order. |
| `keep_extra` | bool | True only when built with `--keep-extra`. |
| `tables` | array | Sorted by `name`. Each entry `{"name":…, "file":"tables/<name>.sptbl", "row_type":"<RowType enum name>", "rows":<u32>, "bytes":<u64>, "blake3":"<64 hex>"}`. |

### How pack_id is computed

Over the tables sorted by `name`, bytewise ascending, feed one BLAKE3-256 hasher:

```
for each table in sorted(tables by name):
    u32le(len(name_utf8)) || name_utf8 || u64le(len(table_bytes)) || table_bytes
pack_id = lowercase_hex(hasher.finalize())          # 32 bytes, 64 hex chars
```

The manifest itself is **not** an input. A `builder.version` bump or a new
`source.commit` that produces identical tables leaves `pack_id` unchanged, which is
the property ADR 0027's handshake comparison needs: the digest answers "is this the
same content", not "was this built by the same run". Length prefixes precede both
name and bytes so no pair of tables can be reassociated into the same byte stream.

### Table encoding (`.sptbl`)

All integers little-endian and unsigned. All offsets absolute from file start.

**Header, exactly 40 bytes:**

| Offset | Size | Field |
|---|---|---|
| 0 | 4 | magic, ASCII `SPK1` |
| 4 | 2 | `format_version`, currently `1` |
| 6 | 2 | `flags`; bit 0 set means the key index is present; all other bits MUST be zero |
| 8 | 4 | `row_type_id`, a value of `sarnaut.content.v1.RowType` |
| 12 | 4 | `row_count` |
| 16 | 4 | `key_index_offset` |
| 20 | 4 | `row_index_offset` |
| 24 | 4 | `row_data_offset` |
| 28 | 4 | `row_data_bytes` |
| 32 | 8 | reserved, MUST be zero |

**Key index** at `key_index_offset` (= 40 in a writer-produced file, so it is
8-byte aligned): `row_count` entries of 12 bytes, `{u64 key_hash; u32 row_ordinal}`,
sorted ascending by `key_hash` then by `row_ordinal`. `key_hash` is the first 8 bytes,
read little-endian, of BLAKE3-256 over the row's canonical id in UTF-8 (for example
`item.junk.clockwork-feather`). Hash collisions are legal: a reader that finds equal
adjacent `key_hash` values decodes each candidate row and compares the id string.
This gives lookup by canonical id without decoding the table.

**Row index** at `row_index_offset`: `row_count + 1` `u32` values, offsets **relative
to `row_data_offset`**, strictly increasing, with `idx[0] == 0` and
`idx[row_count] == row_data_bytes`. Row `i` is `[idx[i], idx[i+1])`. The trailing
entry removes the last-row special case and makes bounds checking O(1).

**Row data** at `row_data_offset`: `row_data_bytes` of concatenated rows, no padding
and no per-row length prefix (the index carries lengths). Each row is a protobuf
message of the type named by `row_type_id`, defined in
**`data-schemas/proto/sarnaut/content/v1/*.proto`**, package `sarnaut.content.v1`.
This is deliberately a **different** proto package from the wire contract
`sarnaut.v1`: content rows carry fields no client ever sees, and coupling them to the
wire evolution rules of ADR 0027 would be wrong. File size is exactly
`row_data_offset + row_data_bytes`.

**Determinism requirements on the writer**: rows sorted by canonical id bytewise
ascending; protobuf fields emitted in ascending field number; no unknown fields
carried through; no timestamps, paths, or hostnames anywhere in table bytes.

**Validation the reader performs before the shard accepts a pack**: magic;
`format_version == 1`; unknown `flags` bits zero; reserved zero; every offset inside
the file and non-overlapping; row index strictly increasing with the required first
and last values; key index sorted; `row_type_id` matches the `row_type` recorded in
the manifest for that table; each table's BLAKE3 matches the manifest; recomputed
`pack_id` matches `manifest.pack_id`. Any failure aborts shard startup. There is no
partial load and no repair path.

### The `extra:` passthrough is stripped by default

`sarnaut-pack` drops every `extra` map while compiling. `--keep-extra` retains it as
`map<string, string>` (values JSON-encoded) on the row message and records
`"keep_extra": true` in the manifest. Three guards follow:

1. The shard refuses to load a pack with `keep_extra: true` unless
   `content.allow_extra: true` is set in config; the default is false. When it does
   load one, it logs a warning naming the pack path and digest.
2. A CI check `server/scripts/check-fixture-pack.ps1` fails if any committed
   `manifest.json` in the repo has `keep_extra: true`.
3. `--keep-extra` is for local mapping work against the private data repo only. Its
   output is a private-path artifact by definition.

### Where packs live

- Packs are built **from the private `data` repo** (ADR 0004) and written to a
  private packs root. `data/.gitignore` carries `packs/`: `data` holds YAML source,
  not build output.
- The shard loads packs **only** from `content.packs_path`, which replaces
  `content.root_path` in `server/config.example.yaml`. It has **no default**; an
  unset value is a fatal configuration error. There is no repo-relative fallback and
  no search path, so a misconfigured shard fails loudly instead of quietly loading
  something else.
- Packs are never committed to any public repository, with exactly one exception:
  the golden fixture below, which contains no MY.GAMES-derived data.

### Overlay merge semantics

Layering implements [ADR 0021](0021-importer-first-data-extraction.md)'s
generated-base plus curated-overlay model.

- **Layers**: the base layer is `data/<ruleset>/**`. Overlay layers are
  `data/overlays/<layer-id>/**`, applied in the order listed in
  `data/overlays/layers.yaml` (`layers: [{id: …, description: …}, …]`). That file is
  the sole authority on ordering — never filesystem order, never lexicographic order,
  never per-file precedence hints. `sarnaut-pack --overlay <id>` selects a subset;
  selected layers always keep `layers.yaml` relative order. The applied list is
  recorded in `manifest.source.overlays`.
- **Merge, per document id**: an overlay document is a **patch over the merged result
  so far**, not a replacement. Scalars replace. Mappings merge recursively,
  key by key. **Sequences replace wholesale** — element-wise merging has no stable
  identity to key on, and a half-merged list is worse than an explicit rewrite.
- **Deletion**: a mapping node may carry `_op: replace` to discard the merged value
  beneath it instead of merging into it; a document may carry a top-level
  `_delete: [<dotted.path>, …]` applied after the merge; a document with top-level
  `_op: delete` removes that id from the pack entirely.
- **`curation_note` is required**: every overlay document MUST carry a non-empty
  top-level `curation_note` explaining why the patch exists. A document without one
  is a **compile error**, not a warning. Notes are stripped from row bytes (so they
  never affect `pack_id`) and aggregated into `build-report.json` beside the
  manifest, which is a private-path artifact and is not part of the pack digest.
- **Conflicts**: when two overlay layers touch the same leaf path of the same
  document, `sarnaut-pack` prints both layer ids with the path and fails, unless
  `--allow-overlay-conflicts` is passed, in which case later layer wins and every
  conflict is listed in `build-report.json`.
- **Determinism**: the same base tree, the same `layers.yaml`, and the same builder
  version produce byte-identical tables and therefore an identical `pack_id`.

### Golden fixture

`sarnaut-pack --src ../data-schemas/demo --out server/testdata/packs/demo` compiles
the three hand-authored demo documents in `data-schemas/demo/` — invented content,
zero MY.GAMES data, per that repo's stated posture — into a pack vendored at
`server/testdata/packs/demo/` and committed to the public `server` repo.
`internal/pack` tests load it, exercise every validation rule against deliberately
corrupted copies made in-memory, and assert the recomputed `pack_id` equals a
constant pinned in the test, so a silent format change fails the build. A `tools` CI
job rebuilds the fixture and diffs it against the vendored copy: that diff is the
check that actually proves the Rust writer and the Go reader still agree.

### Alternatives considered and rejected

- **Compiled JSON**, the other option ADR 0006 left open. Rejected: every field is
  re-parsed into fresh allocations at boot, there is no O(1) row addressing, and
  there is no natural place for a content digest. JSON is kept for `manifest.json`
  alone, which is read once and read by humans.
- **SQLite as the pack container.** Rejected: it forces a cgo-or-pure-Go choice into
  the server, and it makes byte-level determinism — and therefore `pack_id` — depend
  on a third-party writer's page layout and vacuum behavior. A read-only row store
  does not need a query engine.
- **A single zip or tar archive per pack.** Rejected: entry ordering, timestamps, and
  compression settings leak into the digest, and it rules out memory-mapping.
- **Reusing the wire protos (`sarnaut.v1`) as row types.** Rejected: it binds content
  schema changes to the network compatibility rules of ADR 0027 and drags
  server-only fields onto the wire schema.

## Consequences

- ADR 0006's "JSON or binary packs" is now settled: binary tables, JSON manifest.
- `internal/content`'s YAML loading is deleted at cutover; there is no dual-mode
  period where the shard can read either form.
- `data-schemas` gains a `proto/` directory. JSON Schema stays authoritative for
  authored YAML; the content protos describe the compiled row shape; a mismatch
  between them is a `sarnaut-pack` compile error.
- Content changes now require a build step before a shard sees them, and the
  handshake digest of ADR 0027 makes a stale build visible at connect time.

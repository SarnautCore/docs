# ADR 0037 — Public authored content, private game-derived content

**Status**: Accepted (2026-08-20)

## Context

ADR 0001 keeps extracted and converted MY.GAMES content out of public repositories.
ADR 0021 then put both generated data and human curation under `data/`, and ADR 0029
made that private repository the only pack input. This hides work that belongs to
SarnautCore itself. The English overlay, balance changes, and the Paper Harbor demo
are authored by this project and should be inspectable and reusable.

File format is a poor boundary. YAML copied from an XDB document is still derived
game data, while a compiled pack built only from an invented quest is still authored
content. Repository placement must follow provenance through transformations.

## Decision

### 1. Provenance decides placement

Every content artifact is classified by this test:

> Would the artifact contain the same creative payload and gameplay values if its
> author had never inspected Allods game data, client assets, localization packs, or
> converted output?

If the answer is no or cannot be established, the artifact is **game-data-derived**
and lives only in the private `data` repository or another private build location.
Changing its format, field names, ordering, compression, resolution, or canonical ID
does not change that classification.

Game-data-derived artifacts include:

- extracted YAML and reference-preserving overlays;
- retail strings and translations made from retail localization;
- converted textures, models, maps, terrain, animation, audio, and their metadata;
- compiled packs, indexes, fixtures, or reports that contain derived rows;
- patches whose values were copied, normalized, measured, or calculated from source
  game content.

An artifact is **authored** when contributors invented its creative payload and
gameplay values using only public schemas, public specifications, and SarnautCore
interfaces. Authored base documents and authored overlays live in the new public
`content` repository. An authored overlay may target a SarnautCore canonical ID, but
it must not reproduce the source row it patches. A patch derived from comparison with
the source row stays private.

Aggregate facts needed by architecture decisions and mechanics specifications may
remain public under ADR 0011. Counts, formulas, state-machine facts, and type names are
not content rows. Bulk tables, strings, coordinates, or other payloads do not become
public merely because a report summarizes them.

Uncertain provenance defaults to private. Moving an artifact from private to public
requires a provenance review; a formatter or compiler cannot perform that promotion.

### 2. Packs consume two explicit roots

`sarnaut-pack` accepts two independent inputs:

- `--derived-root <path>` points at the private `data` checkout and may be absent;
- `--authored-root <path>` points at the public `content` checkout and may be absent.

At least one root is required. Neither flag has a repository-relative default, search
path, or fallback. A real InstLeague1 build supplies both roots. A zero-IP profile such
as Paper Harbor supplies only the authored root and must build in an environment where
the derived root does not exist.

Pack input order is fixed:

1. derived base documents, when a derived root is present;
2. authored base documents;
3. authored overlay layers in the order declared by
   `<authored-root>/overlays/layers.yaml`.

Duplicate base IDs across roots are errors. A deliberate change to a derived row must
be an overlay, which keeps the change visible and preserves ADR 0029's recursive-map,
sequence-replacement, deletion, conflict, and zone-selection rules. The compiler never
copies both roots into a merged staging tree.

The manifest and build report record each participating root as `derived` or
`authored`, its repository commit, and the applied overlay IDs. They record no local
absolute paths. ADR 0029's `pack_id` remains a function of table bytes, so adding a
second source record does not change the digest of byte-identical tables.

Output inherits the strictest input provenance. If any derived root or derived
artifact participates, the pack is private even when later overlays replace or delete
all visible derived rows. A pack is eligible for a public repository only when it was
built without a derived root and passes the provenance scan below.

### 3. Public repositories get a mechanical provenance scan

`scripts/scan-public-artifacts` runs in CI across every public SarnautCore repository.
It checks these named heuristics:

1. **Authored declaration.** Data-like files in public content locations must validate
   against a public schema and declare `provenance.kind: authored`. Missing, unknown,
   or `derived` provenance fails.
2. **Source provenance fields.** Structured files fail on extractor-only fields and
   values such as `source_path`, `source_xpointer`, `extractor_version`,
   `keep_extra: true`, or a pack source whose kind is `derived` or repository is
   `data`.
3. **Retail path and type fingerprints.** Artifact files fail on `#xpointer(`,
   `gameMechanics.`, `.(Type).xdb`, retail root hrefs such as `/World/`, `/Maps/`,
   and `/Mechanics/`, or source resource IDs. Documentation, specifications, and tool
   source are excluded from this text heuristic because ADR 0011 permits them to
   record facts and parser vocabulary.
4. **Forbidden containers and signatures.** Raw XDB, client/server PAK, localization
   pack, terrain-dump, and source-derived converted asset signatures fail regardless
   of filename.
5. **Compiled-pack provenance.** The scan parses every public pack manifest. It
   rejects `keep_extra`, any derived source, a missing authored source declaration,
   and table files not covered by that manifest.
6. **Repository inventory.** Binary and large-file paths must match the repository's
   checked-in authored-artifact inventory. Each inventory entry binds an exact path,
   digest, generator, and authored source profile.

An exception binds an exact path and digest and carries a reason in the repository's
scan configuration. Directory globs and extension-wide exceptions are forbidden. A
content change invalidates the digest and forces the exception back through review.
The scan is a hard CI failure and includes a planted derived fixture in its own tests.

These checks catch likely leaks; they do not prove originality. The provenance
declaration remains a contributor assertion reviewed under ADR 0025's legal-posture
hard stop.

### 4. Repository and prior-ADR effects

- ADR 0001 is unchanged: public releases still contain zero MY.GAMES assets or game
  data. This ADR adds a CI check and makes clear that compiled or transformed data
  remains private.
- ADR 0004 is amended from eleven repositories to twelve. `content` is public and owns
  authored base content and overlays; `data` remains private and owns every derived
  artifact.
- ADR 0005 is amended to license `content` under **AGPL-3.0**. Gameplay data affects a
  hosted server as directly as server code, so the closed commercial fork concern
  applies to both. `data` remains private with no license granted.
- ADR 0011 still permits public specifications to record extracted mechanics facts.
  It does not permit content rows, retail strings, or converted assets. Authored
  overlays must express SarnautCore decisions rather than literal translations.
- ADR 0021 is amended only in placement: generated bases stay in private `data`, while
  authored curation moves to public `content`.
- ADR 0029 is amended from one pack source root to the two-root model above. Its pack
  encoding, merge rules, and content-only digest remain in force.

## Consequences

The authored overlays move out of `data/overlays/`. Existing fixture packs must
regenerate byte-for-byte after the move; a changed fixture stops the migration rather
than being re-baselined.

Creating the public `SarnautCore/content` repository is owner-gated. Work may stage the
repository tree and CI integration with one explicit owner checklist item, but no
agent changes organization visibility or publishes a staged tree.

Developers with the private checkout can build InstLeague1 from both roots. Public CI
can build and test Paper Harbor from authored content alone. Neither path needs a
temporary copy of private data inside a public worktree.

The scan will have false positives as parser vocabulary grows. Exact-digest exceptions
make those cases visible and narrow. It will also miss novel leak shapes, so provenance
review remains mandatory even when CI is green.

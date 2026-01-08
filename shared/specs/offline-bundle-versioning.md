# Offline data bundle versioning plan (v1)

This spec defines how Nkokan packages and versions **offline lexicon bundles** (seed + full) so the app can:

- work offline-first on constrained devices
- update deterministically over time
- respect source disablement/removal requests
- avoid silent mutation of linguistic behavior

It is **stack-neutral**: it defines contracts and invariants, not storage tech or app architecture.

## Goals

- Define **bundle identity** and **compatibility** separately from app versioning.
- Make bundles **verifiable** (hashes) and **auditable** (manifest).
- Ensure bundles encode which transformation rules were applied (normalization/transliteration/POS mapping).
- Support **seed** and **full** bundles with the same schema and versioning semantics.

## Non-goals

- Choosing where bundles are hosted (CDN, Releases, etc.).
- Shipping any third-party data inside the git repo (bundles are artifacts, not source-controlled content).
- Defining the full lexicon schema (this spec only references it).

## Definitions

- **Bundle**: a distributable offline dataset artifact (e.g., “seed” or “full”).
- **Bundle manifest**: machine-readable metadata that describes bundle contents and provenance.
- **Bundle build**: a deterministic pipeline run that produces one or more bundles.

## Bundle types

Bundles MUST be typed:

- **`seed`**: small starter dataset for first-run experience
- **`full`**: the complete offline dataset available for download

Both bundle types MUST use the same record schema and the same manifest contract.

## Bundle identifiers

Each bundle MUST have a unique identifier:

- **`bundle_id`**: stable string identifier, unique across all time.

Recommended format (illustrative, not required):

- `bundle_{type}_{yyyymmdd}_{short_hash}`

Additionally, bundles SHOULD include:

- **`bundle_semver`**: optional semantic version for human communication (not authoritative)
- **`created_at`**: ISO-8601 timestamp

## Manifest (required)

Every bundle MUST include a manifest file (e.g., `bundle.manifest.json`) containing:

- **Identity**
  - `bundle_id`
  - `bundle_type` (`seed` | `full`)
  - `created_at`
- **Schema compatibility**
  - `record_schema_id` (e.g., `lex_v1`) and `record_schema_version`
  - `manifest_schema_version` (for this manifest format)
- **Rule versions (required)**
  - `rule_versions.normalization` (e.g., `norm_v1`) — see `shared/specs/normalization-versioning.md`
  - `rule_versions.transliteration` (e.g., `nko_translit_v1`) when applicable
  - `rule_versions.pos_mapping` (e.g., `posmap_v1`) when applicable
  - `rule_versions.url_canonicalization` (e.g., `urlcanon_v1`) when applicable
- **Sources included/excluded (required)**
  - `sources.included[]`: list of `source_id` values (from `shared/specs/source-registry.md`)
  - `sources.excluded[]`: list of excluded `source_id` values (with reasons)
- **Build lineage (recommended but high-value)**
  - `build_id` (or `bundle_build_id`)
  - `snapshot_group_ids[]` / `crawl_ids[]` used (if available)
  - `ir_parser_versions[]` used to produce IR
  - `git_commit` (repo commit that produced the bundle tooling/spec)
- **Integrity (required)**
  - `files[]`: list of bundle payload files with `{ path, byte_length, sha256 }`
  - `total_sha256` (hash of a canonicalized file list, or a hash of the archive bytes)

## Integrity rules (normative)

- Manifests MUST be deterministic: the same build inputs must produce byte-identical manifests (excluding `created_at` if you choose to include it; if included, it must be set deterministically by the build system).
- Bundle consumers MUST verify integrity using the manifest hashes before using the data.

## “No silent mutation” across bundles (normative)

If any of these change, you MUST publish a new bundle (new `bundle_id`):

- record schema version
- any `rule_versions.*` value
- included/excluded sources set
- correction dataset (approved RFC 6902 correction records) that affect outputs

This preserves reproducibility and makes trustable diffs possible.

## Disablement / removal handling (required)

If a source is disabled or a rights holder requests removal:

- A new bundle MUST be produced with that `source_id` excluded.
- The manifest MUST list that source under `sources.excluded[]` with a reason and timestamp.
- The system SHOULD support “tombstoning”:
  - keep internal auditability of what changed
  - ensure distributed bundle content no longer contains excluded material

## Seed vs full selection policy (recommended)

Seed bundle SHOULD be selected to maximize learner usefulness:

- high-frequency words/phrases first (when lawful and when data supports it)
- or curated starter list

Any selection heuristic MUST be:

- deterministic
- versioned as part of the bundle build inputs (do not depend on mutable global stats unless versioned)

## Update strategy (consumer-side)

Bundle consumers SHOULD:

- keep track of current `bundle_id`
- download a new bundle when `bundle_id` changes
- verify hashes
- import into local storage (e.g., IndexedDB) idempotently

This spec does not mandate delta updates. If deltas are introduced later, they must also be versioned and hash-verified.

## Minimal example manifest (illustrative)

```json
{
  "manifest_schema_version": "bundle_manifest_v1",
  "bundle_id": "bundle_full_20260105_ab12cd",
  "bundle_type": "full",
  "created_at": "2026-01-05T00:00:00Z",
  "record_schema_id": "lex_v1",
  "record_schema_version": "1",
  "rule_versions": {
    "normalization": "norm_v1",
    "transliteration": "nko_translit_v1",
    "pos_mapping": "posmap_v1",
    "url_canonicalization": "urlcanon_v1"
  },
  "sources": {
    "included": ["src_malipense"],
    "excluded": [
      {
        "source_id": "src_example_removed",
        "reason": "Removal request from rights holder",
        "disabled_at": "2026-01-04T00:00:00Z"
      }
    ]
  },
  "build": {
    "build_id": "build_20260105_01",
    "snapshot_group_ids": ["crawl_2026-01-04_malipense_v1"],
    "ir_parser_versions": ["parser_v1"],
    "git_commit": "abcdef123456"
  },
  "files": [
    { "path": "records.jsonl.zst", "byte_length": 1234567, "sha256": "sha256:..." }
  ],
  "total_sha256": "sha256:..."
}
```


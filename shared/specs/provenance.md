# Provenance specification (v1)

This document defines **required provenance** for any imported/derived language data in Nkokan.  
It is **stack-neutral**: it describes *data contracts*, not implementation.

## Goals

- **Traceability**: every entry/sense/example can be traced back to the source material used to create it.
- **Auditability**: we can explain *what changed*, *when*, and *why* (including normalization/transliteration versions).
- **Revocability**: if a rights holder requests removal or modification, we can disable/remove affected content without rewriting everything else.

## Non-goals

- Defining the full dictionary schema (that will be a separate spec).
- Defining storage technology (SQL/NoSQL/files/IndexedDB/etc.).
- Defining licensing terms (we only record what we know about the source).

## Levels (where provenance must exist)

Provenance is mandatory at **all three** levels:

- **Entry**: headword/lemma record (and its variants/forms).
- **Sense**: meaning-level record (gloss/definition/translation grouping).
- **Example**: example sentences/phrases (including translations if present).

If a source does not provide a level, the record still exists but may have a minimal provenance and an explicit `derivation.kind`.

## Required fields (minimum contract)

Every record that can be displayed to a user MUST carry a `provenance` object with:

- **`source.name`**: human-readable name (e.g. “Mali-pense French → Maninka dictionary”).
- **`source.url`**: canonical URL to the source landing page (or record page) *when available*.
- **`source.retrieved_at`**: ISO-8601 timestamp of when we retrieved the source material used.
- **`source.license_notes`**: free-text notes on known constraints/permission status (may be “unknown” but must not be omitted).
- **`source.record_pointer`**: a stable pointer to the specific source record used (see below).

### `record_pointer` (source pointer)

We need a pointer that is stable even if parsing rules change. This can be one of:

- **`kind: "url"`** with `value` set to a stable record URL (preferred if the source has stable per-entry URLs)
- **`kind: "source_record_id"`** with `value` set to the source’s own identifier (if published)
- **`kind: "snapshot"`** with:
  - `snapshot_id`: internal ID for an immutable raw snapshot
  - `selector`: a CSS selector / XPath / span reference to the extracted fragment (best-effort)
  - `content_hash`: hash of the extracted fragment (optional but recommended)

The key rule: **we must be able to locate the original evidence** without re-scraping.

## Derivation (how a record was produced)

Records may be:

- directly imported from a source (“as-is”)
- normalized (orthography/search keys)
- transliterated (Latin → N’Ko)
- merged from multiple sources

To keep this transparent, records SHOULD include a `derivation` object:

- **`derivation.kind`**: `imported` | `normalized` | `transliterated` | `merged` | `manual_override`
- **`derivation.inputs`**: list of input record references (when derived/merged)
- **`derivation.rule_versions`**: version stamps for:
  - normalization ruleset (e.g. `norm_v1`)
  - transliteration ruleset (e.g. `nko_translit_v1`)

No “silent mutation”: if a ruleset changes, new outputs must carry the new version.

## Multi-source attribution

When content is composed from multiple sources:

- Keep **all** contributing provenance references (do not collapse to one source).
- Prefer a structure like:
  - `provenance.primary` (the backbone source)
  - `provenance.contributors[]` (additional sources)

## Removal / disablement expectations

The system must support a rights holder request to disable/remove a source:

- Content derived from that source should be **excludable** by filtering on `source.*` and `record_pointer`.
- Removal should affect:
  - future imports and builds
  - offline bundles (seed/full) on next release
- Audit trail should remain internally (what was removed/disabled and why), but distribution should respect the request.

## Example payloads (illustrative)

### Minimal provenance (single source, imported)

```json
{
  "provenance": {
    "source": {
      "name": "Mali-pense French → Maninka dictionary",
      "url": "https://example.invalid/mali-pense",
      "retrieved_at": "2026-01-02T00:00:00Z",
      "license_notes": "Public academic resource; permission status pending; attribution required.",
      "record_pointer": { "kind": "url", "value": "https://example.invalid/mali-pense/entry/123" }
    }
  },
  "derivation": {
    "kind": "imported",
    "rule_versions": {
      "normalization": "norm_v1"
    }
  }
}
```

### Provenance with snapshot pointer (lossless capture)

```json
{
  "provenance": {
    "source": {
      "name": "Mali-pense French → Maninka dictionary",
      "url": "https://example.invalid/mali-pense",
      "retrieved_at": "2026-01-02T00:00:00Z",
      "license_notes": "Attribution required; redistribution policy under review.",
      "record_pointer": {
        "kind": "snapshot",
        "snapshot_id": "snap_9f2c...",
        "selector": "div.entry:nth-child(4) > p.headword",
        "content_hash": "sha256:..."
      }
    }
  },
  "derivation": {
    "kind": "normalized",
    "rule_versions": {
      "normalization": "norm_v1",
      "transliteration": "nko_translit_v1"
    }
  }
}
```


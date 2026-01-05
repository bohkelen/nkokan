# Lossless capture & IR specification (v1)

This spec defines how Nkokan captures third-party sources in a **lossless enough** way to support:

- iterative parsing without re-scraping
- fine-grained provenance (entry/sense/example)
- auditability and removal/disablement

It is **stack-neutral**: it defines data contracts and invariants, not implementation.

## Goals

- **Immutable raw snapshots**: the captured “evidence” is stable over time.
- **Reproducible extraction**: parsers can reference the exact fragment used.
- **Traceability**: every extracted piece can point back to a snapshot + fragment pointer.
- **Change tolerance**: parsing/normalization rules can change without requiring re-download.

## Non-goals

- Choosing a crawler or storage backend.
- Defining full dictionary schemas (entries/senses/etc.) beyond what’s needed for capture/IR links.

## Terminology

- **Source**: a third-party website/resource we ingest (e.g., Mali-pense).
- **Snapshot**: an immutable capture of a single retrieved resource (typically one HTML page).
- **Fragment**: a referenced subset of a snapshot (e.g., one dictionary entry block on the page).
- **IR (Intermediate Representation)**: a lossless-enough structured representation produced from snapshots/fragments, prior to normalization into the final lexicon.

## Phase 1.1: Raw capture (snapshots)

### Snapshot invariants

Snapshots MUST be:

- **Immutable** once stored (content never changes in-place)
- **Addressable** by a stable `snapshot_id`
- **Verifiable** by a content hash

### Required snapshot metadata (minimum)

Each snapshot MUST record:

- **`snapshot_id`**: stable internal identifier
- **`source_name`**: must match provenance `source.name`
- **`url`**: URL retrieved
- **`retrieved_at`**: ISO-8601 timestamp
- **`http_status`**: numeric status
- **`content_type`**: as returned (best-effort)
- **`content_hash`**: hash of raw bytes (e.g., `sha256:...`)
- **`byte_length`**: size of raw bytes
- **`encoding`**: if known/declared (best-effort)

Recommended (not required but useful):

- `redirect_chain[]`
- `request_headers` / `response_headers` (redact secrets if any)
- `fetch_tool_version` (crawler version)
- `robots_policy_notes` (what we believed was allowed at time of retrieval)

### Snapshot payload

Snapshot payload SHOULD store:

- raw bytes (HTML) as captured
- optionally a normalized text view *in addition* (never instead of raw)

## Fragment pointers (referencing evidence inside a snapshot)

To support provenance and auditability, IR items MUST be able to reference a fragment of a snapshot.

### Fragment pointer contract

A fragment pointer MUST include:

- **`snapshot_id`**
- **one or more locators**, best-effort, such as:
  - `css_selector`
  - `xpath`
  - `byte_span` (start/end offsets in raw bytes) when feasible
  - `text_quote` (exact snippet or “quote” for anchoring)
- **`fragment_hash`**: hash of the extracted fragment representation (recommended)

Notes:

- Locators are allowed to be imperfect (sites change), but `snapshot_id` + `fragment_hash` MUST still anchor what was used.
- For HTML, prefer a structural locator (`css_selector`/`xpath`) + a `text_quote` anchor.

## Phase 1.2: Lossless IR

IR sits between raw capture and normalized lexicon records.

### IR goals

- Preserve what the source contained, including ambiguous or messy fields.
- Keep enough structure that normalization can be deterministic and versioned later.
- Keep provenance hooks: every IR unit points back to snapshot(s) and fragment(s).

### IR unit shape (conceptual)

An IR unit SHOULD represent “one source record” (e.g., one dictionary entry on a page) and include:

- **`ir_id`**: stable internal identifier
- **`source_name`**
- **`retrieved_at`**: when its evidence snapshot was retrieved
- **`evidence`**: one or more fragment pointers (see above)
- **`fields_raw`**: extracted candidate fields *as found*, e.g.:
  - headword/lemma candidates
  - POS labels (`pos_raw`) as strings
  - gloss/translation strings
  - examples (original + translations if present)
  - notes/usage labels
- **`parse_warnings[]`**: non-fatal warnings when structure is unclear
- **`parser_version`**: version stamp of the parser used to produce this IR (not the normalization version)

IR MUST be “rerunnable”:

- Re-parsing the same snapshots with a new parser version produces a new IR set (new `parser_version`) without mutating old IR.

## Linking IR to provenance (entry/sense/example)

When IR is later converted to normalized lexicon records, provenance MUST be preserved per `shared/specs/provenance.md`.

At minimum, normalized records should be able to set:

- `provenance.source.record_pointer.kind = "snapshot"`
- `snapshot_id` from the IR evidence
- selector/span/quote information (best-effort)
- `retrieved_at` copied through

This ensures we can always explain “where did this come from?”

## Removal/disablement requirements

Design requirement: if a source needs to be disabled or removed:

- We must be able to identify all affected IR and normalized records via:
  - `source_name`
  - `snapshot_id` (and/or `url`)
- We must be able to exclude them from:
  - future normalization builds
  - offline bundles on the next release

## Example payloads (illustrative)

### Snapshot metadata

```json
{
  "snapshot_id": "snap_123",
  "source_name": "Mali-pense French → Maninka dictionary",
  "url": "https://example.invalid/page/42",
  "retrieved_at": "2026-01-04T12:34:56Z",
  "http_status": 200,
  "content_type": "text/html; charset=utf-8",
  "content_hash": "sha256:...",
  "byte_length": 54321,
  "encoding": "utf-8"
}
```

### IR unit with evidence pointers

```json
{
  "ir_id": "ir_abc",
  "source_name": "Mali-pense French → Maninka dictionary",
  "retrieved_at": "2026-01-04T12:34:56Z",
  "parser_version": "parser_v1",
  "evidence": [
    {
      "snapshot_id": "snap_123",
      "css_selector": "div.entry:nth-child(4)",
      "text_quote": "bàra …",
      "fragment_hash": "sha256:..."
    }
  ],
  "fields_raw": {
    "headword_candidates": ["bara", "bàra"],
    "pos_raw": ["n."],
    "translations_raw": ["travail", "ouvrage"],
    "examples_raw": [
      {
        "source_text": "A bàra kɛ.",
        "translation_text": "Il travaille."
      }
    ]
  },
  "parse_warnings": []
}
```


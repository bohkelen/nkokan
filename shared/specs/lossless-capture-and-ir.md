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
- **`source_id`**: stable join key from the Source Registry (see `shared/specs/source-registry.md`)
- **`source_name`**: optional display label (do NOT use as an identifier)
- **`url`**: URL retrieved (as requested/resolved at time of fetch)
- **`retrieved_at`**: ISO-8601 timestamp
- **`http_status`**: numeric status (optional for non-web sources)
- **`content_type`**: as returned (best-effort)
- **`content_hash`**: hash of raw bytes (e.g., `sha256:...`)
- **`byte_length`**: size of raw bytes
- **`encoding`**: if known/declared (best-effort)

Recommended (not required but useful):

- `snapshot_group_id` (or `crawl_id`): identifier for the crawl/batch/build that produced this snapshot
- `url_canonical`: best-effort canonical URL for record identity (deterministic; see below)
- `url_canonicalization_version`: version id of the canonicalization rules used (recommended when `url_canonical` is present)
- `source_record_id`: best-effort stable record identifier when the source exposes one (preferred over URL when available)
- `redirect_chain[]`
- `request_headers` / `response_headers` (only if a redaction policy is applied; see Security section)
- `fetch_tool_version` (crawler version)
- `robots_policy_notes` (operator belief at time of capture; not proof of permission)

#### `url_canonical` (deterministic canonicalization rules)

If you compute `url_canonical`, the transformation MUST be deterministic and versioned (via `url_canonicalization_version`) so two crawls do not create different identities for the same record.

Baseline expectations for a `urlcanon_v1` ruleset:

- normalize scheme and host case (lowercase)
- remove URL fragment (`#...`)
- remove default ports (`:80` for http, `:443` for https)
- normalize trailing slash (remove trailing slash unless the path is `/`)
- normalize query parameter ordering (sort by key, then value)
- remove tracking parameters as defined by the ruleset (e.g., `utm_*`, `gclid`, `fbclid`) while preserving content-changing parameters

Canonicalization MUST be conservative: do not drop parameters unless the ruleset explicitly marks them as tracking/non-semantic.

### Snapshot payload

Snapshot payload SHOULD store:

- raw bytes (HTML) as captured
- optionally a normalized text view *in addition* (never instead of raw)

## Security / privacy hygiene (required if storing headers)

If you store HTTP request/response headers or similar metadata, you MUST:

- apply a **redaction policy** before persistence (cookies, auth headers, tokens, session identifiers, user identifiers)
- record which policy was applied via `redaction_policy_id` (or equivalent) on the snapshot metadata when headers are stored
- store only what’s needed for reproducibility/debugging

Rationale: snapshots are “evidence,” but we must not accidentally persist secrets.

## Fragment pointers (referencing evidence inside a snapshot)

To support provenance and auditability, IR items MUST be able to reference a fragment of a snapshot.

### Fragment pointer contract

A fragment pointer MUST include:

- **`source_id`**: stable join key from the Source Registry
- **`snapshot_id`**
- **one or more locators**, best-effort, such as:
  - `css_selector`
  - `xpath`
  - `byte_span` (start/end offsets in raw bytes) when feasible
  - `text_quote` (exact snippet or “quote” for anchoring)
- **`fragment_hash`**: strongly recommended; required in Phase 2 when feasible (see below)

#### `fragment_hash` definition (must be unambiguous)

If `fragment_hash` is present, the fragment representation MUST be explicitly defined so two implementations compute the same hash.

**Phase 1 (recommended and most stable):**

- If `byte_span` is present: `fragment_hash` MUST be the hash of the exact raw byte slice referenced by `byte_span` (i.e., `snapshot_bytes[start:end]`).

If `byte_span` is not available and you still provide a `fragment_hash`:

- The pointer MUST include `fragment_representation_kind` describing what was hashed (e.g. `dom_outer_html`, `text_quote_utf8`, `pdf_text_quote_utf8`, `image_ocr_text_utf8`)
- The pointer MUST define `fragment_hash` as the hash of the UTF-8 bytes of that representation.
  - This is less preferred than `byte_span` because DOM serialization and OCR can vary by tool/version.

For non-byte-span locators (PDF/scans), implementations SHOULD prefer hashing a canonicalized locator description rather than free-form serialization.

- Recommended (collision-safe across snapshots): compute `fragment_hash` as `sha256:` of the UTF-8 bytes of an **RFC 8785 (JCS) canonical JSON** object containing:
  - `source_id`
  - `snapshot_id` (required for collision safety across snapshots)
  - the locator fields used (e.g., `page_number`/`page_index`, `rotation_degrees`, `bbox`, and `pdf_text_quote`/`ocr_text_quote` when present)
  - `fragment_representation_kind: "locator_jcs_v1"`

**Phase 2 (required when feasible):**

- For HTML (or other byte-addressable formats) where `byte_span` can be recorded, `fragment_hash` MUST be present and computed from the referenced raw byte slice.

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
- **`source_id`**: stable join key from the Source Registry
- **`source_name`**: optional display label
- **`retrieved_at`**: when its evidence snapshot was retrieved
- **`evidence`**: one or more fragment pointers (see above)
- **`record_locator`**: required identity hint for the source record within a snapshot (see below)
- **`fields_raw`**: extracted candidate fields *as found*, e.g.:
  - headword/lemma candidates
  - POS labels (`pos_raw`) as strings
  - gloss/translation strings
  - examples (original + translations if present)
  - notes/usage labels
- **`parse_warnings[]`**: non-fatal warnings when structure is unclear
- **`parser_version`**: version stamp of the parser used to produce this IR (not the normalization version)

### `record_locator` (required; prevents duplicates on multi-entry pages)

IR MUST include a `record_locator` describing how this IR unit corresponds to a specific record within the captured evidence.

#### `record_locator.kind` (minimal enum)

To prevent drift across implementations, `record_locator.kind` MUST be one of:

- `source_record_id`
- `url_canonical+entry_index`
- `css_selector+text_quote`
- `page+bbox+block_index`

Preferred (in order):

- `source_record_id` (if the source publishes stable IDs)
- else `url_canonical` + `entry_index` (index within the page/record list as parsed)
- else `css_selector` + `text_quote` (the same anchors used in evidence extraction)

The goal is not perfection; it is to ensure you can reconcile duplicates and understand how IR units relate to page structure.

#### `entry_index` semantics (required if used)

If you use `entry_index` in a `record_locator`, it MUST be:

- **0-based**
- computed in **DOM order** for HTML snapshots
- computed after applying the parser’s deterministic “entry selection” rule for that source (e.g., selector + filtering)
- stable **within a snapshot for a given parser version**; it is allowed to change across parser versions (which is why `record_locator` should be paired with evidence anchors like selectors/quotes when possible)

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

## Versioning (parser vs normalization)

This spec only requires versioning fields that prevent silent mutation and support deterministic rebuilds:

- **`parser_version`** (mandatory on IR): identifies the extractor that produced IR from snapshot+fragments.
- **Normalization / mapping versions** (mandatory downstream): normalized/display records MUST record the versions of transformation rules applied (e.g., normalization, transliteration, POS mapping). See `shared/specs/provenance.md` (`derivation.rule_versions`).

Do not overload `parser_version` to mean “normalization version”: they change for different reasons and must be recorded independently.

## Removal/disablement requirements

Design requirement: if a source needs to be disabled or removed:

- We must be able to identify all affected IR and normalized records via:
  - `source_id`
  - `snapshot_id` (and/or `url`)
- We must be able to exclude them from:
  - future normalization builds
  - offline bundles on the next release

## Human-in-the-loop corrections (editorial layer)

This spec stops at capture + IR, but it MUST be explicit about mutability boundaries:

- Raw snapshots are immutable.
- IR is immutable once written (new parser versions produce new IR; old IR is preserved).
- Human/editorial changes MUST be stored as **separate override/correction records** that reference a target record + scope; they MUST NOT mutate snapshot, fragment, IR, or normalized base records in place.

This prevents “just edit the DB row” drift and preserves auditability/rollback.

### Minimal correction record schema (required)

Any correction/override record MUST include:

- `correction_id`
- `target_id` (what record is being overridden)
- `target_scope` (e.g., `entry` | `sense` | `example` | `form`)
- `patch_payload` (the changes; format is pinned below)
- `editor_id` (may be `"system"` for automated overrides)
- `review_status` (`pending` | `approved` | `rejected`)
- `created_at` (ISO-8601)

It MAY include `reason_code` and optional evidence pointers (bbox/screenshot/audio).

#### `patch_payload` format (pinned)

To avoid “two valid interpretations,” `patch_payload` MUST be an **RFC 6902 JSON Patch** payload:

- `patch_payload` is an array of operations (`add`, `remove`, `replace`, etc.)
- each operation MUST include `op` + `path`, and `value` when required by RFC 6902

If a correction targets a record scope (`entry`/`sense`/`example`/`form`), the JSON Patch `path` MUST be interpreted relative to the JSON object representing that scope.

## Extension: books (PDFs + scans) as sources (Phase 1 compatible)

Snapshots are not only HTML. Support multiple snapshot kinds:

- `snapshot_kind`: `html` | `pdf` | `image_scan` | `text_file`

Fragment locator extensions:

### PDF fragments

Use:

- `page_number`
- `bbox` (x, y, w, h; see bbox conventions below)
- optional: `pdf_text_quote` (when the PDF has a text layer)

### Scanned image fragments

Use:

- `page_index` (or `image_id`)
- `bbox`
- optional: `ocr_text_quote` (only if OCR is performed)

### `bbox` conventions (normative)

To avoid ambiguity in highlighting and fragment comparison, `bbox` MUST follow these rules:

- **Origin**: top-left of the page/image
- **Units**: normalized floats in the range \([0, 1]\), relative to the full page/image width/height
- **Reference space**: full page/image (not a cropped subregion). If a crop is applied, record the crop explicitly alongside the locator (implementation-defined).
- **Rotation**: locators SHOULD be expressed in a normalized “upright” orientation. If the underlying page/image is rotated, record `rotation_degrees` (one of `0`, `90`, `180`, `270`) and compute `bbox` in the upright coordinate space.

### OCR stance (important)

OCR output is **not raw evidence**. Treat OCR as a **derived artifact**:

- derived from a specific snapshot/page/bbox
- stamped with `ocr_engine` + `ocr_version`
- produces its own IR units so OCR can be re-run later without corrupting lineage

#### OCR-derived IR units (required shape)

If OCR is performed, the OCR output MUST be represented as IR units that:

- include `source_id`, `parser_version`, and normal IR provenance fields
- include `ocr_engine` and `ocr_version`
- reference the underlying evidence fragment (snapshot + page + bbox) in `evidence[]`

## Normative examples

These are intentionally concrete; they are here to prevent “two correct interpretations.”

### Example A: HTML page with 3 entries → 3 IR units

Snapshot metadata:

```json
{
  "snapshot_id": "snap_123",
  "source_id": "src_malipense",
  "source_name": "Mali-pense French → Maninka dictionary",
  "snapshot_kind": "html",
  "url": "https://example.invalid/page/42",
  "snapshot_group_id": "crawl_2026-01-04_malipense_v1",
  "url_canonical": "https://example.invalid/page/42",
  "url_canonicalization_version": "urlcanon_v1",
  "source_record_id": null,
  "retrieved_at": "2026-01-04T12:34:56Z",
  "http_status": 200,
  "content_type": "text/html; charset=utf-8",
  "content_hash": "sha256:...",
  "byte_length": 54321,
  "encoding": "utf-8"
}
```

IR unit #0 (first entry on the page):

```json
{
  "ir_id": "ir_abc_0",
  "source_id": "src_malipense",
  "source_name": "Mali-pense French → Maninka dictionary",
  "retrieved_at": "2026-01-04T12:34:56Z",
  "parser_version": "parser_v1",
  "evidence": [
    {
      "source_id": "src_malipense",
      "snapshot_id": "snap_123",
      "css_selector": "div.entry:nth-child(1)",
      "text_quote": "…",
      "fragment_representation_kind": "text_quote_utf8",
      "fragment_hash": "sha256:..."
    }
  ],
  "record_locator": {
    "kind": "url_canonical+entry_index",
    "url_canonical": "https://example.invalid/page/42",
    "entry_index": 0
  },
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

The same snapshot MUST produce:

- an `entry_index: 1` IR unit for the second entry
- an `entry_index: 2` IR unit for the third entry

### Example B: PDF fragment + derived OCR IR

PDF snapshot metadata:

```json
{
  "snapshot_id": "snap_pdf_001",
  "source_id": "src_some_book",
  "source_name": "Some Book (Maninka lexicon)",
  "snapshot_kind": "pdf",
  "url": "file:///imports/some_book.pdf",
  "retrieved_at": "2026-01-04T12:34:56Z",
  "content_type": "application/pdf",
  "content_hash": "sha256:...",
  "byte_length": 987654
}
```

PDF fragment pointer (page + bbox):

```json
{
  "source_id": "src_some_book",
  "snapshot_id": "snap_pdf_001",
  "page_number": 12,
  "rotation_degrees": 0,
  "bbox": { "x": 0.10, "y": 0.25, "w": 0.80, "h": 0.10 },
  "pdf_text_quote": "…",
  "fragment_representation_kind": "locator_jcs_v1",
  "fragment_hash": "sha256:..."
}
```

Derived OCR IR unit (separate IR; references the same evidence):

```json
{
  "ir_id": "ir_ocr_12_0",
  "source_id": "src_some_book",
  "snapshot_id": "snap_pdf_001",
  "parser_version": "ocr_pipeline_v1",
  "ocr_engine": "tesseract",
  "ocr_version": "5.x",
  "evidence": [
    {
      "source_id": "src_some_book",
      "snapshot_id": "snap_pdf_001",
      "page_number": 12,
      "rotation_degrees": 0,
      "bbox": { "x": 0.10, "y": 0.25, "w": 0.80, "h": 0.10 }
    }
  ],
  "record_locator": {
    "kind": "page+bbox+block_index",
    "page_number": 12,
    "bbox": { "x": 0.10, "y": 0.25, "w": 0.80, "h": 0.10 },
    "block_index": 0
  },
  "fields_raw": {
    "ocr_text": "…"
  },
  "parse_warnings": []
}
```
```


# Normalization rules versioning strategy (v1)

This document defines how Nkokan performs **normalization** while preserving:

- Maninka-first linguistic humility (no invented language)
- provenance + traceability
- stable offline search behavior
- **no silent mutation** when rules change

It is **stack-neutral**: it specifies contracts and invariants, not implementation.

## Goals

- Define what “normalization” means in this project (and what it does not mean).
- Make normalization **deterministic and versioned** (`norm_v1`, `norm_v2`, …).
- Support **diacritics-insensitive search** without destroying display forms.
- Preserve variants and mark preferred forms without collapsing diversity.

## Non-goals

- Hardcoding linguistic assumptions (tone/length rules, etc.) inside code.
- Declaring a single “correct” orthography for Maninka.
- Mutating previously-produced normalized outputs in place.

## Core principle: separation of concerns

Normalization MUST separate:

- **Display forms**: what we show users (faithful to source + editorial corrections)
- **Search keys**: derived representations used for matching/indexing
- **Identity pointers**: stable record pointers/provenance anchors (never derived from normalized text alone)

Normalization must never overwrite evidence (snapshots/IR) or hide uncertainty.

## What counts as “normalization” (Phase 1)

Normalization is a **derived layer** applied to IR to produce stable search/display affordances.

Examples of Phase 1 normalization outputs:

- **Search keys**:
  - diacritics-insensitive key(s)
  - punctuation/whitespace-normalized key(s)
  - case-normalized key(s) (where applicable)
- **Variant handling metadata**:
  - detect and preserve spelling variants
  - mark a preferred form (source-provided or editorial)
- **POS mapping** (optional, best-effort) according to `shared/specs/pos-tagset.md`

Normalization must not:

- invent tones/length/nasalization that are not in the source
- collapse two distinct forms into one “canonical” display form

## Versioning model

Normalization rules MUST be grouped into explicit, immutable rulesets:

- `norm_v1`, `norm_v2`, …

Any normalized record MUST carry the ruleset used, e.g. via:

- `derivation.rule_versions.normalization = "norm_v1"`

Ruleset changes MUST be “append-only”:

- new ruleset version is created
- old rulesets remain available for audit and rebuilds

## No silent mutation (normative)

When a new normalization ruleset is introduced:

- previously-produced normalized outputs MUST NOT be modified in place
- instead, produce a new build/output set stamped with the new ruleset version

This supports reproducibility and preserves trust.

## Determinism requirements

For a fixed inputs set:

- IR unit(s)
- correction records (RFC 6902 patches) that apply
- normalization ruleset version (`norm_vN`)

…normalization output MUST be deterministic:

- same inputs → same outputs

If a rule depends on a mapping table, that table MUST be part of the versioned ruleset artifact.

## Search-key strategy (Phase 1)

### Multiple keys, not one

Records SHOULD store multiple search keys rather than one “magic” normalized string, e.g.:

- `search_key_diacritics_insensitive`
- `search_key_whitespace_normalized`
- `search_key_stripped_punct`

This keeps behavior explicit and testable.

### Diacritics-insensitive keys

We support “written search ignores diacritics/tone but displays them when available”.

Normative expectations:

- Keys SHOULD be computed by removing/normalizing diacritics in a deterministic way.
- Keys MUST NOT be used as display forms.
- If a source lacks diacritics, we do not “add” them in normalization.

## Variants vs preferred

Normalization MUST support:

- preserving multiple orthographic variants for the same lexeme
- marking a preferred form without deleting variants

Preferred form selection:

- MAY follow the source’s own “preferred” marker if present
- MAY follow editorial corrections (separate correction records)
- MUST be transparent and versioned (selection rules belong to `norm_vN`)

## Interaction with corrections (RFC 6902)

Corrections are applied as separate correction records (see lossless spec).

Normalization MUST define the evaluation order:

1. start from IR-derived base record(s)
2. apply approved correction patches (RFC 6902) to the relevant scope
3. compute normalization outputs (search keys, variant/preferred markers) under `norm_vN`

## Interaction with provenance and record pointers

Normalization MUST preserve provenance:

- provenance fields come from IR and source registry
- normalization adds `derivation.rule_versions.normalization`

Normalization MUST NOT change:

- `provenance.source.record_pointer` identity semantics
- snapshot/fragment anchors

## Rollout / upgrade procedure (recommended)

When proposing `norm_v(N+1)`:

- include a written changelog: what behavior changes and why
- provide a fixture set of representative examples (inputs → outputs)
- run the ruleset on the fixture set and review diffs
- only then publish the new ruleset as the current default

## Minimal metadata contract (summary)

Normalized records SHOULD be able to carry:

- `derivation.rule_versions.normalization = "norm_vN"`
- display forms (from source/IR + corrections)
- explicit search keys (derived)
- variant/preferred markers (derived)


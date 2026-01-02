# `shared/`

Shared package for cross-cutting contracts and utilities.

Planned responsibilities (Phase 0/1):

- schemas/specs for:
  - lossless IR
  - normalized entry/sense/translation/form records
  - provenance (entry/sense/example)
  - uncertainty flags
- normalization/transliteration specifications (versioned)

Goal: keep this layer as **stack-neutral** as possible so both `web/` and `api/` can rely on the same contracts.

Specs live in `shared/specs/`.


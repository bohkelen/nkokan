# POS tagset & mapping policy (v1)

This spec defines Nkokan’s **internal part-of-speech (POS) tagset** and how we map source-specific labels into it.

It is **stack-neutral**: it describes required fields and behavior, not storage or implementation.

## Goals

- Provide a **small, stable** internal POS vocabulary for consistent UI/search.
- Preserve **source fidelity** by storing original POS labels (`pos_raw`) alongside mapped POS (`pos`).
- Make it possible to add new sources/dialects later without changing the UI contract.

## Non-goals

- Linguistically exhaustive POS modeling.
- Morphosyntactic feature systems (tense/aspect/noun class, etc.) — later.

## Internal tagset (v1)

Records MUST use one of the following internal tags:

- `NOUN`
- `VERB`
- `ADJ`
- `ADV`
- `PRON`
- `PREP`
- `PART`
- `PHRASE`
- `OTHER`

### Definitions (pragmatic)

- **`NOUN`**: nouns and noun-like lexemes (including proper nouns if the source doesn’t distinguish)
- **`VERB`**: verbs
- **`ADJ`**: adjectives (including adjective-like modifiers when a source uses a dedicated label)
- **`ADV`**: adverbs
- **`PRON`**: pronouns
- **`PREP`**: prepositions / postpositions when a dedicated class exists in the source
- **`PART`**: particles, clitics, function markers that aren’t well captured elsewhere
- **`PHRASE`**: multiword expressions, idioms, set phrases
- **`OTHER`**: fallback for uncategorized or source-unknown POS

## Required fields on normalized records

Any normalized lexical record that exposes POS MUST include:

- **`pos`**: one of the internal tags above
- **`pos_raw`**: the original POS string(s) from the source (nullable if missing)
- **`pos_raw_source`**: optional short identifier of the source taxonomy (e.g. `"mali-pense"`)

If a source provides multiple POS labels for a single record:

- store all raw values in `pos_raw` (string or list, depending on eventual schema)
- choose the best single internal `pos` following the mapping policy below

## Mapping policy

### Principles

- **Do not destroy information**: keep `pos_raw` even if mapping is uncertain.
- **Prefer stability over nuance**: map into the coarse tagset; store nuance elsewhere later.
- **Be explicit about unknowns**: if you cannot confidently map, use `OTHER`.

### Versioning

POS mapping rules MUST be versioned (similar to normalization rules):

- Example version id: `posmap_v1`
- Normalized records SHOULD carry the version used, e.g. in `derivation.rule_versions.pos_mapping`.

Changes to mapping behavior must produce a new version (`posmap_v2`) rather than silently mutating existing data.

### Suggested mapping table shape (implementation-agnostic)

The mapping config SHOULD be representable as a table like:

- `match`: string/regex/synonym set (source labels)
- `maps_to`: internal POS
- `notes`: rationale

Example (illustrative):

| match (raw) | maps_to |
|---|---|
| `n.`, `nom`, `noun` | `NOUN` |
| `v.`, `verbe`, `verb` | `VERB` |
| `adj.`, `adjectif` | `ADJ` |
| `adv.`, `adverbe` | `ADV` |
| `pron.`, `pronom` | `PRON` |
| `prep.`, `préposition` | `PREP` |
| `part.`, `particle` | `PART` |
| `loc.`, `expr.`, `idiome` | `PHRASE` |

## UI/display guidance (Phase 1)

- UI SHOULD display the internal `pos` consistently (localized labels can be added later).
- If `pos_raw` exists and differs significantly, UI MAY expose it in a “Source details” panel, not in the primary display.


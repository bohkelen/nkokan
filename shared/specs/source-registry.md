# Source Registry specification (v1)

This document defines the **Source Registry**: the canonical place to store source-level metadata needed for safe ingestion, attribution, and takedown/removal workflows.

It is **stack-neutral**: it specifies required fields and behaviors, not storage technology.

## Why this exists

Do not sprinkle legal/provenance notes across entries. A single registry reduces legal risk and makes builds reproducible and removable.

## Required fields (minimum contract)

Each source MUST have a registry record with:

- **`source_id`**: stable internal identifier (machine-friendly)
- **`name`**: stable human-readable name (used in provenance; see `shared/specs/provenance.md`)
- **`homepage_url`**: canonical landing page for credit
- **`claimed_license`**: SPDX identifier or a short free-text label when SPDX is not possible
- **`license_inference_note`**: explanation when the license is not explicit (may be `"unknown"` but MUST be present)
- **`redistribution_policy`**: one of `allowed` | `not_allowed` | `unknown` (offline redistribution concerns)
- **`takedown_policy_ref`**: reference to how to handle takedowns for this source (doc link, email process, etc.)

## Recommended fields

- `record_identity_notes`: how to determine “record identity” (e.g., stable per-entry URLs, source record IDs, canonicalization notes)
- `attribution_template`: suggested credit format (e.g., Title/Author/Source/License)
- `maintainer_or_owner`: name/org if known
- `contact`: best-effort contact info
- `contact_attempted_at`
- `contact_status`: e.g. `not_attempted` | `attempted` | `responded` | `denied` | `granted`
- `disabled_at`: when the source was disabled
- `disable_reason`

## Behavioral requirements

- **Single source of truth**: builds/importers MUST reference the registry entry by `source_id` (and MAY also carry `name` for display).
- **Disablement/removal**: a source can be marked disabled so ingestion/build steps can exclude it without deleting historical evidence.


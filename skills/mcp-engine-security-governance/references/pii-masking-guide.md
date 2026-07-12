# PII Masking Guide

This guide explains how PII masking works in this server, how to enable it, and what it affects.

## Related Tools and Resources

- `run_query` (`operation: "execute"`) and `list_model` (`operation: "analyze"`) return values that may be masked
- `../../mcp-engine-bootstrap/references/troubleshooting-guide.md` (licensing/config pitfalls)

## What It Does

When enabled, the server attempts to detect and mask potentially sensitive values such as:

- emails
- payment card numbers validated with a checksum
- payment-card-like values in explicitly card-shaped columns, even when they fail Luhn validation
- international phone numbers and phone-like local formats when metadata or repeated column evidence supports them
- dates of birth in explicitly birth-date-style columns
- bank-account-style values in explicitly account-style columns
- customer-facing usernames, logins, aliases, and handles when semantic evidence is corroborated by sensitive table, data-category, or authoritative metadata context
- country- or organization-specific identifiers defined via custom detectors or explicit model annotations
- other values inferred from strong column hints, explicit model annotations, and custom patterns

Masking is applied to tool responses that return data values (e.g., query results and table previews).

## Pro Licensing Requirement

PII masking requires a Pro tier license allowing the internal feature `pii_masking`. The host mode and startup/runtime settings determine whether the server requests it. If a requested feature is unavailable, the server fails before querying or returning model data with `error_code: "MASKING_UNAVAILABLE"`.

## How to Enable

### Host mode and startup defaults

Set `MCP_ENGINE_DATA_MASKING_MODE` to:

- `configured`: runtime preferences override the `MCP_ENGINE_PII_MASKING` and `MCP_ENGINE_NUMERIC_MASKING` startup defaults.
- `off`: both maskers stay disabled and runtime toggle changes are rejected.
- `locked_on`: both maskers stay enabled and runtime toggle changes are rejected.

The equivalent JSON setting is `DataMasking:Mode`. The default mode is `configured`. The connection status and data-bearing responses report the captured mode plus each feature's requested, enabled, available, source, and optional reason state. Source values are `runtime`, `startup`, `forced`, or `unavailable`.

Version 4 removes the old per-object preference lists and deployment exclusions. Use model annotations instead. The server rejects obsolete JSON keys and these environment variables during startup:

- `MCP_ENGINE_PII_EXCLUDE_COLUMNS`
- `MCP_ENGINE_NUMERIC_EXCLUDE_COLUMNS`
- `MCP_ENGINE_NUMERIC_EXCLUDE_TABLES`
- `MCP_ENGINE_FORCE_DISABLE_PII_MASKING`
- `MCP_ENGINE_FORCE_DISABLE_NUMERIC_MASKING`

The removed preference IDs are `pii_masking_force_columns`, `pii_masking_force_tables`, `pii_masking_exclude_columns`, `pii_masking_exclude_tables`, `numeric_masking_force_columns`, `numeric_masking_force_tables`, `numeric_masking_exclude_columns`, and `numeric_masking_exclude_tables`.

### Runtime enable/disable (via preferences)

In `configured` mode, you can toggle masking at runtime using `manage_preferences` (portable preferences).
This overrides the config default and takes effect for subsequent tool calls. The `off` and `locked_on` modes reject per-toggle writes and imports with `MASKING_SETTINGS_LOCKED`.

```json
// Enable PII masking
{ "action": "set", "resource": "setting", "id": "pii_masking_enabled", "value": "true" }

// Disable PII masking
{ "action": "set", "resource": "setting", "id": "pii_masking_enabled", "value": "false" }

// List all available settings with schemas
{ "action": "list", "resource": "setting" }
```

### Related: Numeric masking

Numeric masking is controlled separately, but uses the same preference-driven toggle pattern:

- Config: `MCP_ENGINE_NUMERIC_MASKING=true|false`
- Hint profiles: `MCP_ENGINE_NUMERIC_MASKING_PROFILES="reference_common,us_common,europe_common"`
- Culture-derived profiles: `MCP_ENGINE_NUMERIC_AUTO_PROFILES_FROM_CULTURE=true|false`
- Preferences: `{ "action": "set", "resource": "setting", "id": "numeric_masking_enabled", "value": "true|false" }`

Numeric values are multiplied by one cryptographically generated scalar that remains stable for the current Desktop or Service model connection session and rotates when a new connection session begins. `ScalarMin` and `ScalarMax` must both be finite and greater than zero, and `ScalarMax` must be strictly greater than `ScalarMin`; fixed or invalid ranges are rejected during startup instead of being replaced with fallback values.

Numeric masking keeps a smaller universal structural core and expands region/reference-code hints from named profiles. The built-in profiles are:

- `reference_common`
- `us_common`
- `europe_common`
- `latam_common`
- `apac_common`
- `oceania_common`
- language semantic lexicons: `zh_hans_semantic`, `zh_hant_semantic`, `ja_semantic`, `th_semantic`

When `EnabledHintProfiles` is left unset, `reference_common`, `us_common`, and `europe_common` supply the base structural/reference-code hints. Profile-derived ratio labels are narrower: they apply only when their profile is explicitly listed in `EnabledHintProfiles` or selected from the connected model culture. For example, the `europe_common` ratio term `part` does not apply globally merely because that profile supplies default structural hints. When `AutoEnableHintProfilesFromModelCulture` is true, model culture can add a matching regional profile, such as `europe_common` for `fr-FR`, `latam_common` for `pt-BR`, `apac_common` plus `ja_semantic` for `ja-JP`, or `oceania_common` for `en-AU`. Chinese script subtags select `zh_hans_semantic` or `zh_hant_semantic`; Thai selects `th_semantic`.

Set `EnabledHintProfiles` to `[]` and `AutoEnableHintProfilesFromModelCulture=false` to disable all built-in and culture-derived numeric profiles. Profile-derived ratio labels use normalized whole-name, explicit-token, and adjacent-token phrase matching; embedded substrings and unresolved compact labels do not count as ratio evidence. A matching ratio label still requires ratio format or value-shape evidence before numeric masking is skipped.

Structural profile entries are sensitivity- and authority-classified. Reviewed non-sensitive reference and classification codes, such as postal codes, NAICS, FIPS, NUTS, NACE, municipality codes, and prefecture codes, can bypass numeric masking only as exact whole identifiers or safely anchored code names. Broad geographic or administrative nouns are corroborating evidence only; a token such as `District` cannot preserve `DistrictBudget`. Personal-capable and business/tax identifiers, including CPF, TFN, My Number, NIF, CNPJ, SIREN/SIRET, GST/GSTIN, ABN, and ACN, remain fail-closed unless authoritative model metadata or an explicit numeric annotation permits preservation. This suppression applies even when the identifier's regional numeric profile is disabled, preventing generic hints such as the `Number` suffix from preserving `MyNumber`.

Structural-name matching is Unicode-aware and uses invariant compatibility normalization rather than the process locale. Full-width forms are folded consistently, while script-essential combining marks remain part of identifier identity. Culture-selected connector rules support fully covered compounds in the recognized Latin-language cultures, while configured and profile hints may use any Unicode script. The scorer does not translate names, apply language-specific business-word lists, or guess boundaries in unsegmented scripts. Exact or delimiter-separated structural identifiers can be preserved; mixed or unresolved names are masked by default. Explicit `StrongStructuralHints` and legacy `ExcludeHints` remain operator-authoritative compatibility controls.

The shared metadata analyzer records Unicode 17.0 script, source span, boundary source, coverage, ambiguity, and unresolved ranges. Chinese, Japanese, and Thai runs remain explicitly unresolved without a dictionary segmenter. Reviewed exact native labels can still match as whole identifiers; compatibility-only, ambiguous, partial compound, and provider-error results cannot independently preserve numeric values. No external tokenizer or NLP package is loaded by the default runtime.

PII and numeric masking remain independently configurable. If PII masking or its paired detector profile is disabled while numeric masking remains enabled, sensitive numeric identifiers still receive numeric masking. If numeric masking is disabled too, protection depends entirely on the configured PII detectors; disabling both masking features returns original values by explicit operator choice.

Host mode has the highest precedence. The bundled Test Runner uses `MCP_ENGINE_DATA_MASKING_MODE=off` so its local workflow receives raw values while sharing the user's existing `~/.mcp-engine` storage.

### Custom detection patterns

Legacy regex support is still available:

- `MCP_ENGINE_PII_PATTERNS="regex1,regex2,..."`

For new configuration, prefer typed custom detectors under the `PiiMasking` section in configuration. These let you:

- assign a detector type such as `NationalId` or `TaxId`
- choose exact-only matching or substring matching inside free text
- keep user-defined `ExactOnly=true` detectors strict by default, with an opt-in to bounded token matches via `AllowTokenMatchesWhenExactOnly`
- add region-specific identifiers without baking country rules into the global defaults

### Default detector profiles

<!-- masking-catalog:start -->
### Generated masking catalog inventory

This table is generated from the runtime catalogs. Regenerate it together with the fixture snapshot after reviewed catalog changes.

| Catalog | Profile | Entries |
| --- | --- | ---: |
| Semantic labels | `cz_common` | 21 |
| Semantic labels | `de_common` | 41 |
| Semantic labels | `en_common` | 84 |
| Semantic labels | `es_common` | 30 |
| Semantic labels | `fr_common` | 28 |
| Semantic labels | `ja_common` | 19 |
| Semantic labels | `th_common` | 20 |
| Semantic labels | `zh_hans_common` | 22 |
| Semantic labels | `zh_hant_common` | 22 |
| Numeric hints | `apac_common` | 6 |
| Numeric hints | `europe_common` | 32 |
| Numeric hints | `ja_semantic` | 11 |
| Numeric hints | `latam_common` | 18 |
| Numeric hints | `oceania_common` | 5 |
| Numeric hints | `reference_common` | 13 |
| Numeric hints | `th_semantic` | 11 |
| Numeric hints | `us_common` | 8 |
| Numeric hints | `zh_hans_semantic` | 11 |
| Numeric hints | `zh_hant_semantic` | 11 |
| Numeric hints | `default` | 67 |
| PII detectors | `apac_common` | 5 |
| PII detectors | `europe_common` | 29 |
| PII detectors | `latam_common` | 4 |
| PII detectors | `oceania_common` | 1 |
| PII detectors | `us_common` | 3 |

| Numeric match mode | Entries | Hard preserve | Corroboration only |
| --- | ---: | ---: | ---: |
| `strong_ratio` | 13 | 0 | 13 |
| `structural_token` | 117 | 89 | 28 |
| `suffix_only` | 10 | 10 | 0 |
| `weak_ratio` | 53 | 0 | 53 |

| Detector overlap group | Priority order | Scope |
| --- | --- | --- |
| `apac_12_digit_national_id` | `detector/apac_common/inaadhaar` → `detector/apac_common/jpmynumber` | `IN`, `JP` |
<!-- masking-catalog:end -->

This release ships with named detector profiles enabled by default:

- `us_common`
- `europe_common`

These profiles compile into internal typed detectors during startup. They currently cover common US national/tax IDs, a broad set of European VAT-style identifiers, and registry-backed IBAN detection.

Additional opt-in profiles are also available:

- `latam_common`
- `apac_common`
- `oceania_common`

These opt-in profiles cover additional regional identifiers such as CPF/CNPJ/CURP/RFC, Aadhaar/PAN/My Number/Chinese Resident ID/Korean RRN, and Australian TFN. They are not enabled by default because several of those formats are numeric-only and are safer to opt into explicitly for your deployment. CNPJ detection accepts both existing numeric registrations and the alphanumeric format introduced for new registrations in 2026.

The shipped regional profiles also opt into bounded token matching for exact detectors. That means validated values such as `DE136695976` can still be redacted when they appear as standalone tokens inside free text or log messages, while user-defined `ExactOnly=true` detectors remain whole-value only unless you explicitly opt in.

### Shape detection and offline validation

Built-in regional detectors separate a matching value shape from a locally validated candidate:

- candidates that pass a documented checksum, range, date, country-length, or structural rule are high confidence;
- candidates that match a built-in shape but fail validation are medium confidence;
- built-in formats without a stable offline validator, such as EIN and some country-specific VAT variants, are also medium confidence;
- medium-confidence profile matches require matching semantic column/metadata context before cell values are masked;
- context-free log redaction only applies high-confidence profile matches;
- user-defined and legacy custom regex detectors keep their existing high-confidence behavior.

Validation is deterministic, allocation-bounded, exception-safe, and performs no network calls. It does not prove that an identifier was issued, is active, or belongs to a person or organization. For example, VAT validation does not call VIES, and IBAN validation checks the SWIFT registry country format plus MOD-97 without contacting a bank.

IBAN matches are reported as `BankAccount`, not `Custom`. Whole values therefore use the bank-account mask and retain only the final four alphanumeric characters. An IBAN-shaped value with an invalid checksum, country length, or BBAN structure is not a global high-confidence match; a strongly bank-account-labeled column can still treat it as contextual medium-confidence evidence.

Profile compilation writes warnings for unknown or repeated profile names and for duplicate/conflicting detector definitions. Built-ins take precedence over typed custom detectors, which take precedence over legacy patterns; later conflicting definitions are skipped deterministically.

You can override them in config:

```json
{
  "PiiMasking": {
    "EnabledDetectorProfiles": ["us_common", "europe_common", "latam_common", "apac_common", "oceania_common"]
  }
}
```

Or replace them via environment variable:

- `MCP_ENGINE_PII_DETECTOR_PROFILES="us_common,europe_common,latam_common,apac_common,oceania_common"`

To disable the shipped regional defaults entirely, set an empty list in configuration.

### Semantic label profiles and metadata evidence

PII value detector profiles are separate from semantic label profiles. Detector profiles match values
such as SSNs, VAT IDs, IBANs, and payment cards. Semantic profiles match model metadata such as
column names, table names, descriptions, display folders, translations, and model culture.

Built-in semantic profiles:

- `en_common`
- `de_common`
- `fr_common`
- `es_common`
- `cz_common`
- `zh_hans_common`
- `zh_hant_common`
- `ja_common`
- `th_common`

Configuration:

```json
{
  "PiiMasking": {
    "EnabledSemanticProfiles": ["en_common", "de_common", "fr_common"],
    "AutoEnableSemanticProfilesFromModelCulture": true,
    "FreeTextSensitivityMode": "StructuredAndContextual"
  }
}
```

Environment variables:

- `MCP_ENGINE_PII_SEMANTIC_PROFILES="en_common,de_common,fr_common"`
- `MCP_ENGINE_PII_AUTO_SEMANTIC_PROFILES_FROM_CULTURE="true"`
- `MCP_ENGINE_PII_FREE_TEXT_MODE="StructuredAndContextual"`

When culture-derived semantic profiles are enabled, model culture adds localized label evidence only.
It does not override runtime preferences, model annotations, or explicit config. Metadata is treated as
evidence, not proof: for example `Name` in a customer/contact/user table can mask, while `Name` in a
product/file/host/category context should remain visible unless explicitly forced.

Native semantic lexicons contribute positive masking evidence only. They cannot create low-sensitivity
table-role evidence or suppress a value-based detector. The generated masking catalog snapshot records
their source, review status, cultures, and scripts so unreviewed native entries fail catalog validation.

PII masking decisions now use internal scored evidence from:

- high-confidence value detectors
- localized semantic labels
- table role hints such as customer, employee, patient, contact, or user
- descriptions, display folders, and translations
- data category
- technical key, relationship, and surrogate-key signals that lower name/address confidence
- the runtime enable preference and `McpEngine_PiiMasking` annotations

Diagnostics are internal only and do not include raw sampled values.

### Legacy `ColumnHints` compatibility

`PiiMasking.ColumnHints` is the legacy untyped hint list. Case-insensitive substring
matching is retained for suspicion checks, while recognized hints are assigned to an
explicit semantic category before they can influence masking:

- `email` / `mail` → email
- `phone` / `mobile` / `cell` / `tel` → phone
- `dob` / `birth` → date of birth
- `name` compounds → name
- `address` / `street` / `shipping` / `billing` → address
- `city` → city; `state` / `province` / `region` → region
- `zip` / `postal` / `postcode` → postal code; `country` / `continent` → country
- bank-account, payment-card, tax-ID, national-ID, demographic, handle, and free-text terms → their matching semantic categories

Specific terms are resolved before generic ones, so `username` maps to the customer
contact-handle category rather than the name category. Bare `contact` and unknown
custom hints remain context-only: they can make `IsSuspiciousColumn` true, but they
do not silently become sensitive semantic categories and do not produce startup
warnings.

Handle values use the dedicated `CustomerContactHandle` PII type and render as
`[MASKED CONTACT HANDLE]`. A handle-shaped value must be 3–50 characters, contain a
letter, use only letters, digits, `@`, `.`, `_`, `-`, or `+`, and not be a generic
boolean/status/technical value. Column-name evidence alone is insufficient on
neutral or low-sensitivity technical tables.

### Freeform text redaction

Freeform fields such as `Comment`, `Notes`, `Description`, `Message`, `Feedback`,
`TicketText`, `ChatTranscript`, `Kommentar`, `Beschreibung`, and `Mensaje` are handled differently
from dedicated fields such as `Email`, `Phone`, `DOB`, or `TaxId`.

By default, freeform text uses `StructuredAndContextual` mode. SemanticOps MCP, formerly MCP Engine, scans for high-confidence
structured PII spans and redacts each accepted non-overlapping span while preserving surrounding text.
It can also redact likely names, addresses, or handles when surrounding wording and model context
indicate the text is about a customer, contact, employee, patient, or user. Multiple embedded values
no longer force a whole-value `[MASKED]` replacement unless the sensitive spans dominate the text.
Explicit `force` annotations still whole-mask the value.

Example:

```text
Customer john.smith@example.com called from +49 30 123456.
```

becomes:

```text
Customer j***@****.com called from ***-***-3456.
```

Freeform scanning is bounded to avoid pathological cost and uses the same regex timeouts as other PII
detection paths. Use `StructuredOnly` if you only want structured identifiers redacted inside comments
and do not want contextual name/address redaction.

### Per-model annotations

Persist per-object masking intent in the connected model with TOM annotations on tables and columns:

- `McpEngine_PiiMasking`: `force` or `exclude`
- `McpEngine_NumericMasking`: `force` or `exclude`

Supported targets in v1:

- tables
- source columns
- calculated columns

Use `manage_schema` to add the annotations. For example, force PII masking on a source column:

```json
{
  "operation": "update_column_properties",
  "table": "Customers",
  "target": "Email",
  "spec": {
    "annotations_upsert": [
      { "name": "McpEngine_PiiMasking", "value": "force" }
    ]
  }
}
```

Exclude a reviewed numeric reference table:

```json
{
  "operation": "update_table",
  "target": "Date",
  "spec": {
    "annotations_upsert": [
      { "name": "McpEngine_NumericMasking", "value": "exclude" }
    ]
  }
}
```

The same annotation fields work with `update_calc_column`. Remove an annotation by passing its name in `annotations_delete`.

These annotations are model-scoped intent:

- they do not enable masking by themselves
- the matching `*_masking_enabled` toggle and Pro license gate still apply
- they are advisory masking hints, not a compliance or access-control boundary

Decision order:

1. table or column annotation `exclude`
2. table or column annotation `force`
3. feature-specific heuristics and metadata rules

Tie-break rules:

- annotations override heuristics
- table-level `exclude` beats column-level `force`
- annotations never enable a masker disabled by the host mode

## What Tools Are Affected

PII masking currently applies to:

- `run_query` results (`operation: "execute"`)
- `list_model` table analysis:
  - `operation: "analyze"` with `spec: { mode: "preview" }`
  - `operation: "analyze"` with `spec: { mode: "column_stats" }` (`values` and `summary` outputs)

## Exclusions and Column/Table Awareness

Masking logic considers:

- globally stable built-ins (email, Luhn-validated payment cards)
- international-first phone detection with confidence gating
- default regional detector profiles (`us_common`, `europe_common`)
- semantic label profiles (e.g., `email`, `phone`, `Geburtsdatum`, `Téléphone`, `Número fiscal`)
- strong column and metadata hints from names, descriptions, display folders, translations, and model culture
- stronger metadata heuristics for `date of birth`, `credit card`, `bank account`, and government-ID style columns
- custom detectors and legacy custom regex patterns
- model annotations (`McpEngine_PiiMasking`, `McpEngine_NumericMasking`)
- heuristic and metadata evidence for scored masking decisions

For DAX query results, the masker now does lightweight source-column resolution for:

- qualified output headers such as `'Table'[Column]`
- simple direct projections via `SELECTCOLUMNS`, `ADDCOLUMNS`, `SUMMARIZE`, `SUMMARIZECOLUMNS`, and `ROW`

This improves parity between previews and direct renamed projections, but it is still best-effort.

Important numeric masking default in this release:

- direct-source structural columns such as keys, date parts, sort helpers, and relationship columns can still remain unmasked
- region and reference-code exclusions are now sourced from named numeric hint profiles instead of being hardcoded into the universal core
- non-sensitive localized reference/classification hints such as `NUTS` and `NACE` can preserve utility, while personal and business/tax identifiers such as `NIF`, `SIREN`, and `SIRET` no longer bypass numeric masking on name alone
- ratio-like labels such as `rate`, `margin`, `share`, `growth`, or `weight` no longer bypass masking on name alone
- localized ratio labels such as `taux`, `marge`, `Anteil`, and `croissance` also require corroborating format or value-shape evidence
- ratio-like columns are excluded from numeric masking only when corroborating evidence also supports ratio semantics
- corroborating evidence currently means percent-style formatting or ratio-shaped sampled values
- in other words, percent-style formatting is an exemption signal for ratio-like numeric columns, not a masking trigger
- unresolved or derived DAX outputs default to masking unless an explicit annotation says otherwise
- exception: a narrow set of simple aggregate expressions can inherit an existing structural exemption from a resolved source column or table
- supported inheritance is limited to obvious single-source aggregates such as `SUM(Table[Column])`, `MIN(...)`, `MAX(...)`, `AVERAGE(...)`, `COUNT(...)`, `DISTINCTCOUNT(...)`, and clear row-count outputs based on `COUNTROWS(Table)`
- constants, iterators, arithmetic, ambiguous expressions, and other unsupported derived outputs still remain masked by default

Important default in this release:

- a generic output header such as `Name` is no longer enough to trigger masking by itself
- DMV/model-metadata results and other unresolved/derived query outputs are no longer masked just because the column is named `Name`
- heuristic name masking is limited to stronger identity hints or direct source-column resolution
- region-specific IDs are shipped through named detector profiles rather than unconditional global built-ins, and can be overridden or disabled per deployment
- ambiguous local phone formats without country code are only masked when column metadata or repeated column evidence supports them
- if you need masking for a generic business column, add a model annotation

Known best-effort gaps:

- name/address heuristics are still metadata-driven and intentionally conservative
- parser-backed numeric masking redacts `FORMAT`-style text outputs when final-output provenance resolves to maskable numeric source columns
- arbitrary stringified numbers, parser-unavailable queries, unsupported conversions, and unknown or ambiguous text provenance remain best-effort gaps
- numeric masking does not attempt to reinterpret arbitrary stringified numbers without parser provenance in this release
- arbitrary DAX expressions are not rewritten or blocked in this release

## Logging Redaction vs Data Masking

This server also has a log redaction service that redacts sensitive strings in logs using the same value detector for emails, payment cards, phones, and configured custom detectors.

Important distinction:

- **Data masking** (tool responses) requires both config + Pro license.
- **Log redaction** is always active for server log sinks and is intended to reduce accidental PII leakage in diagnostic output.

## Recommended Use

- Enable PII masking in any environment where models may contain personal data.
- Add reviewed `exclude` annotations to known-safe identifier columns that match patterns accidentally.
- Avoid logging tool outputs on the client side; treat masked outputs as still potentially sensitive.

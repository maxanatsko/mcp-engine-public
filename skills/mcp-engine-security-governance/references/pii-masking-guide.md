# PII Masking Guide (Pro)

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
- country- or organization-specific identifiers defined via custom detectors, explicit model annotations, or runtime force rules
- other values inferred from strong column hints, explicit model annotations, runtime force rules, and custom patterns

Masking is applied to tool responses that return data values (e.g., query results and table previews).

## Pro Licensing Requirement

PII masking requires:

1. Configuration enabled (`MCP_ENGINE_PII_MASKING=true` or build flag), and
2. A Pro tier license allowing the internal feature `pii_masking`.

If configuration enables masking but the license is not Pro, the server logs a warning and returns unmasked data.

## How to Enable

### Runtime enable/disable

Set:

- `MCP_ENGINE_PII_MASKING=true` (enable)
- `MCP_ENGINE_PII_MASKING=false` (disable)
- `MCP_ENGINE_FORCE_DISABLE_PII_MASKING=1` (force-disable for the current process, even if preferences enable masking)

### Runtime enable/disable (via preferences)

You can also toggle masking at runtime using `manage_preferences` (portable preferences).
This overrides the config default and takes effect immediately for subsequent tool calls.

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
- Force-disable: `MCP_ENGINE_FORCE_DISABLE_NUMERIC_MASKING=1`
- Hint profiles: `MCP_ENGINE_NUMERIC_MASKING_PROFILES="reference_common,us_common,europe_common"`
- Culture-derived profiles: `MCP_ENGINE_NUMERIC_AUTO_PROFILES_FROM_CULTURE=true|false`
- Preferences: `{ "action": "set", "resource": "setting", "id": "numeric_masking_enabled", "value": "true|false" }`
- Exclusions (runtime): `numeric_masking_exclude_columns`, `numeric_masking_exclude_tables`

Numeric masking keeps a smaller universal structural core and expands region/reference-code hints from named profiles. The built-in profiles are:

- `reference_common`
- `us_common`
- `europe_common`
- `latam_common`
- `apac_common`
- `oceania_common`

When `EnabledHintProfiles` is left unset, `reference_common`, `us_common`, and `europe_common` are used as the base defaults. When `AutoEnableHintProfilesFromModelCulture` is true, model culture can add a matching regional profile, such as `latam_common` for `pt-BR`, `apac_common` for `ja-JP`, or `oceania_common` for `en-AU`. These culture-derived profiles are additive, matching PII semantic profile behavior.

Set `EnabledHintProfiles` to `[]` and `AutoEnableHintProfilesFromModelCulture=false` to disable all built-in and culture-derived numeric profiles. Ratio labels from profiles still require ratio format or value-shape evidence before numeric masking is skipped.

Force-disable environment variables have the highest precedence. They are intended for local, process-scoped hosts such as the bundled Test Runner app that need raw values while still sharing the user's existing `~/.mcp-engine` storage.

### Custom detection patterns

Legacy regex support is still available:

- `MCP_ENGINE_PII_PATTERNS="regex1,regex2,..."`

For new configuration, prefer typed custom detectors under the `PiiMasking` section in configuration. These let you:

- assign a detector type such as `NationalId` or `TaxId`
- choose exact-only matching or substring matching inside free text
- keep user-defined `ExactOnly=true` detectors strict by default, with an opt-in to bounded token matches via `AllowTokenMatchesWhenExactOnly`
- add region-specific identifiers without baking country rules into the global defaults

### Default detector profiles

This release ships with named detector profiles enabled by default:

- `us_common`
- `europe_common`

These profiles expand into typed custom detectors during startup. They currently cover common US national/tax IDs and a broad set of European VAT-style identifiers, plus an IBAN-style exact match detector.

Additional opt-in profiles are also available:

- `latam_common`
- `apac_common`
- `oceania_common`

These opt-in profiles cover additional regional identifiers such as CPF/CURP, Aadhaar/PAN/My Number/Chinese Resident ID/Korean RRN, and Australian TFN. They are not enabled by default because several of those formats are numeric-only and are safer to opt into explicitly for your deployment.

The shipped regional profiles also opt into bounded token matching for exact detectors. That means values such as `DE123456789` can still be redacted when they appear as standalone tokens inside free text or log messages, while user-defined `ExactOnly=true` detectors remain whole-value only unless you explicitly opt in.

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

PII masking decisions now use internal scored evidence from:

- high-confidence value detectors
- localized semantic labels
- table role hints such as customer, employee, patient, contact, or user
- descriptions, display folders, and translations
- data category
- technical key, relationship, and surrogate-key signals that lower name/address confidence
- explicit runtime preferences and `McpEngine_PiiMasking` annotations

Diagnostics are internal only and do not include raw sampled values.

### Freeform text redaction

Freeform fields such as `Comment`, `Notes`, `Description`, `Message`, `Feedback`,
`TicketText`, `ChatTranscript`, `Kommentar`, `Beschreibung`, and `Mensaje` are handled differently
from dedicated fields such as `Email`, `Phone`, `DOB`, or `TaxId`.

By default, freeform text uses `StructuredAndContextual` mode. MCP Engine scans for high-confidence
structured PII spans and redacts each accepted non-overlapping span while preserving surrounding text.
It can also redact likely names, addresses, or handles when surrounding wording and model context
indicate the text is about a customer, contact, employee, patient, or user. Multiple embedded values
no longer force a whole-value `[MASKED]` replacement unless the sensitive spans dominate the text.
Explicit force rules still whole-mask the value.

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

### Exclude specific columns

Provide comma-separated column names to never mask:

- `MCP_ENGINE_PII_EXCLUDE_COLUMNS="CustomerId,CountryCode,..."`

Or set exclusions at runtime via preferences (applies immediately; supports comma-separated or JSON array values):

```json
// Exclude specific columns from masking (supports Table[Column] syntax)
{ "action": "set", "resource": "setting", "id": "pii_masking_exclude_columns", "value": "Email,Phone" }

// Exclude entire tables from masking
{ "action": "set", "resource": "setting", "id": "pii_masking_exclude_tables", "value": "Audit,Logs" }
```

### Force-mask specific columns (runtime)

If you have columns that should always be treated as sensitive (even when values don’t match patterns),
you can force-mask them at runtime via preferences:

```json
// Always treat these columns as PII and mask them
{ "action": "set", "resource": "setting", "id": "pii_masking_force_columns", "value": "[\"Customers[Handle]\",\"Customers[InternalToken]\"]" }

// Always treat these tables as PII and mask them
{ "action": "set", "resource": "setting", "id": "pii_masking_force_tables", "value": "[\"Customers\"]" }

// Always numerically mask these columns (even if heuristics would usually exclude them, e.g., *ID)
{ "action": "set", "resource": "setting", "id": "numeric_masking_force_columns", "value": "[\"Customers[CustomerID]\"]" }

// Always numerically mask these tables
{ "action": "set", "resource": "setting", "id": "numeric_masking_force_tables", "value": "[\"FinanceFacts\"]" }
```

Notes:

- exclude lists (`*_exclude_columns`, `*_exclude_tables`) take precedence over force lists
- force lists do not implicitly enable masking; the matching `*_masking_enabled` toggle must also be on
- `manage_preferences` setting responses warn when force settings are configured while the masker is disabled

### Per-model annotations

You can also persist masking intent directly in the connected model with TOM annotations on tables and columns:

- `McpEngine_PiiMasking`: `force` or `exclude`
- `McpEngine_NumericMasking`: `force` or `exclude`

Supported targets in v1:

- tables
- source columns
- calculated columns

These annotations are model-scoped intent only:

- they do not enable masking by themselves
- the matching `*_masking_enabled` toggle and Pro license gate still apply
- they are advisory masking hints, not a compliance or access-control boundary

Precedence order:

1. runtime exclude preferences
2. runtime force preferences
3. annotation exclude
4. annotation force
5. existing heuristics / metadata rules

Tie-break rules:

- runtime preferences override annotations
- annotations override heuristics
- `exclude` beats `force` at the same or lower tier
- table-level `exclude` beats column-level `force`

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
- runtime exclude and force lists
- model annotations (`McpEngine_PiiMasking`, `McpEngine_NumericMasking`)
- heuristic and metadata evidence for scored masking decisions

For DAX query results, the masker now does lightweight source-column resolution for:

- qualified output headers such as `'Table'[Column]`
- simple direct projections via `SELECTCOLUMNS`, `ADDCOLUMNS`, `SUMMARIZE`, `SUMMARIZECOLUMNS`, and `ROW`

This improves parity between previews and direct renamed projections, but it is still best-effort.

Important numeric masking default in this release:

- direct-source structural columns such as keys, date parts, sort helpers, and relationship columns can still remain unmasked
- region and reference-code exclusions are now sourced from named numeric hint profiles instead of being hardcoded into the universal core
- localized structural hints such as `SIREN`, `SIRET`, `NIF`, `NUTS`, and `NACE` can preserve reference-code utility
- ratio-like labels such as `rate`, `margin`, `share`, `growth`, or `weight` no longer bypass masking on name alone
- localized ratio labels such as `taux`, `marge`, `Anteil`, and `croissance` also require corroborating format or value-shape evidence
- ratio-like columns are excluded from numeric masking only when corroborating evidence also supports ratio semantics
- corroborating evidence currently means percent-style formatting or ratio-shaped sampled values
- in other words, percent-style formatting is an exemption signal for ratio-like numeric columns, not a masking trigger
- unresolved or derived DAX outputs default to masking unless an explicit runtime/annotation rule says otherwise
- exception: a narrow set of simple aggregate expressions can inherit an existing structural exemption from a resolved source column or table
- supported inheritance is limited to obvious single-source aggregates such as `SUM(Table[Column])`, `MIN(...)`, `MAX(...)`, `AVERAGE(...)`, `COUNT(...)`, `DISTINCTCOUNT(...)`, and clear row-count outputs based on `COUNTROWS(Table)`
- constants, iterators, arithmetic, ambiguous expressions, and other unsupported derived outputs still remain masked by default

Important default in this release:

- a generic output header such as `Name` is no longer enough to trigger masking by itself
- DMV/model-metadata results and other unresolved/derived query outputs are no longer masked just because the column is named `Name`
- heuristic name masking is limited to stronger identity hints or direct source-column resolution
- region-specific IDs are shipped through named detector profiles rather than unconditional global built-ins, and can be overridden or disabled per deployment
- ambiguous local phone formats without country code are only masked when column metadata or repeated column evidence supports them
- if you need masking for a generic business column, prefer runtime force rules or model annotations

Known best-effort gaps:

- name/address heuristics are still metadata-driven and intentionally conservative
- numeric masking can still be bypassed by converting numbers to strings with functions such as `FORMAT`, `FIXED`, or string concatenation
- numeric masking does not attempt to reinterpret arbitrary stringified numbers in this release; those paths remain documented best-effort gaps
- arbitrary DAX expressions are not rewritten or blocked in this release

## Logging Redaction vs Data Masking

This server also has a log redaction service that redacts sensitive strings in logs using the same value detector for emails, payment cards, phones, and configured custom detectors.

Important distinction:

- **Data masking** (tool responses) requires both config + Pro license.
- **Log redaction** is always active for server log sinks and is intended to reduce accidental PII leakage in diagnostic output.

## Recommended Use

- Enable PII masking in any environment where models may contain personal data.
- Use exclude lists for known-safe identifier columns that match patterns accidentally.
- Avoid logging tool outputs on the client side; treat masked outputs as still potentially sensitive.

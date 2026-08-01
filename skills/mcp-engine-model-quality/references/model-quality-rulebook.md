# Semantic Model Quality Rulebook

Use these rules to ground model-quality findings in authoritative sources. Apply them with judgment and cite the relevant source link in the scorecard.

Each rule includes a read-only probe: the exact tool call sequence that produces its evidence. Run the probe before writing the finding; do not report a rule without probe output or clearly labeled inference. Numeric thresholds in probes are review triggers, not absolute limits.

Whenever a probe needs an exhaustive inventory, page it before classifying, counting, or scoring. For `list_model` `list`, add `"limit": 100, "offset": 0` to `spec`, then repeat with `spec.offset` set to `pagination.next_offset` while `pagination.has_more` is true. For exhaustive `search`, add `"format": "flat", "limit": 100, "offset": 0` to `spec` and follow the same loop. A search used only to prove existence may stop at the first match; ratios, coverage checks, duplicate detection, and absence claims require all pages.

## Source Index

- Microsoft Learn, star schema guidance: https://learn.microsoft.com/en-us/power-bi/guidance/star-schema
- Microsoft Learn, active vs inactive relationships: https://learn.microsoft.com/en-us/power-bi/guidance/relationships-active-inactive
- Microsoft Learn, bi-directional relationship guidance: https://learn.microsoft.com/en-us/power-bi/guidance/relationships-bidirectional-filtering
- Microsoft Learn, many-to-many relationship guidance: https://learn.microsoft.com/en-us/power-bi/guidance/relationships-many-to-many
- Microsoft Learn, import model data reduction: https://learn.microsoft.com/en-us/power-bi/guidance/import-modeling-data-reduction
- Microsoft Fabric, semantic model best practices for data agents: https://learn.microsoft.com/en-us/fabric/data-science/semantic-model-best-practices
- SQLBI, Power BI star schema or single table: https://www.sqlbi.com/articles/power-bi-star-schema-or-single-table/
- SQLBI, optimizing high-cardinality columns in VertiPaq: https://www.sqlbi.com/articles/optimizing-high-cardinality-columns-in-vertipaq/

## Model Shape and Star Schema

### Flat or single-table model used as the main semantic layer

- Risk: high
- Evidence to gather: one very wide visible table, mixed attributes and numeric facts, few/no dimension relationships, many report-facing columns from a fact-like table.
- Probe: `list_model` `{ "operation": "list", "spec": { "type": "tables" } }` for the table inventory, then `{ "operation": "list", "spec": { "type": "columns", "table": "<widest table>", "visibility": "visible" } }` to count visible columns, and the complete paged relationship inventory described above to count relationships touching that table. Trigger review when one visible table carries roughly 30+ visible columns and few or no relationships.
- Why it matters: star schema separates dimensions for filtering/grouping from facts for summarization, improving usability and performance.
- Recommendation: split into fact and dimension tables where feasible, define clear grains, and expose dimensions for slicing.
- Source: Microsoft star schema guidance; SQLBI star schema or single table.

### Fact and dimension concerns mixed in one table

- Risk: medium to high
- Evidence to gather: dimension attributes repeated in a large fact table, visible transaction-level columns, numeric facts and descriptive attributes in the same report-facing surface.
- Probe: `list_model` `{ "operation": "list", "spec": { "type": "columns", "table": "<fact candidate>", "visibility": "visible", "include_details": true } }` — flag text/descriptive columns exposed alongside numeric fact columns on the same table.
- Recommendation: move stable descriptive attributes to dimensions or hide technical/repeated fields when they are not user-facing.
- Source: Microsoft star schema guidance.

### Unclear or inconsistent fact grain

- Risk: high
- Evidence to gather: fact table has multiple date keys or dimension keys suggesting mixed granularities, measure definitions compensate with complex filters, relationships do not match expected grain.
- Probe: `list_model` `{ "operation": "list", "spec": { "type": "columns", "table": "<fact>", "is_key": true } }` and the complete paged relationship inventory described above filtered to the fact table; then `{ "operation": "search", "spec": { "mode": "dax", "query": "USERELATIONSHIP" } }` to find measures compensating for grain mismatches.
- Recommendation: document and normalize grain; split facts with different grains when needed.
- Source: Microsoft star schema guidance.

### Missing or weak date-table pattern

- Risk: medium to high
- Evidence to gather: no obvious date dimension, multiple date columns on facts with no active role clarity, time-intelligence measures over fact date columns.
- Probe: the classic marked-date-table flag is not exposed by `list_model` listings, so gather indirect evidence: `list_model` `{ "operation": "list", "spec": { "type": "calendars" } }` for modern calendars, the complete paged relationship inventory described above for a date-typed dimension wired to fact date keys, and `{ "operation": "search", "spec": { "mode": "dax", "query": "DATESYTD" } }` (repeat for `TOTALYTD`, `SAMEPERIODLASTYEAR`) for time intelligence over fact date columns. Confirm the marked-date-table state with the user or existing external metadata evidence; otherwise report the finding at lower confidence. Do not generate a model report as part of this read-only probe.
- Recommendation: use dedicated role-appropriate date tables or clear `USERELATIONSHIP` patterns depending on reporting needs.
- Source: Microsoft star schema guidance; Microsoft active/inactive relationship guidance.

## Relationships and Filter Propagation

### Bi-directional filtering used without a narrow justification

- Risk: high
- Evidence to gather: relationships with both-direction filtering outside one-to-one, bridge, slicer-with-data, or dimension-to-dimension scenarios.
- Probe: use the complete paged relationship inventory above — list every relationship whose cross-filter direction is both, then classify each against the allowed special cases.
- Why it matters: Microsoft recommends minimizing bi-directional relationships because they can hurt performance and confuse report users.
- Recommendation: use single-direction relationships by default and document any remaining both-direction relationship.
- Source: Microsoft bi-directional relationship guidance.

### Many-to-many relationship lacks an explicit bridge or explanation

- Risk: high
- Evidence to gather: many-to-many cardinality, unclear bridge table, visible bridge/identifier columns, ambiguous totals.
- Probe: use the complete paged relationship inventory above for many-to-many entries; `list_model` `{ "operation": "list", "spec": { "type": "columns", "table": "<bridge candidate>", "visibility": "visible" } }` for exposed bridge keys; validate a suspicious total with one scoped aggregate: `run_query` `{ "operation": "execute", "query": "EVALUATE ROW(\"total\", [<affected measure>])" }`.
- Recommendation: model the many-to-many scenario explicitly, hide bridge and identifier columns when appropriate, and validate totals with representative queries.
- Source: Microsoft many-to-many relationship guidance.

### Inactive relationships hide role-playing dimension needs

- Risk: medium to high
- Evidence to gather: important date/location/customer roles are represented by inactive relationships and only some measures use `USERELATIONSHIP`.
- Probe: use the complete paged relationship inventory above (the default includes active and inactive relationships), then `list_model` `{ "operation": "search", "spec": { "mode": "dax", "query": "USERELATIONSHIP" } }` — compare which inactive relationships are exercised by measures and which roles have none.
- Recommendation: use separate role-playing dimensions when users need to filter/group by roles simultaneously; keep inactive relationships when only specific measures need alternate propagation.
- Source: Microsoft active/inactive relationship guidance.

### Ambiguous or cyclic filter paths

- Risk: critical to high
- Evidence to gather: relationship cycles, multiple active paths, both-direction chains, unexpected totals.
- Probe: use the complete paged relationship inventory above — trace table-pair paths for cycles and chained both-direction links; confirm a suspected ambiguity with one scoped total that has a known expected value: `run_query` `{ "operation": "execute", "query": "EVALUATE ROW(\"total\", [<affected measure>])" }`.
- Recommendation: simplify relationship paths and use explicit measures for exceptional logic.
- Source: Microsoft active/inactive relationship guidance; Microsoft bi-directional relationship guidance.

## DAX and Semantic Layer

### Duplicate or overlapping measures

- Risk: medium to high
- Evidence to gather: similarly named measures such as Sales, Total Sales, Revenue, Sales Amount with unclear descriptions or different expressions.
- Probe: `list_model` `{ "operation": "list", "spec": { "type": "measures", "include_expression": true, "include_details": true } }` (`include_details` is required for descriptions; `include_expression` alone does not emit them) — group by shared name stems and by identical or near-identical expressions; flag groups whose members lack differentiating descriptions.
- Recommendation: define preferred metrics, hide/deprecate helpers, and add descriptions explaining differences.
- Source: Microsoft Fabric semantic model best practices for data agents.

### Helper measures or technical measures visible to users

- Risk: medium
- Evidence to gather: visible measures with technical prefixes, intermediate names, no descriptions, model consumers limited to other measures, and no report or external usage.
- Probe: `list_model` `{ "operation": "list", "spec": { "type": "measures", "visibility": "visible", "include_details": true } }`, then `manage_dependencies` `{ "operation": "used_by", "spec": { "target": { "type": "measure", "table": "<table>", "name": "<measure>" } } }` for each helper candidate. Model consumers limited to other measures establish only a possible helper; require corroborating technical naming or description metadata plus report metadata, external lineage, or user confirmation that the measure is not consumed directly. Without that evidence, label it an unverified helper candidate and do not deduct points or recommend hiding it.
- Recommendation: hide a helper measure or place it in a clearly internal display folder only after confirming it has no report or external consumer.
- Source: Microsoft Fabric semantic model best practices for data agents; Microsoft star schema guidance.

### Calculated columns used where source transformation or measure logic would be safer

- Risk: medium to high
- Evidence to gather: calculated columns on large tables, repeated dimension attributes, columns referenced only by measures.
- Probe: `list_model` `{ "operation": "list", "spec": { "type": "calculated_columns", "include_expression": true } }`; cross-check table size with `run_query` `{ "operation": "vertipaq", "spec": { "table": "<table>" } }` and consumers with `manage_dependencies` `{ "operation": "used_by", "spec": { "target": { "type": "column", "table": "<table>", "name": "<column>" } } }`.
- Recommendation: move row-level attributes upstream where possible; use measures for aggregations and business KPIs.
- Source: Microsoft star schema guidance; Microsoft import model data reduction guidance.

### Repeated complex DAX patterns

- Risk: medium
- Evidence to gather: repeated long expressions, copied filters, repeated time-intelligence logic, similar `CALCULATE` patterns across many measures.
- Probe: `list_model` `{ "operation": "list", "spec": { "type": "measures", "include_expression": true } }` — compare expressions for repeated filter fragments; `{ "operation": "search", "spec": { "mode": "dax", "query": "<repeated fragment>" } }` to count occurrences of a suspected pattern.
- Recommendation: centralize reusable logic through base measures, calculation groups, or clearer measure layering.
- Source: Microsoft star schema guidance; SQLBI modeling guidance.

## Storage and Performance

### High-cardinality columns on large tables

- Risk: high
- Evidence to gather: VertiPaq stats showing high cardinality or large dictionary size for GUID, URL, timestamp, transaction ID, free-text, or precise datetime columns.
- Probe: `run_query` `{ "operation": "vertipaq", "spec": { "table": "<large table>", "include_cardinality": true } }`. If `vertipaq` is unavailable, fall back to `run_query` `{ "operation": "execute", "query": "EVALUATE ROW(\"c\", DISTINCTCOUNT('<table>'[<column>]))" }` for named suspect columns.
- Recommendation: remove unused columns, split date/time when useful, reduce precision, move detail to drill-through/source system, or split identifiers only when analysis still works.
- Source: SQLBI high-cardinality VertiPaq guidance; Microsoft import model data reduction guidance.

### Unnecessary imported columns

- Risk: medium to high
- Evidence to gather: visible or hidden columns not used in relationships, sorting, grouping, summarization, security, partitions, DAX dependencies, report visuals, or external consumers.
- Probe: `list_model` `{ "operation": "list", "spec": { "type": "columns", "table": "<table>", "include_details": true } }`, then `manage_dependencies` `{ "operation": "used_by", "spec": { "target": { "type": "column", "table": "<table>", "name": "<column>" } } }` per candidate column. No returned consumers, relationship role, or sort-by role establishes only that the column appears unused inside the model. Check report metadata, lineage, and known external consumers or obtain user confirmation before calling it unused; until then label it an unverified candidate and do not add deletion to a remediation backlog.
- Recommendation: remove imported columns only after confirming they have no model, report, or external consumer; retain columns that serve user, calculation, relationship, security, sort, refresh, or downstream needs.
- Source: Microsoft import model data reduction guidance.

### Snowflaked dimensions create long filter chains without clear benefit

- Risk: medium
- Evidence to gather: dimension hierarchy split across multiple small related tables, report authors need fields from multiple snowflake tables, longer relationship chains.
- Probe: use the complete paged relationship inventory described above — find dimension-to-dimension chains of two or more hops between a fact and its outermost attribute table.
- Recommendation: denormalize dimension attributes into a single dimension table when it improves usability and performance.
- Source: Microsoft star schema guidance.

### Representative queries are slow because model shape forces complex DAX

- Risk: high
- Evidence to gather: `run_query analyze` timings, query plan pressure, repeated formula-engine-heavy patterns, or expensive bridge/filter logic.
- Probe: `run_query` `{ "operation": "analyze", "query": "<representative aggregate over a core measure>" }` — summarize total duration and the Storage Engine / Formula Engine split; a Formula Engine share above roughly 50% on a simple aggregate signals shape-forced DAX.
- Recommendation: simplify model shape first, then optimize DAX and storage.
- Source: Microsoft star schema guidance; SQLBI star schema or single table.

## Metadata, Naming, and Field Exposure

### Missing descriptions on business-facing objects

- Risk: medium
- Evidence to gather: visible tables, columns, measures, roles, or perspectives with empty descriptions.
- Probe: `list_model` `{ "operation": "list", "spec": { "type": "measures", "visibility": "visible", "include_details": true } }` (repeat for `tables` and `columns`) — count visible objects with empty descriptions and report the ratio per type.
- Recommendation: add concise business descriptions, especially for measures and ambiguous dimensions.
- Source: Microsoft Fabric semantic model best practices for data agents; Microsoft active/inactive relationship guidance for role-playing table descriptions.

### Source-system abbreviations dominate user-facing names

- Risk: medium
- Evidence to gather: cryptic table/field names, acronyms without descriptions, inconsistent suffixes.
- Probe: `list_model` `{ "operation": "list", "spec": { "type": "tables", "visibility": "visible", "include_details": true } }` and `{ "operation": "list", "spec": { "type": "columns", "visibility": "visible", "include_details": true } }` (`include_details` emits descriptions) — flag visible names that are all-caps acronyms, contain underscores or source prefixes, and lack a description.
- Recommendation: rename where possible or add descriptions/synonyms where renaming is not practical.
- Source: Microsoft Fabric semantic model best practices for data agents.

### Technical fields are visible

- Risk: medium
- Evidence to gather: visible keys, bridge columns, sort columns, flags, row hashes, ETL metadata, helper columns.
- Probe: `list_model` `{ "operation": "list", "spec": { "type": "columns", "visibility": "visible", "is_key": true } }`, plus a visible-column pass for names matching key/ID/hash/sort/ETL patterns.
- Recommendation: hide technical fields unless users genuinely analyze them; keep necessary sort/relationship columns hidden.
- Source: Microsoft star schema guidance; Microsoft many-to-many relationship guidance.

### Display folders and perspectives do not curate the model

- Risk: low to medium
- Evidence to gather: many visible measures without folders, broad perspectives, missing perspectives for known audiences.
- Probe: `list_model` `{ "operation": "list", "spec": { "type": "measures", "visibility": "visible", "include_details": true } }` — count visible measures without a display folder; `{ "operation": "list", "spec": { "type": "perspectives" } }` for perspective coverage.
- Recommendation: add or refine display folders and perspectives to improve discoverability. Do not present perspectives as security.
- Source: Microsoft Fabric semantic model best practices for data agents; Microsoft many-to-many relationship guidance.

## Governance and Security Signals

### Hidden fields or perspectives treated as security

- Risk: critical to high
- Evidence to gather: sensitive fields hidden but not covered by RLS/OLS or documented policy, perspectives used to hide sensitive columns.
- Probe: `list_model` `{ "operation": "list", "spec": { "type": "columns", "visibility": "hidden" } }` for sensitive-looking hidden columns, then `{ "operation": "list", "spec": { "type": "roles" } }` — a sensitive hidden column with no role or OLS coverage is the finding.
- Recommendation: use RLS/OLS or server-enforced policy for security; keep hiding and perspectives as curation only.
- Source: Microsoft star schema guidance; Microsoft Fabric semantic model best practices for data agents.

### RLS/OLS exists but lacks validation coverage

- Risk: medium to high
- Evidence to gather: roles are defined, but there are no tests or recent validation queries for role differences.
- Probe: `list_model` `{ "operation": "list", "spec": { "type": "roles" } }`, then `manage_tests` `{ "operation": "list" }` — roles present with zero role-related tests is the finding. With user approval, confirm behavior with `run_query` `{ "operation": "test_access", "query": "EVALUATE ROW(\"t\", [<core measure>])", "spec": { "roles": ["<role name>"] } }` (both the top-level `query` and `spec.roles` are required).
- Recommendation: add small aggregate access tests and document expected totals or visibility outcomes.
- Source: Microsoft relationship and security guidance; local SemanticOps testing guidance.

### Sensitive data exposure is unclear

- Risk: high
- Evidence to gather: visible PII-like columns, free-text fields, user identifiers, or customer attributes with no description, role, OLS, masking annotation, or policy signal.
- Probe: `list_model` `{ "operation": "list", "spec": { "type": "columns", "visibility": "visible", "include_details": true, "include_annotations": true } }` (`include_details` is required to see descriptions), then `{ "operation": "list", "spec": { "type": "roles", "include_details": true } }` for RLS/OLS coverage — flag PII-like names (email, phone, address, birth date, national ID) only when they lack a description, masking annotation, and role coverage. Do not preview the column's values.
- Recommendation: classify sensitivity, hide or secure columns, and validate user-facing access.
- Source: Microsoft Fabric semantic model best practices for data agents; local SemanticOps masking guidance.

## Validation and Test Coverage

### Core metrics lack validation queries or tests

- Risk: medium
- Evidence to gather: high-value measures with complex dependencies and no `manage_tests` suggestions or known validation examples.
- Probe: `manage_tests` `{ "operation": "list" }`, then call `manage_tests` `{ "operation": "get", "target": "<test-id>" }` for every returned test ID and inspect each complete definition's query, context, and target references against the measure inventory. If masking hides those fields, ask for explicit user approval before repeating with `{ "operation": "get", "target": "<test-id>", "spec": { "include_sensitive": true } }`; without approval, label coverage unverified and do not deduct points. Next call `manage_dependencies` `{ "operation": "summary", "spec": { "target": { "type": "measure", "table": "<table>", "name": "<measure>" } } }` for each candidate core measure (`spec.target` is required on every call; there is no global ranking operation) — report which high-dependency measures have no test. Do not infer measure coverage from test names, tags, or list summaries alone.
- Recommendation: add tests for core KPIs across representative dimensions, dates, and security contexts.
- Source: local SemanticOps testing guidance.

### Performance-sensitive queries lack benchmark coverage

- Risk: medium
- Evidence to gather: high-use measures, complex relationship paths, or VertiPaq risks with no representative query analysis.
- Probe: `manage_tests` `{ "operation": "list" }`, then call `manage_tests` `{ "operation": "get", "target": "<test-id>" }` for every returned test ID and inspect the complete `performance_budget` definitions and their queries. If masking hides those fields, ask for explicit user approval before repeating with `{ "operation": "get", "target": "<test-id>", "spec": { "include_sensitive": true } }`; without approval, label coverage unverified and do not deduct points. Check the visible queries for the measures already flagged by storage or relationship probes; absence alongside a confirmed VertiPaq or path risk is the finding. Do not infer benchmark coverage from test names, tags, or list summaries alone.
- Recommendation: create small benchmark queries and compare timings after remediation.
- Source: Microsoft import model data reduction guidance; SQLBI VertiPaq guidance.

### Changes would be risky without dependency review

- Risk: medium
- Evidence to gather: candidate cleanup touches widely referenced columns, measures, or roles.
- Probe: `manage_dependencies` `{ "operation": "used_by", "spec": { "target": { "type": "<type>", "table": "<table>", "name": "<name>" } } }` for each object named in the remediation backlog (`spec.target` is required); `{ "operation": "summary", "spec": { "target": ... } }` for breadth — attach consumer counts to the affected backlog items.
- Recommendation: use dependency analysis before cleanup and stage changes by object group.
- Source: local SemanticOps dependency guidance.

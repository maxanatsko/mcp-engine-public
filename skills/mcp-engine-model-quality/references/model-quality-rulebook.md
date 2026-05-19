# Semantic Model Quality Rulebook

Use these rules to ground model-quality findings in authoritative sources. Apply them with judgment and cite the relevant source link in the scorecard.

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
- Why it matters: star schema separates dimensions for filtering/grouping from facts for summarization, improving usability and performance.
- Recommendation: split into fact and dimension tables where feasible, define clear grains, and expose dimensions for slicing.
- Source: Microsoft star schema guidance; SQLBI star schema or single table.

### Fact and dimension concerns mixed in one table

- Risk: medium to high
- Evidence to gather: dimension attributes repeated in a large fact table, visible transaction-level columns, numeric facts and descriptive attributes in the same report-facing surface.
- Recommendation: move stable descriptive attributes to dimensions or hide technical/repeated fields when they are not user-facing.
- Source: Microsoft star schema guidance.

### Unclear or inconsistent fact grain

- Risk: high
- Evidence to gather: fact table has multiple date keys or dimension keys suggesting mixed granularities, measure definitions compensate with complex filters, relationships do not match expected grain.
- Recommendation: document and normalize grain; split facts with different grains when needed.
- Source: Microsoft star schema guidance.

### Missing or weak date-table pattern

- Risk: medium to high
- Evidence to gather: no obvious date dimension, multiple date columns on facts with no active role clarity, time-intelligence measures over fact date columns.
- Recommendation: use dedicated role-appropriate date tables or clear `USERELATIONSHIP` patterns depending on reporting needs.
- Source: Microsoft star schema guidance; Microsoft active/inactive relationship guidance.

## Relationships and Filter Propagation

### Bi-directional filtering used without a narrow justification

- Risk: high
- Evidence to gather: relationships with both-direction filtering outside one-to-one, bridge, slicer-with-data, or dimension-to-dimension scenarios.
- Why it matters: Microsoft recommends minimizing bi-directional relationships because they can hurt performance and confuse report users.
- Recommendation: use single-direction relationships by default and document any remaining both-direction relationship.
- Source: Microsoft bi-directional relationship guidance.

### Many-to-many relationship lacks an explicit bridge or explanation

- Risk: high
- Evidence to gather: many-to-many cardinality, unclear bridge table, visible bridge/identifier columns, ambiguous totals.
- Recommendation: model the many-to-many scenario explicitly, hide bridge and identifier columns when appropriate, and validate totals with representative queries.
- Source: Microsoft many-to-many relationship guidance.

### Inactive relationships hide role-playing dimension needs

- Risk: medium to high
- Evidence to gather: important date/location/customer roles are represented by inactive relationships and only some measures use `USERELATIONSHIP`.
- Recommendation: use separate role-playing dimensions when users need to filter/group by roles simultaneously; keep inactive relationships when only specific measures need alternate propagation.
- Source: Microsoft active/inactive relationship guidance.

### Ambiguous or cyclic filter paths

- Risk: critical to high
- Evidence to gather: relationship cycles, multiple active paths, both-direction chains, unexpected totals.
- Recommendation: simplify relationship paths and use explicit measures for exceptional logic.
- Source: Microsoft active/inactive relationship guidance; Microsoft bi-directional relationship guidance.

## DAX and Semantic Layer

### Duplicate or overlapping measures

- Risk: medium to high
- Evidence to gather: similarly named measures such as Sales, Total Sales, Revenue, Sales Amount with unclear descriptions or different expressions.
- Recommendation: define preferred metrics, hide/deprecate helpers, and add descriptions explaining differences.
- Source: Microsoft Fabric semantic model best practices for data agents.

### Helper measures or technical measures visible to users

- Risk: medium
- Evidence to gather: visible measures with prefixes, intermediate names, no descriptions, or dependencies used only by other measures.
- Recommendation: hide helper measures or place them in clearly internal display folders.
- Source: Microsoft Fabric semantic model best practices for data agents; Microsoft star schema guidance.

### Calculated columns used where source transformation or measure logic would be safer

- Risk: medium to high
- Evidence to gather: calculated columns on large tables, repeated dimension attributes, columns referenced only by measures.
- Recommendation: move row-level attributes upstream where possible; use measures for aggregations and business KPIs.
- Source: Microsoft star schema guidance; Microsoft import model data reduction guidance.

### Repeated complex DAX patterns

- Risk: medium
- Evidence to gather: repeated long expressions, copied filters, repeated time-intelligence logic, similar `CALCULATE` patterns across many measures.
- Recommendation: centralize reusable logic through base measures, calculation groups, or clearer measure layering.
- Source: Microsoft star schema guidance; SQLBI modeling guidance.

## Storage and Performance

### High-cardinality columns on large tables

- Risk: high
- Evidence to gather: VertiPaq stats showing high cardinality or large dictionary size for GUID, URL, timestamp, transaction ID, free-text, or precise datetime columns.
- Recommendation: remove unused columns, split date/time when useful, reduce precision, move detail to drill-through/source system, or split identifiers only when analysis still works.
- Source: SQLBI high-cardinality VertiPaq guidance; Microsoft import model data reduction guidance.

### Unnecessary imported columns

- Risk: medium to high
- Evidence to gather: visible or hidden columns not used in relationships, sorting, grouping, summarization, security, partitions, or DAX dependencies.
- Recommendation: remove unused imported columns from the model; retain only columns that serve user, calculation, relationship, security, sort, or refresh needs.
- Source: Microsoft import model data reduction guidance.

### Snowflaked dimensions create long filter chains without clear benefit

- Risk: medium
- Evidence to gather: dimension hierarchy split across multiple small related tables, report authors need fields from multiple snowflake tables, longer relationship chains.
- Recommendation: denormalize dimension attributes into a single dimension table when it improves usability and performance.
- Source: Microsoft star schema guidance.

### Representative queries are slow because model shape forces complex DAX

- Risk: high
- Evidence to gather: `run_query analyze` timings, query plan pressure, repeated formula-engine-heavy patterns, or expensive bridge/filter logic.
- Recommendation: simplify model shape first, then optimize DAX and storage.
- Source: Microsoft star schema guidance; SQLBI star schema or single table.

## Metadata, Naming, and Field Exposure

### Missing descriptions on business-facing objects

- Risk: medium
- Evidence to gather: visible tables, columns, measures, roles, or perspectives with empty descriptions.
- Recommendation: add concise business descriptions, especially for measures and ambiguous dimensions.
- Source: Microsoft Fabric semantic model best practices for data agents; Microsoft active/inactive relationship guidance for role-playing table descriptions.

### Source-system abbreviations dominate user-facing names

- Risk: medium
- Evidence to gather: cryptic table/field names, acronyms without descriptions, inconsistent suffixes.
- Recommendation: rename where possible or add descriptions/synonyms where renaming is not practical.
- Source: Microsoft Fabric semantic model best practices for data agents.

### Technical fields are visible

- Risk: medium
- Evidence to gather: visible keys, bridge columns, sort columns, flags, row hashes, ETL metadata, helper columns.
- Recommendation: hide technical fields unless users genuinely analyze them; keep necessary sort/relationship columns hidden.
- Source: Microsoft star schema guidance; Microsoft many-to-many relationship guidance.

### Display folders and perspectives do not curate the model

- Risk: low to medium
- Evidence to gather: many visible measures without folders, broad perspectives, missing perspectives for known audiences.
- Recommendation: add or refine display folders and perspectives to improve discoverability. Do not present perspectives as security.
- Source: Microsoft Fabric semantic model best practices for data agents; Microsoft many-to-many relationship guidance.

## Governance and Security Signals

### Hidden fields or perspectives treated as security

- Risk: critical to high
- Evidence to gather: sensitive fields hidden but not covered by RLS/OLS or documented policy, perspectives used to hide sensitive columns.
- Recommendation: use RLS/OLS or server-enforced policy for security; keep hiding and perspectives as curation only.
- Source: Microsoft star schema guidance; Microsoft Fabric semantic model best practices for data agents.

### RLS/OLS exists but lacks validation coverage

- Risk: medium to high
- Evidence to gather: roles are defined, but there are no tests or recent validation queries for role differences.
- Recommendation: add small aggregate access tests and document expected totals or visibility outcomes.
- Source: Microsoft relationship and security guidance; local SemanticOps testing guidance.

### Sensitive data exposure is unclear

- Risk: high
- Evidence to gather: visible PII-like columns, free-text fields, user identifiers, or customer attributes with no description, role, OLS, masking annotation, or policy signal.
- Recommendation: classify sensitivity, hide or secure columns, and validate user-facing access.
- Source: Microsoft Fabric semantic model best practices for data agents; local SemanticOps masking guidance.

## Validation and Test Coverage

### Core metrics lack validation queries or tests

- Risk: medium
- Evidence to gather: high-value measures with complex dependencies and no `manage_tests` suggestions or known validation examples.
- Recommendation: add tests for core KPIs across representative dimensions, dates, and security contexts.
- Source: local SemanticOps testing guidance.

### Performance-sensitive queries lack benchmark coverage

- Risk: medium
- Evidence to gather: high-use measures, complex relationship paths, or VertiPaq risks with no representative query analysis.
- Recommendation: create small benchmark queries and compare timings after remediation.
- Source: Microsoft import model data reduction guidance; SQLBI VertiPaq guidance.

### Changes would be risky without dependency review

- Risk: medium
- Evidence to gather: candidate cleanup touches widely referenced columns, measures, or roles.
- Recommendation: use dependency analysis before cleanup and stage changes by object group.
- Source: local SemanticOps dependency guidance.

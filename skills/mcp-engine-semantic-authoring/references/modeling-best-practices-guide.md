# Modeling Best Practices Guide (Pro)

This guide is a practical checklist for building maintainable, performant Power BI semantic models. It is written to be used alongside this server’s tools.

## Related Tools and Resources

- Discovery: `list_model` (`operation: "list"`, `operation: "search"`)
- Authoring: `manage_schema` (tables, columns, partitions, relationships, hierarchies), `manage_semantic` (measures, calc groups, named expressions), `manage_model_properties`
- Validation: `run_query` (`operation: "execute"`, `operation: "analyze"` - Pro)
- Storage analysis: `run_query` (`operation: "vertipaq"` - Pro)
- Model tracking: `manage_model_changes` (Pro)
- Related resources:
  - `../../mcp-engine-query/references/dax-query-guide.md`
  - `../../mcp-engine-schema-authoring/references/relationships-guide.md`
  - `measure-authoring-guide.md`
  - `../../mcp-engine-schema-authoring/references/column-and-table-authoring-guide.md`
  - `../../mcp-engine-schema-authoring/references/hierarchies-guide.md`
  - `calc-groups-guide.md`
  - `../../mcp-engine-schema-authoring/references/incremental-refresh-policy-guide.md` (Pro)
  - `../../mcp-engine-query/references/vertipaq-optimization-guide.md` (Pro)

## 1) Schema Design (Star First)

- Prefer a star schema: facts in the center, dimensions around.
- Avoid snowflaking unless it materially reduces duplication and doesn’t introduce ambiguous filter paths.
- Keep relationship keys stable and low-cardinality where possible (surrogate integer keys are often best).

Checklist:

- `list_model` with `operation: "list"`, `spec: { type: "tables" }`: identify fact vs dimension tables.
- `list_model` with `operation: "list"`, `spec: { type: "relationships" }`: confirm there are no unintended cycles/ambiguities.

## 2) Relationships

- Use single-direction filtering by default (`cross_filter_direction="OneDirection"`).
- Use bidirectional relationships sparingly, only with a clear reason and ambiguity checks.
- Prefer inactive relationships for alternate paths (e.g., multiple date keys) and use `USERELATIONSHIP` in measures.

Before changing relationships:

- Run `list_model` with `operation: "search"` and `spec: { mode: "dax", query: "USERELATIONSHIP(" }` to understand dependencies.
- Re-validate key measures with `run_query` (`operation: "execute"`).

## 3) Date Tables and Time Intelligence

- Always use a dedicated date table for time intelligence.
- Mark it as the model date table and choose a Date/DateTime column as the date column.

Typical workflow:

1. Ensure date table exists and has a proper Date column (`list_model` with `operation: "list"`, `spec: { type: "columns", table: "DimDate" }`).
2. Mark it: `manage_schema` with `operation: "update_table"` and `spec: { mark_date_table: true, date_column: "..." }`.
3. Use calculation groups (if applicable) for consistent time intelligence transformations.

## 4) Measures vs Calculated Columns

- Prefer measures for aggregations and business KPIs (evaluated at query time).
- Use calculated columns only when you need persisted row-level attributes (they often increase model size).
- Avoid “calculated columns that replicate dimension attributes” on fact tables unless necessary.

Refactoring tip:

- If a calculated column is only used inside measures, consider moving the logic into measures/variables.

## 5) Naming, Folders, and Semantics

- Use consistent naming conventions for tables/columns/measures.
- Use `display_folder` for measures and columns to keep the field list curated.
- Hide technical keys and helper measures/columns (`is_hidden=true`) while keeping the model functional.
- Add descriptions for business-facing objects.

Suggested conventions:

- Measures: noun phrases for outcomes (e.g., `Total Sales`, `Sales YTD`, `Margin %`).
- Helper measures: prefix like `_` or folder like `Internal\\...`, and set `is_hidden=true`.

## 6) Avoiding Model Bloat (VertiPaq)

Common bloat sources:

- High-cardinality text columns in large tables
- GUIDs and long strings used widely
- Unused columns imported “just in case”

Workflow (Pro):

1. Run `run_query` with `operation: "vertipaq"` to find the largest tables/columns.
2. Remove unused columns and re-check size and performance.
3. Prefer dimension attributes over repeating attributes on facts.

## 7) Partitions, Refresh, and Incremental Refresh

- Partitioning and refresh strategy should match data size and latency requirements.
- Use incremental refresh for large fact tables sourced via Power Query (M).
- Ensure `RangeStart`/`RangeEnd` parameters exist for incremental refresh policies.

See:

- `../../mcp-engine-schema-authoring/references/partitions-refresh-guide.md`
- `../../mcp-engine-schema-authoring/references/incremental-refresh-policy-guide.md` (Pro)

## 8) Calculation Groups (When They Help)

Use calculation groups when you need a consistent transformation applied across many measures:

- time intelligence (YTD, MTD, YoY)
- currency conversion
- scenario switching

Guidance:

- Keep calc item expressions simple and use `SELECTEDMEASURE()` patterns.
- Use `format_string_expression` when the transformation changes units/meaning.

## 9) Security is Not Curation

- Perspectives are for curation, not security.
- Use roles (RLS/OLS) for actual access control.
- Validate that sensitive columns are protected even if hidden.

See:

- `../../mcp-engine-security-governance/references/security-roles-guide.md`
- `../../mcp-engine-security-governance/references/perspectives-guide.md`

## 10) Validation and Regression Checks

Before/after a change (especially refactors):

- `list_model` with `operation: "search"` and `spec: { mode: "dax", query: "[MeasureName]" }` to find impacted dependencies.
- `run_query` validation queries for core KPIs (multiple scenarios/filters).
- `run_query` with `operation: "analyze"` (Pro) for performance regressions on representative queries.
- Consider pinning a checkpoint and/or using changesets before large batches (`../../mcp-engine-testing-changes/references/model-changes-guide.md`).

## 11) Microsoft Fabric and Lakehouse Considerations

When using Microsoft Fabric, modeling decisions differ from traditional Power BI Desktop:

### Direct Lake Mode

- Semantic models can read directly from Delta tables in OneLake
- No data import/refresh required (data is always current)
- Some DAX patterns cause "fallback" to DirectQuery (monitor capacity metrics)
- Not all Power BI features are available in Direct Lake mode

**Best practices for Direct Lake:**
- Keep measures simple to avoid fallback
- Test representative queries for fallback behavior
- Consider aggregations for complex calculations
- Use Fabric capacity admin tools to monitor performance

### Lakehouse vs Warehouse

- **Lakehouse**: Best for unstructured/semi-structured data, data engineering workloads
- **Warehouse**: Best for structured data, SQL-centric workloads, T-SQL compatibility

Both can serve as sources for Direct Lake semantic models.

### Shortcuts vs Data Copy

- **Shortcuts**: Reference data in other locations without copying (OneLake, ADLS, S3)
- **Copy**: Physically move data into the lakehouse

Shortcuts minimize storage costs but may have cross-region latency considerations.

### Hybrid Approaches

Consider composite models combining:
- Direct Lake for large fact tables (real-time, no refresh)
- Import for small dimension tables (best performance)
- DirectQuery for external sources when needed

### Tool Considerations

This MCP server connects to Power BI Desktop models. For Fabric semantic models:
- Use XMLA endpoints for programmatic access
- Some operations may behave differently in Direct Lake mode
- Incremental refresh policies are not applicable to Direct Lake tables

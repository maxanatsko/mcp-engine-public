# Troubleshooting Guide

This guide lists common issues when using MCP Engine (the MCP server for Power BI Desktop) and practical steps to resolve them.

## Related Tools and Resources

- `manage_model_connection`: `operation="open"|"list"|"list_workspaces"|"select"|"get_current"|"reload"|"authenticate"|"sign_out"|"set_impersonation"` for model selection, service sign-in, and metadata refresh
- `list_model`: `operation="list"|"search"|"analyze"|"info"` for discovery and diagnosis
- `run_query`: `operation="execute"|"analyze"|"vertipaq"|"test_access"` for validation and performance checks
- `tool-invocation-conventions.md`: argument shape and bulk patterns

## Connection and Discovery

### “Not connected” / no model selected

Symptoms:

- Tools that require a live model connection return an error about connection/model selection.

Fix:

1. Run `manage_model_connection`:
   - `operation="list"` to discover available models
   - `operation="select"` to select the intended model
   - `operation="get_current"` to confirm
2. Then retry the tool.

### Tools return empty results unexpectedly

Common causes:

- Wrong model selected (multiple Power BI Desktop instances).
- Filtering too aggressively (`list_model` with incorrect `table` or `name_filter`).
- `list_model` `operation="list"` filters internal/system artifacts by default. For raw diagnostics, set `spec.include_system_artifacts: true`.

Fix:

- Re-check selected model with `manage_model_connection operation="get_current"`.
- Use broader discovery:
  - `list_model` with `operation: "list"`, `spec: { type: "tables" }`
  - `list_model` with `operation: "search"`, `spec: { query: "...", mode: "name" }` (wildcards supported)
  - `list_model` with `operation: "search"`, `spec: { query: "...", mode: "text" }` (substring over metadata; optionally `include_fields: ["folders","formatting","members","security"]`)

### "Model changed externally" / "Operation blocked"

Symptoms:

- Tools fail with an error mentioning "model changed externally" or "Operation blocked".
- A prompt may appear asking to reload metadata.

Cause:

- The model was modified outside of MCP (e.g., in Power BI Desktop UI or Tabular Editor) since MCP last read the metadata.
- MCP detects this via a best-effort mechanism:
  - Power BI Desktop: uses an Analysis Services server trace when available (more reliable for metadata edits).
  - Fallback: compares the model's last-modified timestamp with a stored baseline.

Fix:

1. Run `manage_model_connection` with `operation="reload"` to refresh TOM metadata.
2. Retry your original tool call.

Alternatively, if prompted to reload via elicitation:

- Accept the reload prompt to refresh metadata automatically.
- Decline if you want to continue with potentially stale metadata (only works if `ExternalChangeFailClosed=false`).

Configuration:

- Set `MCP_ENGINE_EXTERNAL_CHANGE_AUTO_RELOAD=true` to automatically reload without prompting.
- Set `MCP_ENGINE_EXTERNAL_CHANGE_FAIL_CLOSED=false` to allow operations on stale metadata.

## Argument Shape and Naming Issues

### “Unknown property” / “missing required field”

Cause:

- Tool argument keys are strict, and the server expects `snake_case`.
- Some clients default to `camelCase` field names (or auto-transform keys), which can cause schema mismatches.

Fix:

- Inspect the tool schema via the MCP `tools/list` method and use the exact key names.
- All tools use `snake_case` consistently:
  - `manage_semantic`: spec contains `format_string`, `format_string_expression`, `display_folder`, `is_hidden`
  - `manage_schema`: spec contains `format_string`, `display_folder`, `is_hidden`, `new_name`

See: `tool-invocation-conventions.md`.

### Bundled reference path treated as a web URL

Cause:

- Bundled reference paths in this skill are local markdown files, not HTTP(S) URLs.

Fix:

- Open the corresponding markdown file in this skill's `references/` directory directly.
- Use `resources/list` when you need to discover available documentation resources.
- Do not send bundled reference paths to `web_fetch`.

### Bulk mode “items must be an array”

Cause:

- Bulk-enabled write tools expect `items: [ { ... }, { ... } ]`.

Fix:

```json
{
  "operation": "update_measure",
  "transaction": true,
  "dry_run": true,
  "items": [
    { "table": "Sales", "target": "Total Sales", "spec": { "expression": "..." } }
  ]
}
```

Exception:

- `manage_semantic` with `create_calc_group` uses `bulk_items` for bulk operations (because `items` is reserved for calculation items).

Bulk validation notes:

- Top-level object identifiers such as `table` and `target` are single-item fields.
- In bulk mode, each item must include its own identifiers; missing fields are reported per item instead of inheriting top-level values.

## DAX and Expression Errors

### DAX syntax error / “The expression is not valid”

Fix:

- Validate quoting rules (tables in quotes, columns/measures in brackets).
- Use `../../mcp-engine-query/references/dax-query-guide.md` patterns.
- If the failing expression is in a measure, search for dependencies:
  - `list_model` with `operation: "search"` and `spec: { mode: "dax", query:"MeasureName"` or query for `"[Measure]"`.

### DAX query returns no rows

Common causes:

- Filters eliminate all rows.
- Using `SUMMARIZECOLUMNS` without measures (may return no rows because it removes all-blank rows).

Fix:

- Start from a simpler query:
  - `EVALUATE TOPN(10, 'Table')`
- Add filters gradually.

### `COUNTROWS(...)` or scalar queries return `null`

Cause:

- `run_query` preserves DAX `BLANK` as JSON `null`.
- This can happen for genuinely blank results, not only for missing tables or objects.

Fix:

- Treat `null` as a DAX result first, then verify object existence separately with `list_model`.
- When debugging a newly created calculated table, inspect the create-table warnings and processing signals before assuming the table is missing.

## Relationship / Modeling Issues

### Ambiguous filter path / unexpected totals

Cause:

- Multiple paths between tables, bidirectional relationships, or incorrect cardinalities.

Fix:

- Inspect relationships:
  - `list_model` with `operation: "list"`, `spec: { type: "relationships" }`
- Prefer:
  - star schema
  - `cross_filter_direction="OneDirection"`
- Use inactive relationships for alternate date keys and `USERELATIONSHIP` in measures.

See: `../../mcp-engine-schema-authoring/references/relationships-guide.md` and `../../mcp-engine-semantic-authoring/references/modeling-best-practices-guide.md` (Pro).

### Relationship was created but `RELATED()` or validation still fails

Cause:

- New relationships can require a `Calculate` refresh before validation-sensitive measures see the updated relationship state.

Fix:

1. Run `refresh` with `scope: "Table"` or `scope: "Model"` and `refresh_type: "Calculate"`.
2. Retry the measure validation or DAX query.
3. If the error persists, inspect the relationship definition and key column types.

## Compatibility Level Issues

### “Compatibility level too low” (calc groups / incremental refresh / UDFs)

Cause:

- The feature requires a minimum model compatibility level.

Fix:

1. Check current level:
   - `manage_model_properties operation="get"`
2. Upgrade if appropriate:

```json
{
  "operation": "update",
  "compatibility_level": 1702,
  "allow_compatibility_upgrade": true
}
```

Notes:

- Upgrades are irreversible and may be blocked by the host.
- Keep a PBIX backup before upgrading.

## Incremental Refresh Policy Issues

### RangeStart/RangeEnd not found

Cause:

- Required DateTime parameters are missing.

Fix:

- Create Power Query parameters named `RangeStart` and `RangeEnd` (DateTime), or set `range_start_parameter` / `range_end_parameter` to the names you use.

See: `../../mcp-engine-schema-authoring/references/incremental-refresh-policy-guide.md` (Pro).

### “refresh_policy.source_expression is required”

Fix:

- Provide `refresh_policy.source_expression`, or ensure the first partition has an M expression the server can clone.

## Licensing and Pro Gating

### Resource or parameter requires Pro

Symptoms:

- `resources/read` fails with “requires Pro tier”.
- Some tool parameters are gated (e.g., performance plan capture).

Fix:

- Use non-Pro equivalents where possible, or upgrade your tier.
- For performance: run without `include_query_plan` first, then enable it when needed.

## Large Outputs / Token Pressure

### Responses are too large (query plans, expressions)

Fix:

- Prefer preview patterns:
  - `list_model` with `operation: "list"`, `spec: { type: "measures", include_expression: false, preview_chars: N }` (adjust `type` as needed)
- Only enable:
  - `include_expression=true`
  - `include_query_plan=true`
  - `include_details=true`
  when you actually need the full output.

## Expression and Reference Errors

### "Expression references a column that doesn't exist"

Cause:

- Column was renamed or deleted
- Relationship not established
- Typo in column/table name

Fix:

1. Use `list_model` with `operation: "search"` and `spec: { mode: "dax", query: "'TableName'[ColumnName]" }` to find broken references
2. Check column exists: `list_model` with `operation: "list"`, `spec: { type: "columns", table: "TableName" }`
3. Check relationship setup: `list_model` with `operation: "list"`, `spec: { type: "relationships" }`
4. Update the expression with correct column reference

### "Circular dependency detected"

Cause:

- Measure A references Measure B, which references Measure A
- Calculated column references itself
- Calculation group items with circular references

Fix:

1. Map the dependency chain:
   - `list_model` with `operation: "search"` and `spec: { mode: "dax", query: "[MeasureName]" }`
2. Refactor to break the cycle:
   - Use variables (VAR) to store intermediate results
   - Restructure measure logic to avoid mutual references

### "The expression for column cannot be determined"

Cause:

- Calculated column expression has row context issues
- Missing relationship for RELATED/RELATEDTABLE

Fix:

- For calculated columns, ensure the expression evaluates in row context
- Verify relationships exist: `list_model` with `operation: "list"`, `spec: { type: "relationships" }`
- Consider using measures instead of calculated columns for aggregations

## Direct Lake / Fabric Issues

### Direct Lake fallback to DirectQuery

Symptoms:

- Queries slower than expected
- Fabric capacity metrics show high fallback rate

Common causes:

- Complex iterators (SUMX, FILTER with many rows)
- Unsupported DAX patterns
- Cross-filter scenarios not optimized for Direct Lake

Fix:

- Simplify measure logic
- Pre-aggregate in the lakehouse where possible
- Check Fabric documentation for supported patterns
- Consider hybrid model with aggregation tables

### "Cannot connect to semantic model"

Cause (Fabric):

- XMLA endpoint not enabled
- Capacity paused or unavailable
- Permission issues

Fix:

- Verify XMLA read/write is enabled in capacity settings
- Check capacity is running
- Verify user has appropriate workspace permissions

## Refresh and Processing Errors

### "Refresh failed: Timeout expired"

Cause:

- Large data volume
- Slow data source
- Complex Power Query transformations

Fix:

- Increase timeout settings where available
- Optimize Power Query (reduce steps, use query folding)
- Consider incremental refresh for large tables
- Check data source performance

### "Memory allocation failure during refresh"

Cause:

- Model too large for available memory
- High-cardinality columns consuming memory

Fix:

- Reduce model size (see `../../mcp-engine-query/references/vertipaq-optimization-guide.md`)
- Remove unused columns
- Consider aggregations or Direct Lake mode
- Upgrade capacity tier if needed

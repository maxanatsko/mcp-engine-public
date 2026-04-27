# Measure Authoring Guide (`manage_semantic`)

This guide explains common patterns for creating, updating, and refactoring measures with `manage_semantic`.

## Related Tools

- `manage_semantic`: Create/update/delete measures (operations: `create_measure`, `update_measure`, `delete_measure`)
- `manage_security`: Manage perspectives (operations: `create_perspective`, `update_perspective`, `delete_perspective`) when you want curated measure/table subsets for report consumers. Perspectives live on `manage_security`; see `../../mcp-engine-security-governance/references/perspectives-guide.md`.
- `list_model`: Inventory measures (`operation: "list"`, `spec: { type: "measures" }`)
- `list_model`: Find dependencies by searching expressions (`operation: "search"`, `spec: { mode: "dax", query: "[MeasureName]" }`)
- `run_query`: Validate results for key scenarios (`operation: "execute"`)

## When to Use a Measure vs a Calculated Column

- Prefer **measures** for aggregations, ratios, and time intelligence (evaluated at query time).
- Prefer **calculated columns** (via `manage_schema` with `operation: "create_calc_column"`) when you need a row-level attribute persisted in the model (often increases model size).

## Create a Measure

```json
{
  "operation": "create_measure",
  "table": "Sales",
  "target": "Total Sales",
  "spec": {
    "expression": "SUM('Sales'[Amount])",
    "validate": true,
    "format_string": "$#,0",
    "display_folder": "Sales\\Revenue",
    "description": "Total sales amount in USD"
  }
}
```

### Validation Errors

Power BI Desktop can accept a measure definition even when its DAX has a compilation/evaluation error. By default, `manage_semantic` runs a lightweight DAX query to evaluate the measure after save and returns any engine error in the response.

To skip validation (for performance or when authoring many measures at once), set `spec.validate=false`.

## Update a Measure

Update only the fields you intend to change:

```json
{
  "operation": "update_measure",
  "table": "Sales",
  "target": "Total Sales",
  "spec": {
    "validate": true,
    "expression": "SUMX('Sales', 'Sales'[Amount])"
  }
}
```

### Rename a Measure

```json
{
  "operation": "update_measure",
  "table": "Sales",
  "target": "Total Sales",
  "spec": { "new_name": "Total Sales Amount" }
}
```

## Formatting

### Static format string

Use `format_string` for a constant format:

```json
{
  "operation": "update_measure",
  "table": "Sales",
  "target": "Total Sales",
  "spec": { "format_string": "#,##0.00" }
}
```

### Dynamic format string expression

Use `format_string_expression` to compute formatting (e.g., units or percent vs currency):

```json
{
  "operation": "update_measure",
  "table": "Sales",
  "target": "KPI Value",
  "spec": {
    "format_string_expression": "IF(SELECTEDVALUE('Units'[Unit]) = \"%\", \"0.0%\", \"#,##0\")"
  }
}
```

Important:

- `format_string_expression` and `format_string` are mutually exclusive.

## Display Folders and Presentation

- `display_folder` is a path-like string (commonly `Folder\\Subfolder`).
- `is_hidden` hides the measure in client tools (use for intermediate helpers).
- `data_category` can be set for special rendering (e.g., `WebUrl`, `ImageUrl`).

## Detail Rows Expression (Drillthrough)

Set drillthrough detail rows:

```json
{
  "operation": "update_measure",
  "table": "Sales",
  "target": "Total Sales",
  "spec": {
    "detail_rows_expression": "SELECTCOLUMNS('Sales', \"OrderId\", 'Sales'[OrderId], \"Amount\", 'Sales'[Amount])"
  }
}
```

Clear it:

```json
{
  "operation": "update_measure",
  "table": "Sales",
  "target": "Total Sales",
  "spec": { "clear_detail_rows": true }
}
```

## Dependency-Aware Refactors

Measure renames do not automatically rewrite other DAX expressions. Recommended workflow:

1. Identify dependents:
   - `list_model` with `operation: "search"` and `spec: { mode: "dax", query: "[Total Sales]" }`.
2. Rename with `new_name` in spec.
3. Update dependent measures / calc columns / calc items that reference the old name.
4. Validate with `run_query` (`operation: "execute"`) for key visuals.

Example dependency search:

```json
{ "operation": "search", "spec": { "query": "[Total Sales]", "mode": "dax", "limit_per_type": 50 } }
```

## Bulk Authoring

Create multiple measures:

```json
{
  "operation": "create_measure",
  "transaction": true,
  "items": [
    { "table": "Sales", "target": "Total Sales", "spec": { "expression": "SUM('Sales'[Amount])", "format_string": "$#,0" } },
    { "table": "Sales", "target": "Total Cost", "spec": { "expression": "SUM('Sales'[Cost])", "format_string": "$#,0" } }
  ]
}
```

Use `dry_run=true` first when changing many measures.

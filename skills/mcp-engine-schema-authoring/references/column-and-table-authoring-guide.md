# Column and Table Authoring Guide (`manage_schema`)

This guide covers common table/column operations, data type gotchas, and schema-safe workflows.

## Related Tools

- `manage_schema`: Create/update/delete tables (operations: `create_table`, `update_table`, `delete_table`)
- `manage_schema`: Create/update field parameters (operations: `create_field_parameter`, `update_field_parameter`)
- `manage_schema`: Create/update/delete/refresh partitions (operations: `create_partition`, `update_partition`, `delete_partition`, `refresh_partition`)
- `manage_schema`: Create/delete source columns; update column properties; create/update/delete calculated columns (operations: `create_column`, `delete_column`, `update_column_properties`, `create_calc_column`, `update_calc_column`, `delete_calc_column`)
- `manage_security`: Manage perspectives (operations: `create_perspective`, `update_perspective`, `delete_perspective`). Perspectives are presentation/curation features hosted on `manage_security` for backward compatibility; see `perspectives-guide.md (from the mcp-engine-security-governance skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`.
- `list_model`: Inspect current schema (`operation: "list"`, `spec: { type: "tables|columns|calculated_columns|partitions" }`). For columns, use `include_details: true` to include formatting metadata (`format_string`, `data_category`, `summarize_by`, `source_column`, etc.). For tables, `table_type` indicates `"calculated"` vs `"regular"`.
- `list_model`: Find references before refactors (`operation: "search"`):
  - DAX: `spec: { mode: "dax", query: "..." }`
  - M: `spec: { mode: "m", query: "..." }`
  - Non-M partitions: `spec: { mode: "partition", query: "..." }`
  - Metadata: `spec: { mode: "text", query: "..." }`

## Data Types (`data_type`)

When defining columns for table creation or partition schema updates, `data_type` accepts common names:

- `Int64` (also: `whole`, `integer`)
- `Double` (also: `number`, `float`)
- `Decimal` (also: `currency`)
- `String` (also: `text`)
- `DateTime` (also: `date`, `time`)
- `Boolean` (also: `bool`)

Unknown types default to `String`.

## Creating Tables (`manage_schema` with `create_table`)

`create_table` creates a table with an initial partition. Partition types include:

- `M` (Power Query)
- `Query` (SQL)
- `Calculated` (DAX calculated table)
- `Entity` (Dataflow entity)

Optional formatting:

- `format_m`: only applies when `partition_type: "M"` and `expression_kind` is literal
- `format_dax`: only applies when `partition_type: "Calculated"` and `expression_kind` is literal
- `query_group`: optional for M partitions to place the table query into a Power Query group
- `partition_name`: optional on `create_table`; when omitted, SemanticOps MCP, formerly MCP Engine, uses `<TableName>_Partition`
- `partition_name`: required on `create_partition`

### Create an M table (requires `columns`)

```json
{
  "operation": "create_table",
  "target": "DimCustomer",
  "spec": {
    "partition_name": "DimCustomer",
    "partition_type": "M",
    "expression": "let Source = ... in Source",
    "query_group": "Dimensions/Customer",
    "format_m": { "enabled": true, "consent": true },
    "columns": [
      { "name": "CustomerId", "data_type": "Int64" },
      { "name": "CustomerName", "data_type": "String" }
    ],
    "process": true,
    "refresh_type": "Full"
  }
}
```

`query_group` applies only to M partitions. To change/remove group later, use `manage_schema` `update_partition` with `spec.query_group` or `spec.clear_query_group=true`.

Desktop M writes require MCP-side verification. On June 2026+ Power BI Desktop, M/TOM changes are reflected immediately and silently with no Desktop action needed. Older Desktop versions may still show an external-changes banner; only if prompted, click `Discard` to accept MCP's external changes and do not click `Apply`. In all Desktop versions, call `manage_model_connection` with `operation="reload"` before continuing non-M operations.

**Processing options:**
- `process`: When `true`, refreshes the table after creation (loads data)
- `refresh_type`: Controls how data is refreshed:
  - `Full`: Clear existing data and reload completely
  - `DataOnly`: Refresh data without recalculating
  - `Calculate`: Recalculate calculated columns/measures only

**Create-table responses:**
- Default responses include `processed` and `row_count_known`; `row_count` is included when the engine can verify it.
- When `process=true`, warnings are surfaced if refresh/calculation succeeds only partially or post-create validation cannot confirm the expected table shape.
- If a calculated table saves but post-create validation still cannot confirm materialized user-facing columns, the tool returns an error payload with a recommended `Calculate` refresh next action.

### Create a calculated table (do NOT pass `columns`)

```json
{
  "operation": "create_table",
  "target": "TopCustomers",
  "spec": {
    "partition_name": "TopCustomers",
    "partition_type": "Calculated",
    "expression": "TOPN(100, SUMMARIZECOLUMNS('Customer'[CustomerId], \"Sales\", [Total Sales]), [Sales], DESC)",
    "format_dax": { "enabled": true },
    "process": true
  }
}
```

Notes for calculated tables:

- Calculated tables infer columns from the DAX expression; do not pass `columns`.
- `process=true` triggers a `Calculate` refresh after the table is saved.
- A successful save does not guarantee that the calculated table materialized usable user-facing columns; review returned warnings and verify row counts when debugging.
- If relationships or measures depend on a newly created calculated table, confirm the table columns are visible before creating the relationship.

## Updating Table Properties (`manage_schema` with `update_table`)

Common updates include:

- `new_name` to rename
- `is_hidden` to hide/show
- `description` to document
- date table marking (`mark_date_table` + `date_column`)

Example:

```json
{
  "operation": "update_table",
  "target": "DimDate",
  "spec": {
    "mark_date_table": true,
    "date_column": "Date"
  }
}
```

## Field Parameters (`manage_schema`)

Use `create_field_parameter` to create a Power BI field parameter table with the required calculated-table DAX and field-parameter metadata. Use `update_field_parameter` to replace the generated entries and reapply the metadata.

The calculated partition, complete `CalculatedTableColumn` schema, and field-parameter metadata are committed together before optional post-commit processing. Creation runs a Calculate refresh by default. `spec.process` is supported on `update_field_parameter` only; set it to `false` to skip that post-commit Calculate refresh.

Basic example:

```json
{
  "operation": "create_field_parameter",
  "target": "Parameter",
  "spec": {
    "entries": [
      { "label": "Customer", "field": "'Customer'[Customer]" },
      { "label": "Category", "field": "'Product'[Category]" },
      { "label": "Color", "field": "'Product'[Color]" },
      { "label": "Product", "field": "'Product'[Product]" }
    ]
  }
}
```

Generated DAX follows the Power BI field parameter tuple pattern:

```dax
{
    ("Customer", NAMEOF('Customer'[Customer]), 0),
    ("Category", NAMEOF('Product'[Category]), 1)
}
```

Supported entry fields:

- `label`: display text shown in the slicer.
- `field`: a DAX field reference such as `[Total Sales]`, `'Sales'[Total Sales]`, or `'Customer'[Customer]`.
- `ordinal`: optional integer. When omitted, ordinals are generated from entry order.
- `extra_values`: optional trailing tuple values. Use only with matching `extra_columns`.

Optional custom column names:

- `display_column`: defaults to the table name.
- `fields_column`: defaults to `<TableName> Fields`.
- `order_column`: defaults to `<TableName> Order`.

Optional custom trailing columns:

```json
{
  "operation": "create_field_parameter",
  "target": "Parameter",
  "spec": {
    "extra_columns": [
      { "name": "Group", "data_type": "String" },
      { "name": "Visible", "data_type": "Boolean", "is_hidden": true }
    ],
    "entries": [
      {
        "label": "Customer",
        "field": "'Customer'[Customer]",
        "extra_values": ["Dimensions", true]
      },
      {
        "label": "Revenue",
        "field": "[Revenue]",
        "extra_values": ["Measures", true]
      }
    ]
  }
}
```

Rules:

- Every entry must provide the same number of `extra_values` as `extra_columns`.
- Extra values support only DAX literals: string, number, boolean, and null.
- The tool applies the field-parameter metadata to the fields column and hides the fields/order columns.
- Delete a field parameter with `delete_table`; there is no separate delete operation.

## Column Properties (`manage_schema` with `update_column_properties`)

Key fields in spec:

- `new_name`, `data_type`, `description`, `is_hidden`, `format_string`, `data_category`, `display_folder`
- `source_column`: update the underlying source mapping for non-calculated (data) columns (useful after M output renames)
- `summarize_by`: `none|sum|min|max|average|count|distinct_count`
- `sort_by`: another column in the same table (empty string clears sort-by)

Example: change type and set sort-by:

```json
{
  "operation": "update_column_properties",
  "table": "DimDate",
  "target": "MonthName",
  "spec": {
    "data_type": "String",
    "sort_by": "MonthNumber",
    "summarize_by": "none"
  }
}
```

## Source Columns (`manage_schema`)

Create a non-calculated (data/source) column:

```json
{
  "operation": "create_column",
  "table": "Sales",
  "target": "NewColumn",
  "spec": {
    "data_type": "String",
    "source_column": "NewColumn"
  }
}
```

Delete a non-calculated (data/source) column:

```json
{
  "operation": "delete_column",
  "table": "Sales",
  "target": "NewColumn",
  "spec": { "force": false }
}
```

### Bulk `create_column` example

Top-level bulk controls apply to the whole request. Per-item `table`, `target`, and `spec` must be present inside each item.

```json
{
  "operation": "create_column",
  "transaction": true,
  "dry_run": false,
  "items": [
    {
      "table": "Employees",
      "target": "EmployeeID",
      "spec": { "data_type": "Int64", "source_column": "EmployeeID" }
    },
    {
      "table": "Employees",
      "target": "EmployeeName",
      "spec": { "data_type": "String", "source_column": "EmployeeName" }
    }
  ]
}
```

Do not rely on top-level `table` or `target` being copied into `items[]`.

## Calculated Columns (`manage_schema`)

Calculated-column writes default to `expression_context: "standard"` and automatically attempt a `Calculate` refresh for the owning table after save. Responses include `processed`, `expression_context`, and may include warnings plus a `next_action` refresh hint when the host defers or rejects inline calculation.

Use `expression_context: "user_context"` only when the column must evaluate at query time under the current user's security/culture context, for example with `USERCULTURE()`, `USERPRINCIPALNAME()`, `USEROBJECTID()`, `USERNAME()`, or `CUSTOMDATA()`. User-context calculated-column values are virtual/unmaterialized, but SemanticOps still attempts the same table `Calculate` refresh after save so the column is registered and queryable. If the model is below compatibility level 1705, retry with `allow_compatibility_upgrade: true` only after acknowledging that the compatibility-level upgrade is irreversible.

### Create

```json
{
  "operation": "create_calc_column",
  "table": "Sales",
  "target": "Margin",
  "spec": {
    "expression": "'Sales'[Revenue] - 'Sales'[Cost]",
    "summarize_by": "sum",
    "expression_context": "standard"
  }
}
```

Notes:
- `create_calc_column` infers the column data type from the DAX expression. `spec.data_type` is not supported.
- Allowed create fields are `expression`, `description`, `is_hidden`, `format_string`, `data_category`, `display_folder`, `summarize_by`, `sort_by`, `expression_context`, `allow_compatibility_upgrade`, and `format_dax`.
- `expression_context` accepts only `standard` or `user_context`. Omit it for the historical standard/materialized behavior.
- User-context calculated columns cannot be relationship endpoints and cannot be referenced directly or indirectly by standard calculated columns, calculated tables, or RLS expressions.
- For localization slicers, prefer pairing user-context display columns with stable sort/group-by columns so selections are not stored as translated strings.

### Update

```json
{
  "operation": "update_calc_column",
  "table": "Sales",
  "target": "Margin",
  "spec": {
    "expression": "'Sales'[Revenue] - 'Sales'[Cost]",
    "expression_context": "user_context"
  }
}
```

Changing `expression_context` changes materialization semantics. Use `standard` when a column participates in relationships or when refresh-time materialization is required for performance. Use `user_context` when the value depends on user identity/culture or when a simple row-level expression should remain virtual and query-time evaluated.

When changing to `user_context` on a model below compatibility level 1705, include `allow_compatibility_upgrade: true` only when the irreversible compatibility upgrade is intended.

### Delete

```json
{
  "operation": "delete_calc_column",
  "table": "Sales",
  "target": "Margin"
}
```

## Safe Schema Workflows

Recommended ordering for new modeled tables:

1. Create the table.
2. Review the create-table response for `processed`, `row_count_known`, `row_count`, and `warnings`.
3. If the table is calculated, confirm user-facing columns materialized before creating relationships.
4. Create relationships.
5. Run a `Calculate` refresh before validating measures that use `RELATED()` or newly created filter paths.

Recommended sequence for structural changes:

1. `list_model` for current state (tables/columns/relationships).
2. Use `dry_run` bulk where available for batches.
3. Apply table and column changes.
4. If you create calculated tables or new relationships, run or confirm a `Calculate` refresh before validating dependent measures.
5. Re-list to confirm columns and relationships are visible.
6. Validate key DAX queries via `run_query` with `operation: "execute"`.

When refactoring names, use `list_model` with `operation: "search"` to find dependent DAX/M expressions before renames.

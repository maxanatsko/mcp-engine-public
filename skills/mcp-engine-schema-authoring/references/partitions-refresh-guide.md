# Partitions & Refresh Guide (`manage_schema`)

This guide covers partition lifecycle operations and safe refresh workflows.

## Related Tools

- `manage_schema`: Create/update/delete/refresh partitions on existing tables (operations: `create_partition`, `update_partition`, `delete_partition`, `refresh_partition`)
- `manage_schema`: Create tables with an initial partition (`operation: "create_table"`)
- `list_model`: Inspect partitions (`operation: "list"`, `spec: { type: "partitions", include_expression: true }`)
- `refresh`: Plan and execute model/table/partition processing, check best-effort live status, inspect Desktop MCP-initiated history, and inspect Power BI Service refresh history/diagnostics

## Partition Types and Inputs

Partition types:

- `M`: Power Query (M)
- `Query`: SQL
- `Calculated`: DAX calculated partition/table
- `Entity`: Dataflow entity

Common fields in spec:

- `partition_type` (required for create; optional for update if changing)
- `expression` (required unless M + `expression_kind="named"`)
- `mode`: `Import|DirectQuery|Dual`
- `query_group`: assign an M partition to a Power Query group
- `clear_query_group` (update only): clear existing group assignment
- Optional formatting:
  - `format_m`: only applies to `partition_type: "M"` with literal expressions
  - `format_dax`: only applies to `partition_type: "Calculated"` with literal expressions

## Create a Partition

```json
{
  "operation": "create_partition",
  "table": "Sales",
  "target": "FY2025",
  "spec": {
    "partition_type": "M",
    "expression": "let Source = ... in Source",
    "mode": "Import",
    "process": true,
    "refresh_type": "Full"
  }
}
```

### Format M on Create

```json
{
  "operation": "create_partition",
  "table": "Sales",
  "target": "FY2025",
  "spec": {
    "partition_type": "M",
    "expression": "let Source = ... in Source",
    "format_m": { "enabled": true, "consent": true }
  }
}
```

## Refresh a Partition

```json
{
  "operation": "refresh_partition",
  "table": "Sales",
  "target": "FY2025",
  "spec": { "refresh_type": "DataOnly" }
}
```

**`refresh_type` values:**

| Value | Description | Use Case |
|-------|-------------|----------|
| `Full` | Clear all data and reload completely | Initial load, schema changes, data corrections |
| `DataOnly` | Refresh data without recalculating | Standard incremental data updates |
| `Calculate` | Recalculate only (no data reload) | After changing calculated columns/measures |

**Recommended patterns:**
- Use `DataOnly` for routine refreshes (fastest)
- Use `Full` after schema changes or data quality issues
- Use `Calculate` when only DAX expressions changed
- Unsupported values are rejected before execution. Valid values are `Full`, `DataOnly`, and `Calculate`.

## Update a Partition

You can update the partition expression or storage mode. For M partitions you can also use:

- `expression_kind="named"` + `named_expression`
- `query_group` to assign/change group (M partitions only)
- `clear_query_group=true` to remove current group assignment

Schema updates:

- `columns`: array of `{ name, data_type, source_column?, is_hidden?, format_string? }`
- `drop_extra_columns=true` removes columns not listed (use with extreme caution)
- `force=true` (only with `drop_extra_columns=true`) bypasses the best-effort DAX/role dependency scan, but structural blocks still apply (relationships, key columns, sort-by)

Notes:
- Prefer snake_case keys (`data_type`, `is_hidden`, `format_string`, `source_column`), but common camelCase variants are accepted (`dataType`, `isHidden`, `formatString`, `sourceColumn`).
- If `columns` is provided but no valid entries can be parsed (missing `name` or `data_type`), the tool returns an error instead of silently ignoring the input.

```json
{
  "operation": "update_partition",
  "table": "Sales",
  "target": "FY2025",
  "spec": {
    "expression": "let Source = ... in Source",
    "mode": "Import",
    "partition_type": "M",
    "format_m": { "enabled": true, "consent": true }
  }
}
```

## Assign / Clear Query Group

Assign group without redefining partition source:

```json
{
  "operation": "update_partition",
  "table": "Sales",
  "target": "FY2025",
  "spec": {
    "query_group": "Landing/Raw"
  }
}
```

Clear group:

```json
{
  "operation": "update_partition",
  "table": "Sales",
  "target": "FY2025",
  "spec": {
    "clear_query_group": true
  }
}
```

You can also assign a group during create:

```json
{
  "operation": "create_partition",
  "table": "Sales",
  "target": "FY2025",
  "spec": {
    "partition_type": "M",
    "expression": "let Source = ... in Source",
    "query_group": "Landing/Raw"
  }
}
```

Rules:
- `query_group` is only supported for M partitions.
- `query_group` and `clear_query_group=true` cannot be used together.
- `spec.query_group` cannot be empty. Use `clear_query_group=true` instead.
- If source is redefined to a non-M partition type, query group is removed.

## Safe Refresh Workflow

Recommended for production-like models:

1. `list_model` with `operation: "list"`, `spec: { type: "partitions" }` to understand current state.
2. Use `operation: "plan"` on `refresh` before model/table refreshes.
3. Apply updates.
4. `refresh_partition` one partition first.
5. Validate key visuals/queries before refreshing remaining partitions.

For `refresh` model scope, tables marked `exclude_from_model_refresh` are skipped by default for Desktop and Service/XMLA connections. Use `include_excluded: true` only when you explicitly intend to include them. Targeted table/partition refreshes do not apply this model-scope skip.

## Bulk Examples

Bulk refresh multiple partitions:

```json
{
  "operation": "refresh_partition",
  "transaction": false,
  "items": [
    { "table": "Sales", "target": "FY2024", "spec": { "refresh_type": "DataOnly" } },
    { "table": "Sales", "target": "FY2025", "spec": { "refresh_type": "DataOnly" } }
  ]
}
```

## Power BI Service Refresh Diagnostics

Use `operation="plan"` before Desktop-local refresh execution when you need to inspect target scope, affected tables/partitions, calculated partition overrides, excluded-table skips, and broad-refresh warnings:

```json
{
  "operation": "plan",
  "scope": "Model",
  "refresh_type": "Full"
}
```

`operation="status"` is a best-effort DMV probe. It returns `best_effort=true`, `status_source="discover_commands_dmv"`, confidence, and limitations; it is not durable history, a request/job ID, reliable progress tracking, or guaranteed in-progress detection.

For Desktop connections, `operation="history"` returns local MCP-initiated refresh history only. It includes explicit `refresh` executions and automatic post-write refreshes started by `manage_schema` (for example `process=true` or calculated-column materialization), with `trigger_source` identifying the initiator. Schema-triggered refresh history is best-effort/non-audit so schema writes keep their existing success/warning behavior if local history cannot be recorded. It does not include refreshes started in Power BI Desktop or other tools.

For Service connections, use the read-only `refresh` diagnostics operations when you need operational evidence rather than a new processing request. `details`, `diagnose`, and `reliability` are Power BI Service/XMLA diagnostics operations; Desktop connections should use `history` for local MCP-initiated refresh history and `status` for the best-effort DMV probe.

```json
{ "operation": "history", "limit": 20 }
```

```json
{ "operation": "diagnose" }
```

```json
{
  "operation": "reliability",
  "limit": 20,
  "max_age_minutes": 120,
  "expected_frequency_minutes": 60,
  "max_duration_minutes": 30
}
```

Use `operation="details"` with a `request_id` from history when you need per-attempt messages, objects, execution metrics, and sanitized service exception evidence. Service history/details/reliability require Power BI REST access.

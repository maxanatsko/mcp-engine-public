# Partitions & Refresh Guide (`manage_schema`)

This guide covers partition lifecycle operations and safe refresh workflows.

## Related Tools

- `manage_schema`: Create/update/delete/refresh partitions on existing tables (operations: `create_partition`, `update_partition`, `delete_partition`, `refresh_partition`)
- `manage_schema`: Create tables with an initial partition (`operation: "create_table"`)
- `list_model`: Inspect partitions (`operation: "list"`, `spec: { type: "partitions", include_expression: true }`)
- `manage_refresh`: Plan and execute model/table/partition processing, check best-effort live status, inspect Desktop MCP-initiated history, and inspect Power BI Service refresh history/diagnostics

## Partition Types and Inputs

Partition types:

- `M`: Power Query (M)
- `Query`: SQL
- `Calculated`: DAX calculated partition/table
- `Entity`: Dataflow entity

Common fields in spec:

- `partition_type` (required for create; optional for update; when omitted on update, MCP infers the existing partition type)
- `expression` (required unless M + `expression_kind="named"`)
- `mode`: `Import|DirectQuery|Dual`
- `query_group`: assign an M partition to a Power Query group
- `clear_query_group` (update only): clear existing group assignment
- `preserve_schema_mapping=true`: preserve the existing TOM column names and types while the caller accepts responsibility for M-output compatibility
- Optional formatting:
  - `format_m`: only applies to `partition_type: "M"` with literal expressions
  - `format_dax`: only applies to `partition_type: "Calculated"` with literal expressions

## Create a Partition

For M partitions on an existing table, provide `columns` matching the existing TOM table schema or set `preserve_schema_mapping=true`. The preservation flag is a caller assertion: MCP carries the existing TOM mapping forward but does not execute or inspect the M output.

```json
{
  "operation": "create_partition",
  "table": "Sales",
  "target": "FY2025",
  "spec": {
    "partition_type": "M",
    "expression": "let Source = ... in Source",
    "mode": "Import",
    "preserve_schema_mapping": true,
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
- For M expression updates, provide `columns` when the output shape may change.
- Use `preserve_schema_mapping=true` only when the existing TOM column names and types must be carried forward and the caller accepts responsibility for compatibility. The assertion does not cause MCP to execute or inspect the revised M output for schema compatibility. A separately requested refresh may execute the query, but it does not turn this assertion into runtime output-shape validation.
- Added, removed, renamed, or type-changed M output columns are not reconciled under mapping preservation. Supply explicit `columns` to change the TOM mapping.
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
    "preserve_schema_mapping": true,
    "format_m": { "enabled": true, "consent": true }
  }
}
```

M partition writes use the same caller-assertion contract for Power BI Desktop and XMLA connections. `preserve_schema_mapping=true` proves only that SemanticOps planned the write with the existing TOM mapping; it is not evidence that the revised M output is compatible. The response reports `schema_mapping.runtime_output_validation_performed=false` and identifies the carried-forward TOM mapping as its evidence boundary.

M partition writes are applied to the semantic model through the TOM endpoint. MCP reports the mutation outcome but cannot verify the open Power Query document or any Power BI Desktop external-change prompt. Depending on the operation and Desktop version, Desktop may show a refresh prompt or an Apply/Discard prompt. Refresh when data requires it. Do not treat `Apply` or `Discard` as a generic way to accept MCP changes; either action can replace one side's state. Inspect the model before dependent edits. MCP does not block unrelated writes after an M write, and no reload is required before continuing.

**Known limitation on Desktop connections:** M objects written through MCP may not reach Desktop's Power Query document, and pressing `Apply` in Desktop can discard them. Verify M changes in Desktop before relying on them.

## Assign / Clear Query Group

QueryGroup paths use `\` as the hierarchy separator; `/` remains a literal character. Use `manage_schema` for explicit group lifecycle, and create each parent before its children. Empty groups persist until `delete_query_group` is called.

Assign group without redefining partition source:

```json
{
  "operation": "update_partition",
  "table": "Sales",
  "target": "FY2025",
  "spec": {
    "query_group": "Landing\\Raw"
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
    "query_group": "Landing\\Raw"
  }
}
```

Rules:
- `query_group` is only supported for M partitions.
- `query_group` and `clear_query_group=true` cannot be used together.
- `spec.query_group` cannot be empty. Use `clear_query_group=true` instead.
- A direct assignment may create a missing target group when its parent exists. In a staged changeset, create the group in an earlier operation.
- If source is redefined to a non-M partition type, query group is removed.

## Safe Refresh Workflow

Recommended for production-like models:

1. `list_model` with `operation: "list"`, `spec: { type: "partitions" }` to understand current state.
2. Use `operation: "plan"` on `manage_refresh` before model/table refreshes.
3. Apply updates.
4. `refresh_partition` one partition first.
5. Validate key visuals/queries before refreshing remaining partitions.

For `manage_refresh` model scope, tables marked `exclude_from_model_refresh` are skipped by default for Desktop and Service/XMLA connections. Use `include_excluded: true` only when you explicitly intend to include them. Targeted table/partition refreshes do not apply this model-scope skip.

### Execution outcomes and recovery

Refresh execution validates the complete target plan before requesting first-write approval. After approval, SemanticOps revalidates the selected model and target identities, stages every target once, and makes one commit attempt.

Execution responses separate `command_execution` from `target_validation`. The command status reports whether the provider commit was applied, not applied, or unknown. After an applied table- or partition-scoped refresh, target validation reloads only the requested table/partition metadata and checks partition state, calculated-column state, and calculated-table column materialization. Model-scoped refreshes return `target_validation.status="not_checked"` rather than performing a whole-model validation scan.

An applied command can therefore return `refreshed=true` with overall failure when the target remains invalid or post-command readback is unavailable. `target_validation.status="invalid"` includes sanitized requested-object diagnostics; `unavailable` means the command was applied but connection, session, timeout, or metadata-readback evidence could not confirm the target. Neither result is safe for automatic replay.

- `NotApplied` means no refresh was committed and rollback was proven. Correct the validation, approval, or cancellation cause before retrying.
- `Applied` means the commit returned successfully. Do not replay the refresh if later finalization, history, notification, snapshot, audit, or response handling fails; inspect the model and recover the incomplete follow-up work.
- `Unknown` means SemanticOps could not prove whether the commit applied, such as after a lost commit response or failed discard. Do not retry automatically; inspect the model and refresh history first.

An explicit `manage_refresh` execution fails closed when Desktop history cannot be stored, but its outcome remains `Applied` after a successful commit. Schema-triggered history remains best-effort and surfaces a warning instead.

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

`operation="status"` is a best-effort DMV probe. It defaults to model scope when `scope` is omitted, so `{ "operation": "status" }` is valid. It returns `best_effort=true`, `status_source="discover_commands_dmv"`, confidence, and limitations; it is not durable history, a request/job ID, reliable progress tracking, or guaranteed in-progress detection.

For Desktop connections, `operation="history"` returns local MCP-initiated refresh history only. It includes explicit `manage_refresh` executions and automatic post-write refreshes started by `manage_schema` (for example `process=true` or calculated-column materialization), with `trigger_source` identifying the initiator. Schema-triggered refresh history is best-effort/non-audit so schema writes keep their existing success/warning behavior if local history cannot be recorded. It does not include refreshes started in Power BI Desktop or other tools.

For Service connections, use the read-only `manage_refresh` diagnostics operations when you need operational evidence rather than a new processing request. `details`, `diagnose`, and `reliability` are Power BI Service/XMLA diagnostics operations; Desktop connections should use `history` for local MCP-initiated refresh history and `status` for the best-effort DMV probe.

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

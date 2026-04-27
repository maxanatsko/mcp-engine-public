# Incremental Refresh Policy Guide (`manage_schema`) (Pro)

This guide explains how to configure and troubleshoot incremental refresh policies using `manage_schema` with `operation: "update_table"` and the `refresh_policy` object in spec.

## Related Tools

- `manage_schema`: Set or clear a refresh policy (`operation: "update_table"`)
- `manage_schema`: Refresh partitions after policy changes (`operation: "refresh_partition"`)
- `list_model`: Inspect partitions created by incremental refresh (`operation: "list"`, `spec: { type: "partitions" }`)
- `manage_semantic`: Manage Power Query parameters (`operation: "create_pq_parameter|update_pq_parameter"` - RangeStart/RangeEnd must exist; see ../../mcp-engine-semantic-authoring/references/power-query-parameters-guide.md)

## Prerequisites

- Your fact table is sourced from an M partition (Power Query) or you provide `refresh_policy.source_expression`.
- You have two DateTime Power Query parameters (named expressions) in the model:
  - `RangeStart`
  - `RangeEnd`
- The model compatibility level must support incremental refresh (the server enforces a minimum compatibility level and may require an upgrade).

## Refresh Policy Object Shape

`refresh_policy` is an object with fields:

- `mode`: `import` | `hybrid` (required)
- `incremental_granularity`: `day` | `month` | `quarter` | `year` (required)
- `incremental_periods`: integer (> 0, required)
- `rolling_window_granularity`: `day|month|quarter|year` (optional; must be paired with `rolling_window_periods`)
- `rolling_window_periods`: integer (> 0, optional; must be paired with `rolling_window_granularity`)
- `incremental_periods_offset`: integer (>= 0, optional; default 0)
- `source_expression`: string (optional if the first partition has an M expression to clone; otherwise required)
- `polling_expression`: string (optional)
- `range_start_parameter`: string (default `RangeStart`)
- `range_end_parameter`: string (default `RangeEnd`)
- `allow_compatibility_upgrade`: boolean (default false)

## Set an Incremental Refresh Policy

Example: keep 5 years monthly, refresh last 2 months, Import mode.

```json
{
  "operation": "update_table",
  "target": "FactSales",
  "spec": {
    "refresh_policy": {
      "mode": "import",
      "incremental_granularity": "month",
      "incremental_periods": 60,
      "rolling_window_granularity": "month",
      "rolling_window_periods": 2,
      "incremental_periods_offset": 0,
      "allow_compatibility_upgrade": true
    }
  }
}
```

## Hybrid (Real-time + Import) Mode

Hybrid mode allows a "hot" portion of data to remain in DirectQuery while older data is imported.

```json
{
  "operation": "update_table",
  "target": "FactSales",
  "spec": {
    "refresh_policy": {
      "mode": "hybrid",
      "incremental_granularity": "day",
      "incremental_periods": 730,
      "rolling_window_granularity": "day",
      "rolling_window_periods": 7,
      "allow_compatibility_upgrade": true
    }
  }
}
```

### Hybrid Mode vs Direct Lake (Fabric)

**Hybrid mode** (incremental refresh):
- Combines Import partitions with a DirectQuery partition for recent data
- Works with Power BI Premium and Pro workspaces
- Requires Power Query M expressions with RangeStart/RangeEnd parameters
- Data is copied into the semantic model

**Direct Lake** (Microsoft Fabric only):
- Reads directly from Delta tables in OneLake without data copy
- No incremental refresh policy needed (data is always current)
- Requires Fabric capacity and Lakehouse/Warehouse
- Different performance characteristics and DAX pattern support

If you're using Microsoft Fabric with Direct Lake mode, incremental refresh policies are not applicable. Direct Lake semantic models automatically reflect changes in the underlying Delta tables.

## Clearing a Refresh Policy

```json
{
  "operation": "update_table",
  "target": "FactSales",
  "spec": {
    "clear_refresh_policy": true
  }
}
```

Notes:

- `clear_refresh_policy` cannot be combined with `refresh_policy`.

## `RangeStart` / `RangeEnd` Troubleshooting

If you see errors like “Parameter 'RangeStart' not found”:

- Create DateTime parameters in Power Query named `RangeStart` and `RangeEnd`.
- Or set `range_start_parameter` / `range_end_parameter` to the names you used.

Example override:

```json
{
  "operation": "update_table",
  "target": "FactSales",
  "spec": {
    "refresh_policy": {
      "mode": "import",
      "incremental_granularity": "month",
      "incremental_periods": 60,
      "range_start_parameter": "MyRangeStart",
      "range_end_parameter": "MyRangeEnd"
    }
  }
}
```

## `source_expression` Troubleshooting

If the table’s first partition does not have an M expression to clone (or the engine cannot infer it), you must set:

- `refresh_policy.source_expression`

This should be an M expression that defines the table’s source query.

## Post-change Validation

After setting a policy:

1. `list_model` with `operation: "list"`, `spec: { type: "partitions", table: "FactSales" }` to verify partitions exist.
2. Refresh a partition or the table as needed:
   - `manage_schema` with `operation: "refresh_partition"` (`table` + partition `target`) for target partitions.
3. Validate visuals and key measures with `run_query` (`operation: "execute"`).

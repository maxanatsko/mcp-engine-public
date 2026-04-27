# Named Expressions Guide (`manage_semantic`)

Named expressions are model-level shared **Power Query expressions** (including parameters) that can be referenced by M partitions.

If you're specifically creating **Power Query parameters** (for example, `RangeStart` / `RangeEnd` for incremental refresh), prefer:

- `manage_semantic` with `operation: "create_pq_parameter|update_pq_parameter"` (see: `power-query-parameters-guide.md`)

## Related Tools

- `manage_semantic`: Create/update/read/delete named expressions (operations: `create_named_expression`, `update_named_expression`, `read_named_expression`, `delete_named_expression`)
- `manage_semantic`: Create/update/read/delete Power Query parameters (operations: `create_pq_parameter`, `update_pq_parameter`, `read_pq_parameter`, `delete_pq_parameter`)
- `manage_schema`: Create tables; can reference a named expression for M partitions (`operation: "create_table"`)
- `manage_schema`: Create/update partitions; can reference a named expression (`operation: "create_partition"`, `"update_partition"`)
- `list_model`: Inspect existing named expressions (`operation: "list"`, `spec: { type: "named_expressions", include_expression: true }`)
- `list_model`: Search M text across model (`operation: "search"`, `spec: { mode: "m", query: "RangeStart" }`)
  - Use `mode: "partition"` to search non-M partition expressions (SQL / Calculated / Entity)

## Create a Named Expression

```json
{
  "operation": "create_named_expression",
  "target": "SharedSource",
  "spec": {
    "expression": "let Source = Sql.Database(\"server\", \"db\") in Source",
    "description": "Shared SQL source"
  }
}
```

Optional:

- Use the exact field name `spec.expression_kind`, not `spec.kind`.
- `spec.expression_kind`: sets the underlying TOM `ExpressionKind` (omit to use the default for shared expressions).
- `list_model` and `read_named_expression` return `expression_kind`; `update_named_expression` preserves the existing kind unless you explicitly set `spec.expression_kind`.
- `spec.query_group`: assigns the named expression to a Power Query group.
- `spec.allow_compatibility_upgrade`: allows automatic model compatibility upgrade when required (query groups require compatibility level 1480+).
- `spec.format_m`: formats the M expression via the online M-formatter service before saving (requires consent).

### Format M on Create

```json
{
  "operation": "create_named_expression",
  "target": "SharedSource",
  "spec": {
    "expression": "let Source = Sql.Database(\"server\", \"db\") in Source",
    "format_m": { "enabled": true, "consent": true }
  }
}
```

## Read a Named Expression

```json
{
  "operation": "read_named_expression",
  "target": "SharedSource"
}
```

`read_named_expression` and `list_model` (`type: "named_expressions"`) return `query_group` when available.

## Update / Rename

```json
{
  "operation": "update_named_expression",
  "target": "SharedSource",
  "spec": {
    "new_name": "SharedSqlSource",
    "description": "Renamed for clarity"
  }
}
```

## Assign / Clear Query Group

Set group on create or update:

```json
{
  "operation": "create_named_expression",
  "target": "SharedSource",
  "spec": {
    "expression": "let Source = Sql.Database(\"server\", \"db\") in Source",
    "query_group": "Landing/Raw",
    "allow_compatibility_upgrade": true
  }
}
```

```json
{
  "operation": "update_named_expression",
  "target": "SharedSource",
  "spec": {
    "query_group": "Landing/Curated",
    "allow_compatibility_upgrade": true
  }
}
```

Clear existing group:

```json
{
  "operation": "update_named_expression",
  "target": "SharedSource",
  "spec": {
    "clear_query_group": true,
    "allow_compatibility_upgrade": true
  }
}
```

### Format M on Update

`format_m` is only valid for M expressions. If `spec.expression_kind` is explicitly set to a non-M value, the request is rejected.

When `spec.expression_kind` is omitted, the server may read the current `expression_kind` to decide if M formatting is allowed. If the current kind cannot be determined in your environment, set `spec.expression_kind: "M"` explicitly when using `format_m`.

```json
{
  "operation": "update_named_expression",
  "target": "SharedSource",
  "spec": {
    "expression": "let Source = Sql.Database(\"server\", \"db\") in Source",
    "expression_kind": "M",
    "format_m": { "enabled": true, "consent": true }
  }
}
```

## Use a Named Expression in an M Partition

When creating/updating an M partition, set:

- `expression_kind="named"`
- `named_expression="<name>"`

Example (`manage_schema` create partition):

```json
{
  "operation": "create_partition",
  "table": "Sales",
  "target": "Sales",
  "spec": {
    "partition_type": "M",
    "expression_kind": "named",
    "named_expression": "SharedSqlSource",
    "process": true
  }
}
```

## Common Pitfalls

- `expression_kind="named"` is only valid for `partition_type="M"`.
- `query_group` and `clear_query_group=true` are mutually exclusive in the same update request.
- `query_group` cannot be empty; to remove a group use `clear_query_group=true`.
- Assigning/clearing query groups on named expressions requires model compatibility level 1480+.
  - Use `allow_compatibility_upgrade=true` on create/update, or upgrade first via `manage_model_properties`.
- Renaming a named expression does not automatically update partitions that reference it; search via:
  - `list_model` with `operation: "search"`, `spec: { mode: "m", query: "SharedSqlSource" }` for M partitions/named expressions
  - `list_model` with `operation: "search"`, `spec: { mode: "partition", query: "SharedSqlSource" }` for non-M partition expressions

## Bulk Operations

```json
{
  "operation": "create_named_expression",
  "dry_run": true,
  "items": [
    { "target": "SharedSource", "spec": { "expression": "let Source = ... in Source" } },
    { "target": "SharedDimDate", "spec": { "expression": "let Date = ... in Date" } }
  ]
}
```

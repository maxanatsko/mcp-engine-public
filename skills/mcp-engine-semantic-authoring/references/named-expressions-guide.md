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
- `spec.format_m`: formats the M expression locally with `Pbi.PqParser` before saving; optional per-operation settings override runtime preferences.

### Format M on Create

```json
{
  "operation": "create_named_expression",
  "target": "SharedSource",
  "spec": {
    "expression": "let Source = Sql.Database(\"server\", \"db\") in Source",
    "format_m": { "enabled": true, "record_layout": "auto", "max_line_width": 100 }
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

QueryGroup paths use the Analysis Services separator `\`. `/` is a literal name character, not a hierarchy separator. Use `manage_schema` for explicit group lifecycle and create parents before children:

```json
{ "operation": "create_query_group", "target": "Landing" }
```

```json
{ "operation": "create_query_group", "target": "Landing\\Raw" }
```

Empty groups remain until `delete_query_group` is called. Rename or move a group with `update_query_group` and `spec.new_path`; deletion is rejected while the group still has children or members.

A direct single-object assignment may create its missing target group when its parent already exists. In a staged changeset, create the group in an earlier operation so final ownership and ordering are explicit.

Set group on create or update:

```json
{
  "operation": "create_named_expression",
  "target": "SharedSource",
  "spec": {
    "expression": "let Source = Sql.Database(\"server\", \"db\") in Source",
    "query_group": "Landing\\Raw",
    "allow_compatibility_upgrade": true
  }
}
```

```json
{
  "operation": "update_named_expression",
  "target": "SharedSource",
  "spec": {
    "query_group": "Landing\\Curated",
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
    "format_m": { "enabled": true, "record_layout": "auto", "max_line_width": 100 }
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
- M named-expression and M partition writes are applied to the semantic model through the TOM endpoint. MCP reports the mutation outcome but cannot verify the open Power Query document or any Power BI Desktop external-change prompt. Depending on the operation and Desktop version, Desktop may show a refresh prompt or an Apply/Discard prompt. Refresh when data requires it. Do not treat `Apply` or `Discard` as a generic way to accept MCP changes; either action can replace one side's state. Inspect the model before dependent edits. MCP does not block unrelated writes after an M write.
- **Known limitation on Desktop connections:** M objects written through MCP may not reach Desktop's Power Query document, and pressing `Apply` in Desktop can discard them. Verify M changes in Desktop before relying on them.
- `query_group` and `clear_query_group=true` are mutually exclusive in the same update request.
- `query_group` cannot be empty; to remove a group use `clear_query_group=true`.
- Assigning/clearing query groups on named expressions requires model compatibility level 1480+.
  - Use `allow_compatibility_upgrade=true` on create/update, or upgrade first via `manage_model_properties`.
- A named expression cannot be deleted or renamed directly while authoritative dependents remain. Exact partition references are reported by dependency analysis; the final live guard also fails closed on identifier tokens inside partition or named-expression M bodies and identifies the dependent object.
- The M-body guard is intentionally a bounded lexical safety heuristic, not an M parser or lineage engine. It does not resolve local shadowing, so ambiguous identifier tokens are rejected conservatively. Parser-backed replacement is tracked in [#1141](https://github.com/maxanatsko/mcp-engine/issues/1141).
- Use `manage_dependencies` with `operation="used_by"` and target type `named_expression` to inspect the exact dependency edge before refactoring.
- To rename a referenced expression, use one staged Desktop M changeset that renames the expression and rewrites every partition or named-expression reference. The final graph is validated before its single TOM commit, so an unsafe batch performs no model mutation.
- There is no `force` override for a referenced named-expression delete or rename.

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

Bulk named-expression writes use the same guarded model-write lifecycle as
single writes:

- `dry_run=true` parses every item and projects the resulting model state, but
  does not request approval or commit.
- `transaction=true` (the default) projects the complete batch, rejects
  duplicate/conflicting targets before approval, requests approval once, and
  commits the compatibility upgrade and all expression changes once.
- `transaction=false` gives each item its own write lifecycle. A known
  not-applied failure continues only when mandatory audit finalization is
  complete. An unknown commit outcome or incomplete audit finalization stops
  the batch because continuing or replaying may duplicate an applied change or
  bypass a fail-closed audit control.
- An update whose requested state already matches the named expression is a
  successful no-op. It does not request approval or commit.

Every write response includes `operation_outcome`. If
`model_mutation="unknown"` or an applied write reports a post-commit projection
failure, do not repeat the request. Follow the reported recovery action and
inspect the model and model-change history first.

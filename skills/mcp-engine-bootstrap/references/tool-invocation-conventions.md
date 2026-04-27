# Tool Invocation Conventions

This guide describes consistent JSON argument patterns across tools in this MCP server, including bulk operations and common field naming conventions.

## Related Tools and Resources

- `tool-invocation-conventions.md` (this guide)
- `troubleshooting-guide.md` (common mistakes + fixes)
- `../../mcp-engine-schema-authoring/references/column-and-table-authoring-guide.md` (schema operations)
- `../../mcp-engine-semantic-authoring/references/measure-authoring-guide.md` (semantic layer operations)

## Core Concepts

- Tools accept a single JSON object as `arguments`.
- Most write tools use an `operation` field to select the action (e.g., `create_measure`, `update_table`, `delete_role`).
- Many write tools support **bulk mode** via an `items` array (see below).
- Some tools support `include_details` to return richer, server-computed details (often token-heavy).
- `references/*.md` files in this skill are bundled markdown guides. Open them directly from the skill instead of treating them as web URLs.
- Query outputs may contain `null` for DAX `BLANK`; treat that as query semantics, not automatic evidence that an object is missing.

## Common Argument Keys

### `operation`

Used by most write tools to pick an action:

```json
{ "operation": "create_measure" }
```

### `action` + `resource` (manage_preferences)

The `manage_preferences` tool uses a resource+action pattern instead of `operation`:

```json
// List runtime settings with validation schemas
{ "action": "list", "resource": "setting" }

// Set a runtime preference
{ "action": "set", "resource": "setting", "id": "max_query_rows", "value": "500" }

// Create a naming rule
{ "action": "set", "resource": "rule", "id": "measure_naming", "type": "naming_rule", "value": "Use PascalCase" }

// Export all preference resources
{ "action": "export", "resource": "all" }
```

Resources: `rule`, `alias`, `setting`, `rendered`, `all`
Actions: `status`, `list`, `set`, `delete`, `reset`, `export`, `import`

`resource: "all"` supports `status`, `list`, `export`, and `reset` in write-capable mode. In read-only mode it supports `status`, `list`, and `export`.

### `table`, `target`, and object identifiers

Many tools use:

- `table`: table name (for table-scoped operations)
- `target`: object name (measure, role, relationship, etc.)

Example (`manage_semantic`):

```json
{
  "operation": "create_measure",
  "table": "Sales",
  "target": "Total Sales",
  "spec": { "expression": "SUM('Sales'[Amount])" }
}
```

### Naming conventions for keys

All tool parameter names use `snake_case` convention consistently across all tools.

Examples:

- `manage_semantic` uses `new_name`, `format_string`, `display_folder`, `is_hidden` in spec.
- `manage_schema` uses `new_name`, `format_string`, `display_folder`, `is_hidden` in spec.
- `list_model` uses `query`, `mode`, `types`, `case_sensitive`, `include_expression`, `include_fields`, `limit_per_type`, `preview_chars`, `format`, `limit`, `offset` in spec.

Use the MCP `tools/list` method (and each tool's `inputSchema`) to confirm parameter names.

Write tools generally validate `spec` keys and will return an error for unknown/unsupported properties (to avoid silently ignoring typos).

## Online Formatting Helpers (`format_dax`, `format_m`)

Some write operations support optional online formatting for expressions before saving.

### DAX formatting (`format_dax`)

Used by DAX-bearing operations (e.g., calculated partitions, measures, calc items):

```json
{
  "format_dax": { "enabled": true, "consent": true, "on_error": "fail" }
}
```

### M formatting (`format_m`)

Used by M-bearing operations (e.g., M partitions, named expressions):

```json
{
  "format_m": { "enabled": true, "consent": true, "on_error": "fail" }
}
```

Notes:

- `consent=true` is required when `enabled=true` because code is sent to an external formatting service.
- `on_error` controls behavior when formatting fails: `fail` (default) or `skip`.

## `list_model` Search Conventions

`list_model` search supports multiple surfaces depending on `mode`:

- `mode: "name"`: wildcard pattern matching (`*`, `?`) over object names.
- `mode: "text"`: substring search over metadata (name/description + opt-in fields) for all entity types; does not search expressions. Returns `match_type` (which field matched) and `match_context` (a value snippet).
- `mode: "dax"` / `mode: "m"` / `mode: "partition"`: substring search over expressions. Returns `match_type` (which expression field matched) and `match_context` (a snippet around the match).
- `mode: "any"`: union of `text` + `dax` + `m` + `partition` (flat format only).

For `mode: "text"` and `mode: "any"`, use `include_fields` to control breadth (always includes `core`; `translations` and `annotations` can be noisy/large).

`types` for `list_model` search must use canonical plural names such as `tables`, `columns`, and `measures`. Short singular aliases such as `table`, `column`, and `measure` are rejected.

`include_fields` also supports a macro:

- `include_fields: ["all"]`: expands to all implemented field groups (`core`, `folders`, `formatting`, `members`, `security`, `annotations`, `translations`).

Annotation search can be noisy due to system/tool-generated annotations. When `include_fields` includes `annotations`, you can optionally use:

- `exclude_system_annotations`: boolean (default `true` when annotations are enabled)
- `annotation_prefix_exclude`: string[] (e.g., `["PBI_", "TabularEditor_"]`)

## `list_model` List Filters

Some `list_model` `operation: "list"` types support lightweight filters:

- `visibility`: `"all"` (default), `"visible"`, `"hidden"` (applies to types that have `is_hidden`)
- `is_hidden`: boolean override for `visibility` (when present)
- Columns: `is_key`: boolean filter (key columns only)
- Relationships: `active_only` (alias: `is_active`) supports `true` (active only) / `false` (inactive only) / omitted (all)
- `include_system_artifacts`: default `false`, which filters internal Power BI artifacts from list results. Set `true` to return raw metadata.
  Filtered artifacts include `LocalDateTable_*`, `DateTableTemplate`, `DateTableTemplate_*`, `RowNumber-*`, `column_type="RowNumber"`, and list items owned by filtered system tables.

`operation: "list"` can also opt into richer outputs:

- `include_details`: adds additional metadata fields for certain types (e.g., columns, measures, calculated_columns)
- `include_annotations`: includes `annotations` arrays for types that support them (e.g., tables, measures, columns, calculated_columns, named_expressions)

## `list_model` Analyze and Info Requirements

- `operation: "analyze"` requires `target` (table name) and `spec.mode`.
- `spec.mode: "column_stats"` also requires `spec.column`.
- `spec.mode: "preview"` uses `max_rows`; legacy `rows` is still accepted.
- `operation: "info"` requires `spec.info_type`.
- Valid `info_type` values are `schema` and `data_sources`.

## `list_model` Search Visibility Filter

`operation: "search"` supports the same visibility controls for types where `is_hidden` is available:

- `visibility`: `"all"` (default), `"visible"`, `"hidden"`
- `is_hidden`: boolean override for `visibility`

## Pagination

List-style operations support consistent pagination via `limit` and `offset` parameters.

### Input Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | integer | 50-100 (tool-specific) | Max items to return (capped at tool's max, typically 1000) |
| `offset` | integer | 0 | Number of items to skip |

### Output Object

All paginated responses include a `pagination` object:

```json
{
  "count": 150,  // Back-compat: total count (same as pagination.total)
  "pagination": {
    "limit": 50,       // Requested limit (capped to max)
    "offset": 0,       // Requested offset
    "returned": 50,    // Actual items returned in this response
    "total": 150,      // Total items available
    "has_more": true,  // Whether more items exist beyond this page
    "next_offset": 50  // Offset for next page (only when has_more=true)
  },
  "items": [...]  // Actual data (key varies: models, measures, rules, etc.)
}
```

### Conventions

- **`count == pagination.total`**: The top-level `count` field equals `pagination.total` for backwards compatibility.
- **Deterministic ordering**: Results are sorted consistently (typically alphabetically by name) for stable pagination.
- **Silent capping**: If `limit` exceeds the tool's max, it's silently capped (no error).
- **Grouped search exception**: `list_model` search with `format: "grouped"` uses `limit_per_type` instead of global pagination.

### Tools with Pagination

| Tool | Default Limit | Max Limit |
|------|--------------|-----------|
| `list_model` (list, flat search) | 100 | 1000 |
| `manage_model_connection` | 50 | 1000 |
| `manage_preferences` | 50 | 1000 |
| `manage_policy` | 50 | 1000 |
| `manage_model_changes` | 50 | 1000 |
| `manage_audit` | 100 | 10000 |

## Bulk Operations (Most Write Tools)

Many tools that support bulk have these additional arguments:

- `items`: array of per-item argument objects (each item uses the same fields as the single call)
- `transaction`: boolean, default `true` (stop on first error)
- `dry_run`: boolean, default `false` (validate and return what would happen)
- `include_items`: boolean, default `false` (echo original items back in results)
- `include_details`: boolean, default `false` (include server details where supported)

Bulk precedence rules:

- Top-level bulk controls apply to the whole request.
- Each item must carry its own `table`, `target`, and `spec` when the single-item operation requires them.
- Top-level `table`, `target`, and `spec` values are not copied into `items[]`.
- If a bulk item omits a required identifier, the server validates that item as missing data instead of falling back to top-level object identifiers.

### Bulk example (multiple creates)

```json
{
  "operation": "create_measure",
  "transaction": true,
  "dry_run": false,
  "items": [
    { "table": "Sales", "target": "Total Sales", "spec": { "expression": "SUM('Sales'[Amount])" } },
    { "table": "Sales", "target": "Total Cost", "spec": { "expression": "SUM('Sales'[Cost])" } }
  ]
}
```

### Bulk example (`manage_schema` create_column)

```json
{
  "operation": "create_column",
  "transaction": true,
  "items": [
    {
      "table": "Employees",
      "target": "EmployeeID",
      "spec": { "data_type": "Int64", "source_column": "EmployeeID" }
    }
  ]
}
```

If an item omits `target`, a top-level `target` will not be used as a fallback.

### Bulk response shape

Bulk responses return an envelope:

```json
{
  "success": true,
  "transactional": true,
  "dry_run": false,
  "results": [
    { "ok": true, "result": { "status": "ok", "message": "..." } }
  ],
  "errors": []
}
```

Notes:

- `success` is **per-batch** success (`errors.length == 0`).
- Some tools return a top-level tool success even when the bulk envelope has `success=false`; always check the envelope.

### `dry_run` behavior

When `dry_run=true`, tools typically return `results[].detail` entries describing what would happen, and do not apply changes.

## Bulk Exception: `manage_semantic` with Calc Groups

`manage_semantic` with `create_calc_group` already uses `items` for calculation items, so **bulk uses `bulk_items`**:

```json
{
  "operation": "update_calc_group",
  "transaction": true,
  "dry_run": true,
  "bulk_items": [
    { "target": "Time Intelligence", "spec": { "items_upsert": [{ "name": "YOY", "expression": "/* DAX */" }] } }
  ]
}
```

## Annotations Pattern (Where Supported)

Some tools support annotations in spec:

- `annotations_upsert`: `[{ "name": "Key", "value": "Value" }]`
- `annotations_delete`: `["KeyToRemove"]`

Example (`manage_semantic`):

```json
{
  "operation": "update_measure",
  "table": "Sales",
  "target": "Total Sales",
  "spec": { "annotations_upsert": [{ "name": "owner", "value": "finance" }] }
}
```

## Include Details

Some tools support `include_details` to include richer server-computed details. Prefer:

- `include_details=false` by default (token-efficient)
- enable only when you need to inspect what was created/updated

## Discovery Before Edits (Recommended)

Before write operations, discover current state:

- `list_model` for structured listing by type (`operation: "list"`)
- `list_model` for searching names or expressions (`operation: "search"`)
- `run_query` to validate DAX results quickly after changes (`operation: "execute"`)

## Output Discipline (stdout/stderr)

- Tool results are returned as structured JSON payloads.
- Server logs are written to stderr; stdout is reserved for MCP protocol messages. Treat stderr as diagnostic only.

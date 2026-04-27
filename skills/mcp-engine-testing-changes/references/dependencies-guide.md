# Dependency & Impact Analysis Guide (Pro)

`manage_dependencies` is a Pro tool for answering:
- “What uses this measure/column/M expression?”
- “What will be impacted if I change X?”

It combines expression-based matching with optional structural and metadata-derived edges, supports transitive traversal, and can export graphs.

## Quick Start

### 1) Measure impact (direct)
Call `manage_dependencies`:
```json
{
  "operation": "used_by",
  "spec": {
    "target": { "type": "measure", "table": "Sales", "name": "Total Sales" },
    "scope": "dax",
    "depth": 1
  }
}
```

### 2) Column impact (transitive)
```json
{
  "operation": "used_by",
  "spec": {
    "target": { "type": "column", "table": "Sales", "name": "Amount" },
    "scope": "any",
    "depth": 2
  }
}
```

### 3) Summary rendering (Mermaid mindmap)
```json
{
  "operation": "summary",
  "spec": {
    "target": { "type": "measure", "table": "Sales", "name": "Total Sales" },
    "scope": "dax",
    "depth": 2,
    "render": "mermaid_mindmap"
  }
}
```

### 4) Graph export (Mermaid / DOT / CSV)
```json
{
  "operation": "graph",
  "spec": {
    "target": { "type": "measure", "table": "Sales", "name": "Total Sales" },
    "scope": "dax",
    "depth": 2,
    "render": "mermaid_flowchart"
  }
}
```

## Operations

### `operation="used_by"`
Find downstream dependents (impact analysis).

Common spec fields:
- `target` (required): `{ id?, type, name?, table?, calculation_group?, from_table?, from_column?, to_table?, to_column? }`
  - `id`: optional canonical id (e.g., copied from `graph.nodes[].id`)
  - `type`: `table|measure|column|calculated_column|hierarchy|relationship|calculation_group|calculation_item|kpi|partition|named_expression|udf|role|perspective|culture|calendar|model_property`
  - `table` is required for: `measure|column|calculated_column|partition|hierarchy|calendar|kpi`
  - `calculation_group` is required for `calculation_item`
  - `relationship` can be targeted via `id`, or via `from_table/from_column/to_table/to_column`
- `scope` (expressions): `dax|m|partition|any` (default `any`)
- `depth`: transitive traversal depth (default `1`, max `4`)
- `include_structural`: include exact (non-expression) edges (default `true`)
- `include_metadata`: include metadata field matches as dependency edges (default `false`)
- `include_fields` (metadata): `core|folders|formatting|members|security|annotations|translations|all` (default `["core"]`; `core` is always included)
- `exclude_self_references`: when `include_metadata=true` and `target.type="table"`, omit metadata hits belonging to that table (default `false`)
- Annotation noise controls (only apply when `include_fields` includes `annotations`):
  - `exclude_system_annotations`: `true|false` (default `true`)
  - `annotation_prefix_exclude`: string array of annotation key prefixes to exclude (optional)
  - Default system filters include annotation names starting with `PBI_` / `TabularEditor_`, plus `SummarizationSetBy` and `PBI_FormatHint`.
- `expand_metadata`: include metadata edges in traversal (default `false`; root-only)
- `types`: restrict dependent types returned (optional)
- `case_sensitive`: `true|false` (default `false`)
- `limit_per_type`: cap items returned per bucket (default comes from server config)
- Limits: `max_nodes`, `max_edges`, `max_expansions_per_node`
- `confidence_min`: `exact|high|any` (default `high`)

### Structural edges (exact)
With `include_structural=true`, `manage_dependencies` can return non-expression dependents like:
- Relationship endpoints (column/table used in a relationship)
- Sort-by dependencies (column used as a sort-by)
- Hierarchy membership (column used in a hierarchy level)
- Perspective membership
- Role object permissions (OLS)
- KPI base measure
- Calendar members (auto date/time metadata)

These appear in `graph.edges[]` with `mode="structural"` and a `kind` describing the relationship (e.g., `relationship_column`, `sort_by`).

### Metadata edges (mentions)
With `include_metadata=true`, matches in metadata fields (like `description`, `display_folder`, annotations, translations) are returned as `mode="text"` edges with `kind="mentions"`.

Name (and `qualified_name`) fields use word-boundary matching (e.g., targeting `Sales` matches `US Sales Only` but not `SalesThreshold`).

### `operation="summary"`
Same analysis as `used_by`, plus optional rendering:
- `render`: `none|mermaid_mindmap|markdown_tree`

### `operation="graph"`
Returns `graph.nodes[]` and `graph.edges[]` (directed “dependent -> dependency”), plus optional rendering:
- `render`: `none|mermaid_flowchart|dot|csv_edges|csv_nodes`

CSV notes:
- `csv_edges` includes extra columns for triage (`field_group`, `pattern`, `detail`, `match_context`).
- `csv_nodes` includes extra columns (`calculation_group`, `role`, `partition_type`).

## Accuracy & Limitations

This tool is fast and practical, but not a full parser:
- It primarily matches via substring patterns in expressions (DAX/M/partition SQL).
- For column targets, unqualified `[Column]` references are matched within table-scoped contexts (RLS filters, calculated columns).
- It may produce false positives when names are ambiguous (e.g., short measure names).
- Some targets (like `partition` and `role`) are inherently more heuristic.
- Metadata edges (`include_metadata=true`) are intentionally fuzzy and can be noisy; keep `include_fields` narrow unless you need broader coverage.

## Performance & Caching

To keep repeated calls fast (especially during depth traversal), the server caches a per-model dependency corpus (expression index) for a short time and invalidates it on model reload/switch.

Metadata note: `expand_metadata=true` can be expensive because it scans metadata for each expanded node; keep `depth` low and `include_fields` narrow.

Memory note: the corpus stores full expression texts (DAX/M/partition) for the indexed objects. Very large models can use noticeable memory while cached; reduce `...MAX_ENTRIES`, reduce TTL, or disable caching if needed.

Optional env overrides:
- `MCP_ENGINE_DEPENDENCY_CORPUS_CACHE_ENABLED` (`true|false`)
- `MCP_ENGINE_DEPENDENCY_CORPUS_CACHE_TTL_SECONDS` (default `30`; set to `0` to disable caching while leaving it enabled)
- `MCP_ENGINE_DEPENDENCY_CORPUS_CACHE_MAX_ENTRIES` (default `8`)
- `MCP_ENGINE_DEPENDENCY_CORPUS_CACHE_BUILD_TIMEOUT_SECONDS` (default `300`)

If you need a lower-level primitive, `list_model operation="search"` can be used directly with `mode="dax"|"m"|"partition"|"any"`.

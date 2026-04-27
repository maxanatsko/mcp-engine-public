# Unit Testing Guide (`manage_tests`) (Pro)

This guide explains how to use the Pro Unit Testing feature to create, run, and manage tests for Power BI semantic models.

## Related Tools

- `manage_tests` (Pro): Create/run/export tests (13 operations)
- `list_model`: Inspect model schema for test targets (`operation: "list"`)
- `run_query`: Validate DAX queries before using in tests (`operation: "execute"`)
- `manage_model_connection`: Connect to a model before running tests

## Test Types

| Type | Purpose |
|------|---------|
| `measure_assertion` | Assert measure values with tolerance |
| `dax_assertion` | Assert manual DAX query results with scalar comparisons |
| `rls_validation` | Validate RLS rules filter data correctly |
| `ols_validation` | Validate OLS table/column access per role or identity |
| `performance_budget` | Assert query performance thresholds |
| `regression_snapshot` | Detect data drift via baseline comparison |
| `referential_integrity` | Detect orphan keys in relationships |
| `metadata_compliance` | Lint model metadata (descriptions, naming) |

## Operations Overview

| Operation | Requires Model | Description |
|-----------|----------------|-------------|
| `capabilities` | Yes | Return environment capabilities |
| `list` | No | List tests with filtering |
| `get` | No | Get single test definition |
| `put` | No | Create/update test |
| `delete` | No | Remove test(s) |
| `run` | Yes | Execute tests |
| `runs_list` | Yes | List persisted run history for the connected model |
| `export` | No | Format results (json/junit/markdown/html) |
| `snapshot` | Yes | Capture/list/delete baselines |
| `validate` | No | Validate test definitions |
| `packs_list` | No | List available built-in packs |
| `packs_apply` | Yes | Generate tests from pack |
| `suggest` | Yes | LLM-assisted test generation (planned) |

---

## Stable model identity (survives renames/moves)

When a model is connected, `manage_tests` associates tests, baselines, and run history with a **stable model id** stored in the model metadata as a TOM annotation:

- Canonical annotation name: `McpEngine_StableModelId`
- Legacy annotation name: `PowerBIMcp_StableModelId` (read for compatibility and backfilled to the canonical name during confirmed write-capable flows)
- Value: GUID string
- Stored `model_id` in the test database: `stable:{guid}`

This keeps unit tests tied to the same model even if the PBIX is renamed/moved (Desktop) or if the dataset display name changes (Service).

Note: the first persisted testing action for a model may write the `McpEngine_StableModelId` annotation into model metadata. That is a model change, and you need to save the PBIX if you want the annotation to persist.

The Tauri test runner uses `runs_list` to hydrate persisted run history for the currently connected model. Run history remains server-side only; the UI does not keep a separate local history store.

## Quick Start

### 1. Check capabilities

```json
{ "operation": "capabilities" }
```

### 2. Apply a built-in pack

```json
{ "operation": "packs_apply", "spec": { "pack_id": "metadata-quality" } }
```

### 3. Run all tests

```json
{ "operation": "run" }
```

### 4. Export results

```json
{ "operation": "export", "spec": { "format": "markdown" } }
```

---

## Creating Tests

### Bulk create/delete (items)

`put` and `delete` support bulk via an `items` array (similar to other `manage_*` tools):

```json
{
  "operation": "put",
  "items": [
    {
      "id": "test-1",
      "name": "Example test 1",
      "type": "measure_assertion",
      "meta": { "tags": ["smoke"] },
      "spec": { "measure": "Sales[Total Sales]" },
      "assert": { "kind": "scalar", "op": "gte", "expected": { "type": "number", "value": 0 } }
    },
    {
      "id": "test-2",
      "name": "Example test 2",
      "type": "measure_assertion",
      "meta": { "tags": ["critical"] },
      "spec": { "measure": "Sales[Total Sales]" },
      "assert": { "kind": "scalar", "op": "lte", "expected": { "type": "number", "value": 1000000 } }
    }
  ],
  "transaction": true,
  "dry_run": false,
  "include_items": false
}
```

Note: `transaction: true` is fail-fast bulk behavior. The handler stops on the first item error, but it does not wrap the batch in a single atomic rollback transaction.

Bulk delete:

```json
{
  "operation": "delete",
  "items": [
    { "target": "test-1" },
    { "target": "test-2" }
  ]
}
```

### measure_assertion

Assert a measure equals an expected value:

```json
{
  "operation": "put",
  "spec": {
    "id": "total-sales-2024",
    "name": "Total Sales 2024",
    "type": "measure_assertion",
    "meta": { "tags": ["smoke"], "severity": "error" },
    "context": {
      "filters": [
        { "table": "Date", "column": "Year", "op": "eq", "values": [{ "type": "number", "value": 2024 }] }
      ]
    },
    "spec": { "measure": "Sales[Total Sales]" },
    "assert": {
      "kind": "scalar",
      "op": "equals",
      "expected": { "type": "number", "value": 1250000 },
      "tolerance": { "rel": 0.01 }
    }
  }
}
```

Assertion operators: `equals`, `greaterThan`, `lessThan`, `inRange`, `notBlank`

Important:
- `spec` is required for `measure_assertion` and only accepts `measure`.
- Do not pass `spec.table`; `measure_assertion` does not have a separate table field.
- Canonical `spec.measure` forms are `[Total Sales]` or `Sales[Total Sales]`. Bare human-readable measure names may work, but bracketed measure references are the safest form.
- Put assertions in the top-level `assert` block, not inside `spec`.
- Use `context.filters` for filtering. `spec.filter` is not supported.
- Canonical `assert.expected` shape is a TypedValue object like `{ "type": "number", "value": 1250000 }`.
- Raw scalar shorthand is also accepted for common handcrafted tests, for example `"expected": 1250000`, `"expected": "OK"`, or `"expected": true`. The server normalizes these to the canonical TypedValue shape when saving the test.
- Canonical tolerance shape is `{"abs": <number>}` and/or `{"rel": <number>}`. Plain numeric tolerance may be normalized on input, but stored tests use `abs`/`rel`.

### dax_assertion

Assert the result of a manual DAX query:

```json
{
  "operation": "put",
  "spec": {
    "id": "kpi-visual-2025",
    "name": "KPI visual query returns expected value",
    "type": "dax_assertion",
    "spec": {
      "query": "DEFINE MEASURE Sales[__KPI] = CALCULATE([Total Sales], 'Date'[Year] = 2025)\nEVALUATE ROW(\"Value\", [__KPI])"
    },
    "assert": {
      "kind": "scalar_query",
      "op": "equals",
      "path": ["Value"],
      "expected": { "type": "number", "value": 1250000 },
      "tolerance": { "rel": 0.01 }
    }
  }
}
```

Important:
- `spec.query` runs verbatim and may include full `DEFINE` / `EVALUATE` queries copied from Performance Analyzer.
- `context.filters` is not supported for `dax_assertion`; encode filter context directly in the query text.
- If the query returns multiple columns, set `assert.path` to the first-row column name to compare.
- v1 is scalar-only: without `assert.path`, the query must return a single-row, single-column result.
- `assert.expected` uses the same canonical TypedValue shape as `measure_assertion`; raw number/string/boolean literals are accepted and normalized on save.

### rls_validation

Test that RLS roles filter data correctly:

```json
{
  "operation": "put",
  "spec": {
    "id": "rls-sales-role",
    "name": "Sales role sees limited data",
    "type": "rls_validation",
    "requires": ["rls_roles"],
    "spec": {
      "query": "EVALUATE ROW(\"rows\", COUNTROWS(Sales))",
      "principals": [
        { "role": "SalesRole" },
        { "role": "ManagerRole" }
      ],
      "mode": "matrix"
    },
    "assert": {
      "kind": "table",
      "op": "row",
      "path": ["rows"],
      "per_principal": { "max": 10000 }
    }
  }
}
```

Note: `rls_effective_user` capability (identity impersonation) requires Power BI Service XMLA endpoint.
The impersonated identity must also have both Read and Build permission on the semantic model. `Build` enables XMLA/query access; it does not bypass RLS for Viewer users.
Current scope:
- Legacy row-bounds assertions remain supported via `kind: "table"`, `op: "row"`, optional `path`, and `per_principal` min/max bounds.
- Canonical direct assertions use `kind: "rls"` with one expectation per principal.
- Supported canonical operators are `query_succeeds`, `query_fails`, `is_blank`, `is_not_blank`, `equals`, `row_count_equals`, `row_count_min`, `row_count_max`, and `row_count_between`.
- `equals`, `is_blank`, and `is_not_blank` require `assert.path`.
- `query_fails` may include `error_contains`.
- Canonical principals shape is `[{ "role": "RoleName", "identity": "user@contoso.com"? }]`.

### ols_validation

Test direct table and column access enforced by Object-Level Security:

```json
{
  "operation": "put",
  "spec": {
    "id": "ols-sales-access",
    "name": "Sales role cannot read Customer SSN",
    "type": "ols_validation",
    "requires": ["rls_roles"],
    "spec": {
      "principals": [
        { "role": "SalesRole" },
        { "role": "ManagerRole", "identity": "manager@contoso.com" }
      ],
      "targets": [
        { "object_type": "table", "table": "Sales" },
        { "object_type": "column", "table": "Customer", "column": "SSN" }
      ],
      "mode": "matrix"
    },
    "assert": {
      "kind": "ols",
      "expectations": [
        {
          "role": "SalesRole",
          "object_type": "table",
          "table": "Sales",
          "op": "access_allowed"
        },
        {
          "role": "SalesRole",
          "object_type": "column",
          "table": "Customer",
          "column": "SSN",
          "op": "access_denied",
          "error_contains": "permission"
        },
        {
          "role": "ManagerRole",
          "identity": "manager@contoso.com",
          "object_type": "table",
          "table": "Sales",
          "op": "access_allowed"
        },
        {
          "role": "ManagerRole",
          "identity": "manager@contoso.com",
          "object_type": "column",
          "table": "Customer",
          "column": "SSN",
          "op": "access_allowed"
        }
      ]
    }
  }
}
```

Current scope:
- OLS targets support only `table` and `column`.
- `column` is required only when `object_type` is `column`.
- Assertions require exactly one expectation per principal-target pair.
- Supported operators are `access_allowed` and `access_denied`.
- `error_contains` is allowed only for `access_denied`.
- OLS tests reuse `rls_roles` capability for role impersonation and additionally require `rls_effective_user` when any principal includes `identity`.

### performance_budget

Assert query executes within time budget:

```json
{
  "operation": "put",
  "spec": {
    "id": "yoy-performance",
    "name": "YoY measure under 200ms",
    "type": "performance_budget",
    "spec": {
      "query": "EVALUATE { [Sales YoY %] }",
      "runs": { "cold": 1, "warm": 3 }
    },
    "assert": {
      "kind": "performance",
      "total_ms": { "max": 500 },
      "warm_ms": { "p95_max": 200 }
    }
  }
}
```

### regression_snapshot

Detect data drift by comparing to baseline:

```json
{
  "operation": "put",
  "spec": {
    "id": "monthly-sales-snapshot",
    "name": "Monthly sales by category",
    "type": "regression_snapshot",
    "spec": {
      "query": "EVALUATE SUMMARIZE(Sales, Product[Category], \"Total\", [Total Sales])",
      "snapshot": { "mode": "hash", "columns": ["Category", "Total"], "row_cap": 5000 }
    },
    "assert": {
      "kind": "snapshot",
      "baseline_id": "baseline:model:monthly-sales-snapshot@2025-01-01T00:00:00Z",
      "op": "equals"
    }
  }
}
```

Snapshot modes: `hash` (deterministic hash), `aggregate` (row count + checksums), `topn` (first N rows), `full` (all data)

For `DaxQueryResult` snapshots:
- `hash` and `aggregate` operate on canonicalized row data, not transport-specific CLR numeric types.
- Whole numbers are normalized so Desktop and Service/XMLA transports hash and compare consistently when the result values are semantically equal.
- `topn` and `full` preserve the same envelope shape, but row cell values are canonicalized before comparison.

### referential_integrity

Detect orphan keys:

```json
{
  "operation": "put",
  "spec": {
    "id": "orphan-products",
    "name": "No orphan product keys",
    "type": "referential_integrity",
    "spec": {
      "checks": [
        { "type": "orphan_keys", "from": "Sales[ProductId]", "to": "Product[ProductId]" }
      ]
    },
    "assert": {
      "kind": "integrity",
      "op": "equals",
      "expected": { "type": "number", "value": 0 }
    }
  }
}
```

Important:
- `spec.checks` is required for both `put` and `validate`.
- Each `checks[]` entry must include `type`, `from`, and `to`.
- The supported check type is currently `orphan_keys`.

### metadata_compliance

Lint model metadata:

```json
{
  "operation": "put",
  "spec": {
    "id": "metadata-quality",
    "name": "All measures have descriptions",
    "type": "metadata_compliance",
    "spec": {
      "rules": [
        { "id": "measure_requires_description" },
        { "id": "naming_convention", "scope": { "entity_types": ["measures"] } },
        { "id": "column_has_format_string", "scope": { "include_tables": ["Sales", "Products"] } },
        { "id": "hidden_columns_in_relationships" }
      ]
    },
    "assert": { "kind": "lint", "max_issues": 0 }
  }
}
```

Available rules:
- `measure_requires_description`
- `table_requires_description`
- `column_requires_description`
- `naming_convention` (PascalCase)
- `column_has_format_string`
- `measure_has_format_or_dynamic_format_string`
- `no_calculated_columns`
- `hidden_columns_in_relationships`
- `relationship_not_bidirectional`
- `relationship_not_many_to_many`
- `key_columns_hidden`
- `visible_measures_have_display_folder`
- `visible_columns_have_display_folder`
- `date_table_marked`
  - no params: assert that the model has some marked date table
  - with `params.table`: assert that a specific table is the marked date table

Discovery and aliases:
- `operation: "capabilities"` returns `metadata_compliance.rules` with the built-in rule catalog, `metadata_compliance.entity_types`, canonical `assertion_list_field: "rules"`, and accepted aliases.
- `spec.rules` is the canonical assertion list field.
- `spec.checks` is accepted as an alias for `metadata_compliance` and is normalized to `spec.rules` when tests are saved.
- Shorthand rules such as `{ "id": "local-id", "type": "require_format_string", "object_types": ["measure"] }` are accepted and normalized to the matching built-in rule id and scope.

Example:

```json
{
  "id": "date_table_marked",
  "params": {
    "table": "Calendar"
  }
}
```

#### Rule Scope (optional)

Each rule accepts an optional `scope` object to restrict which entities it checks. When omitted, the rule checks all applicable entities.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `entity_types` | `string[]` | all applicable | Object kinds to check: `"tables"`, `"measures"`, `"columns"`, `"relationships"` |
| `include_tables` | `string[]` | all tables | Only check entities in these tables |
| `exclude_tables` | `string[]` | none | Skip entities in these tables (takes precedence over include) |

All table name comparisons are case-insensitive. Both `entity_types` and table filters apply cumulatively (AND logic).

```json
{
  "id": "naming_convention",
  "scope": {
    "entity_types": ["measures"],
    "include_tables": ["Sales"]
  }
}
```

---

## Built-in Packs

### List available packs

```json
{ "operation": "packs_list" }
```

### Apply a pack

```json
{ "operation": "packs_apply", "spec": { "pack_id": "metadata-quality" } }
```

Use canonical `spec.pack_id`. The alias `spec.pack` may be normalized on input, but prompts and saved examples should use `pack_id`.

| Pack ID | Generated Tests |
|---------|-----------------|
| `metadata-quality` | Documentation, naming, formatting, display folders, and relationship metadata best practices |
| `documentation-baseline` | Table, column, and measure description checks |
| `presentation-hygiene` | Column/measure format strings and display-folder checks |
| `referential-integrity` | Orphan key checks for each relationship plus relationship governance checks |
| `relationship-governance` | Relationship directionality, cardinality, and key-column visibility checks |
| `time-intelligence` | Marked date-table check plus heuristic continuous date range validation |

Notes:
- `time-intelligence` currently uses heuristics. Its continuity test assumes a conventionally named date table with a `[Date]` column and does not attempt deeper date-table inference.
- Packs for `rls_validation`, `ols_validation`, `performance_budget`, and `regression_snapshot` are intentionally not built in because they require model-specific expectations, budgets, principals, or baseline decisions.

---

## Running Tests

### Run all tests

```json
{ "operation": "run" }
```

### Run by tags

```json
{ "operation": "run", "spec": { "tags": ["smoke"] } }
```

### Run by type

```json
{ "operation": "run", "spec": { "types": ["measure_assertion", "dax_assertion", "performance_budget"] } }
```

### Run specific tests

```json
{ "operation": "run", "spec": { "test_ids": ["total-sales-2024", "yoy-performance"] } }
```

Use canonical `spec.test_ids`. The alias `spec.ids` may be normalized on input, but prompts and saved examples should use `test_ids`.

### Stop on first failure

```json
{ "operation": "run", "spec": { "stop_on_first_failure": true } }
```

### Diagnostics levels

```json
{ "operation": "run", "spec": { "diagnostics_level": "full" } }
```

Levels:
- `none`: Only pass/fail status
- `safe`: Which assertion failed (no values)
- `full`: Include actual/expected values

Note: When numeric/PII masking is enabled, `full` is blocked and `safe` omits values.

Additional notes:
- For `metadata_compliance`, `safe` includes a short sample of failing objects to help pinpoint what needs fixing.
- Markdown/HTML/JUnit exports include assertion messages (when present) to preserve detailed failure context in reports.

---

## Exporting Results

### Export as JSON

```json
{ "operation": "export", "spec": { "format": "json" } }
```

### Export as JUnit XML (for CI/CD)

```json
{ "operation": "export", "spec": { "format": "junit", "save_to_path": "./test-results/junit.xml" } }
```

### Export as Markdown

```json
{ "operation": "export", "spec": { "format": "markdown" } }
```

### Export as HTML

```json
{ "operation": "export", "spec": { "format": "html", "save_to_path": "./test-results/report.html" } }
```

Notes:
- `save_to_path` is optional for any export format. When set, the server writes the export to disk and returns `saved_to` with `content_omitted=true`.
- `saved_to` is a server-local absolute path. Remote MCP clients may not be able to open that path directly.
- Relative `save_to_path` values are resolved under `MCP_ENGINE_TESTS_EXPORT_ROOT` (if set), otherwise under `~/.mcp-engine/tests`.
- Large exports may be automatically saved to disk when they exceed `MCP_ENGINE_TESTS_MAX_EXPORT_BYTES`.

---

## Managing Baselines (Snapshots)

### Capture a baseline

```json
{
  "operation": "snapshot",
  "spec": {
    "sub_operation": "capture",
    "test_id": "monthly-sales-snapshot"
  }
}
```

Recapture guidance:
- Existing `hash` baselines should be recaptured when you need Desktop and Service/XMLA parity after upgrading to the canonicalized snapshot behavior.
- Existing `aggregate` baselines captured before this fix should be treated as legacy and recaptured.

### List baselines

```json
{
  "operation": "snapshot",
  "spec": {
    "sub_operation": "list",
    "test_id": "monthly-sales-snapshot"
  }
}
```

### Delete a baseline

```json
{
  "operation": "snapshot",
  "spec": {
    "sub_operation": "delete",
    "baseline_id": "baseline:model:monthly-sales-snapshot@2025-01-01T00:00:00Z"
  }
}
```

Valid snapshot `spec.sub_operation` values:
- `capture`: capture a new baseline for a `regression_snapshot` test
- `list`: list baselines for a test
- `delete`: delete a baseline by `baseline_id`

Snapshot notes:
- Set `spec.sub_operation` explicitly, or use `spec.action` as an alias. `create` and `save` are not valid snapshot sub-operations.
- `capture` and `list` require `test_id`.
- `delete` requires `baseline_id`.

End-to-end baseline flow:
1. Save a `regression_snapshot` test with `operation: "put"`.
2. Capture its baseline with `operation: "snapshot"` and `spec.sub_operation: "capture"`.
3. Run the test with `operation: "run"` to compare current results against the stored baseline.

---

## Listing and Managing Tests

### List all tests

```json
{ "operation": "list" }
```

### List by tags

```json
{ "operation": "list", "spec": { "tags": ["smoke"] } }
```

### Get a specific test

```json
{ "operation": "get", "target": "total-sales-2024" }
```

### Delete a test

```json
{ "operation": "delete", "target": "total-sales-2024" }
```

### Validate a test definition

```json
{
  "operation": "validate",
  "spec": {
    "id": "my-test",
    "name": "My Test",
    "type": "measure_assertion",
    "spec": { "measure": "Sales[Total]" },
    "assert": { "kind": "scalar", "op": "notBlank" }
  }
}
```

Notes:
- `validate` is single-definition oriented. For bulk validation, use `put` with `dry_run: true`.
- If you wrap a single definition in `spec.items`, include exactly one item.
- `validate` uses the same per-test schema as `put`, including required nested fields such as `measure_assertion.spec.measure` and `referential_integrity.spec.checks`.

Example (`referential_integrity`):

```json
{
  "operation": "validate",
  "spec": {
    "id": "sales-customer-no-orphans",
    "name": "Sales has no orphan customer keys",
    "type": "referential_integrity",
    "spec": {
      "checks": [
        { "type": "orphan_keys", "from": "Sales[CustomerKey]", "to": "Customer[CustomerKey]" }
      ]
    },
    "assert": {
      "kind": "integrity",
      "op": "equals",
      "expected": { "type": "number", "value": 0 }
    }
  }
}
```

---

## Capabilities and Skip Behavior

### Check environment capabilities

```json
{ "operation": "capabilities" }
```

Response includes:
- `rls_roles`: Role-only impersonation (works everywhere)
- `rls_effective_user`: Identity impersonation (Service/XMLA only)
- `cache_control`: Cache clearing support
- `trace_fe_se_split`: FE/SE timing split (planned)

### Skipped tests

Tests with unmet `requires` are skipped with a reason and workaround:

```json
{
  "status": "skipped",
  "skip_reason": {
    "type": "capability",
    "capability": "rls_effective_user",
    "message": "Effective User requires XMLA endpoint (Premium/Fabric)",
    "workaround": "Connect to Power BI Service, grant the impersonated identity Read + Build on the semantic model, or use role-only tests"
  }
}
```

---

## Masking Compatibility

When numeric/PII masking is enabled:
- Assertions evaluate on raw values internally (stable behavior)
- Tool output omits `expected`/`actual` values
- `diagnostics_level: full` is blocked
- Snapshot modes `topn`/`full` are blocked unless `allow_sensitive_storage: true`

Bundled Test Runner note:
- The desktop Test Runner sidecar force-disables numeric and PII masking for its own engine process so the app can display raw query and test result values.
- It still uses the same `~/.mcp-engine` tests, baselines, and persisted run storage as the main MCP Engine installation.
- This override is process-local; regular MCP clients continue to honor the user's masking preferences.

---

## Recommended Workflows

### CI/CD Pipeline

1. Apply packs: `packs_apply` with `metadata-quality`, then add `referential-integrity` or `relationship-governance` as needed
2. Run tests: `run` with `stop_on_first_failure: true`
3. Export: `export` with `format: "junit"` and `save_to_path`
4. Fail pipeline if tests fail

### Pre-deployment Validation

1. Connect to model
2. Check capabilities
3. Run smoke tests: `run` with `tags: ["smoke"]`
4. Run full suite if smoke passes
5. Export markdown report for review

### Regression Testing

1. Capture baselines after major changes: `snapshot` with `sub_operation: "capture"`
2. Run regression tests regularly
3. Update baselines when intentional changes occur

# Unit Testing Guide (`manage_tests`)

This guide explains how to use the Pro Unit Testing feature to create, run, and manage tests for Power BI semantic models.

## Related Tools

- `manage_tests`: Pro feature for creating, running, exporting, and transferring tests (14 operations)
- `list_model`: Inspect model schema for test targets (`operation: "list"`)
- `run_query`: Validate DAX queries before using in tests (`operation: "execute"`)
- `manage_model_connection`: Select the model that owns tests, baselines, and run history

## Test Types

<!-- generated:begin:test-types -->
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
<!-- generated:end:test-types -->

## Operations Overview

<!-- generated:begin:operations -->
| Operation | Requires Model | Description |
|-----------|----------------|-------------|
| `capabilities` | Yes | Return environment capabilities |
| `list` | Yes | List tests owned by the connected model |
| `get` | Yes | Get a test definition owned by the connected model |
| `put` | Yes | Create/update a test for the connected model |
| `delete` | Yes | Remove test(s) owned by the connected model |
| `run` | Yes | Execute tests |
| `runs_list` | Yes | List persisted run history for the connected model |
| `export` | Yes | Format connected-model results (json/junit/markdown/html) |
| `export_tests` | Yes | Export connected-model test-definition bundles |
| `import_tests` | No (dry_run); Yes (apply) | Preview/apply portable test-definition bundles |
| `snapshot` | Yes | Capture/list/delete baselines |
| `validate` | No | Validate test definitions |
| `packs_list` | No | List available built-in packs |
| `packs_apply` | Yes | Generate tests from pack |
<!-- generated:end:operations -->

---

## Stable model identity (survives renames/moves)

When a model is connected, `manage_tests` associates tests, baselines, and run history with a **stable model id** stored in the model metadata as a TOM annotation:

- Canonical annotation name: `McpEngine_StableModelId`
- Legacy annotation name: `PowerBIMcp_StableModelId` (read during the 3.x compatibility window and backfilled to the canonical name during confirmed write-capable flows; the legacy reader is removed in 4.0.0)
- Value: GUID string
- Stored `model_id` in the test database: `stable:{guid}`

After identity resolution, SemanticOps automatically migrates persisted tests, baselines, and run history from known legacy model IDs to the canonical identity in one transaction. Existing test, baseline, and run IDs are preserved; subsequent model-scoped operations query only the canonical identity. Until a Desktop model has a stable identity, read-only operations continue using the legacy database-name or port identity and do not migrate storage to the path-scoped fallback.

This keeps unit tests tied to the same model even if the PBIX is renamed/moved (Desktop) or if the dataset display name changes (Service).

Every persisted test definition, baseline, and run belongs to exactly one canonical model id. Test ids remain globally unique: a test id already owned by another model is reported as a conflict on write and behaves as not found on reads or deletes. There is no global/shared test scope and no ownership-transfer operation.

`list`, `get`, `put`, `delete`, `run`, `runs_list`, `export`, `export_tests`, snapshot operations, and pack application require a connected model. `validate`, `packs_list`, and an `import_tests` schema preview (`dry_run: true`) can run while disconnected; applying an import requires an explicitly selected model.

Note: the first persisted testing action for a model may write the `McpEngine_StableModelId` annotation into model metadata. That is a model change, and you need to save the PBIX if you want the annotation to persist.

The Tauri test runner uses `runs_list` to hydrate persisted run history for the currently connected model. Run history remains server-side only; the UI does not keep a separate local history store.

### Run stability

`runs_list` preserves its existing response by default. Set `spec.include_stability` to `true` to add backend-owned stability classifications and retry evidence to the response. `spec.stability_window` defaults to `10` and must be at least `4`; `spec.limit` still controls how much persisted history is available, so a sparse or truncated history can remain insufficient.

```json
{
  "operation": "runs_list",
  "spec": {
    "limit": 50,
    "include_stability": true,
    "stability_window": 10
  }
}
```

The opt-in response adds a top-level `stability` array. Each entry contains `test_id`, `test_name`, `classification`, evaluated `window`, `passed_on_retry_count`, and `alternation_count`. Classifications use the newest comparable same-definition segment:

- `insufficient_history`: fewer than four comparable passed or failed results, including legacy or otherwise hashless history.
- `flaky`: any passed-on-retry evidence, or another mixed pass/fail history not classified as degraded.
- `stable`: every comparable result passed.
- `degraded`: every comparable result failed, or the newest two results failed after earlier passes.

Skipped, XFail, and error results do not count toward the window or alternations. A definition identity change stops comparison with older results. SemanticOps retains the internal identity for masked and unmasked runs, but never returns `definition_hash` in client responses or exports. Legacy masked runs without an identity remain non-comparable.

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

### 5. Transfer test definitions

```json
{ "operation": "export_tests", "spec": { "tags": ["smoke"] } }
```

```json
{ "operation": "import_tests", "dry_run": true, "spec": { "bundle": { "kind": "mcp_engine_test_bundle", "bundle_version": "1.0", "exported_at": "2026-05-13T00:00:00Z", "count": 0, "tests": [] } } }
```

---

## Creating Tests

### Single create/update and validate shapes

Single-test `put` and `validate` accept the canonical nested test-definition shape:

```json
{
  "operation": "put",
  "spec": {
    "id": "total-sales-positive",
    "name": "Total Sales is positive",
    "type": "measure_assertion",
    "spec": { "measure": "Sales[Total Sales]" },
    "assert": { "kind": "scalar", "op": "gt", "expected": { "type": "number", "value": 0 } },
    "context": {
      "filters": [
        { "table": "Date", "column": "Year", "op": "eq", "values": [{ "type": "number", "value": 2026 }] }
      ]
    }
  }
}
```

The request-level `spec` is the complete test definition, including its `assert` and `context`. Request-root `assert` and `context` fields are rejected. Type-specific fields remain inside the definition's own `spec`; for example a `measure_assertion` put uses request `spec.spec.measure`.

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
  "stop_on_error": true,
  "dry_run": false,
  "include_items": false
}
```

`stop_on_error` defaults to `true`. It stops processing after the first item error but does not provide atomic rollback. Bulk responses report `total_items` as the submitted count, plus `processed_items`, `failed_items`, `stop_on_error`, `dry_run`, `results`, and `errors`.

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
- `assert.expected` must use the TypedValue object shape.
- Tolerance uses `{"abs": <number>}` and/or `{"rel": <number>}`.

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
      "path": ["[Value]"],
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
- `assert.expected` uses the same canonical TypedValue shape as `measure_assertion`.

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
      "kind": "rls",
      "expectations": [
        { "role": "SalesRole", "op": "row_count_max", "max": 10000 },
        { "role": "ManagerRole", "op": "query_succeeds" }
      ]
    }
  }
}
```

Note: `rls_effective_user` capability (identity impersonation) requires Power BI Service XMLA endpoint.
The impersonated identity must also have both Read and Build permission on the semantic model. `Build` enables XMLA/query access; it does not bypass RLS for Viewer users.
Current scope:
- RLS principals must use `{ "role": "RoleName" }` or `{ "role": "RoleName", "identity": "user@contoso.com" }`.
- Do not use `{ "kind": "role", "name": "RoleName" }`; `kind` and `name` are not supported principal fields for `rls_validation`.
- RLS assertions use `kind: "rls"` with exactly one expectation per principal. Omitted `kind` is normalized to `rls`.
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

Cold and warm run behavior:
- `spec.runs.cold` iterations attempt to clear the VertiPaq cache before each query execution when `cache_control` is supported.
- Desktop connections support `cache_control`; Power BI Service/shared-capacity XMLA connections may restrict cache clearing.
- If cache clearing fails or is unavailable, the affected cold iteration still runs and is reported in safe diagnostics as `cache_cleared: false`; interpret that timing as warm-cache.
- Budgets with cold runs consume the `cache_control` capability when supported, regardless of which timing thresholds are asserted; unsupported cache control is surfaced in assertion diagnostics as `cache_cleared: false`.
- When a run includes a supported `cache_control` test, the runner executes the batch sequentially so other DAX-running tests cannot warm the model cache between clear and measurement.
- Supported `cache_control` runs remain isolated because each engine host executes at most one `manage_tests` run at a time, so another task-backed run cannot warm the model cache while cold measurements are in progress.
- Warm-only budgets (`spec.runs.cold: 0`) do not require `cache_control` and run unchanged on Service.

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
      "baseline_id": "baseline:stable:11111111-1111-1111-1111-111111111111:monthly-sales-snapshot@2026-08-12T10:15:30.0000000Z",
      "op": "equals"
    }
  }
}
```

Snapshot modes: `hash` (deterministic hash), `aggregate` (row count + checksums), `topn` (first N rows), `full` (all data)

`baseline_id` is an opaque identifier returned by snapshot capture or list. Do not construct it. Omit `assert` or `assert.baseline_id` when saving a draft before its first capture. A durable `put` with a baseline reference verifies that the baseline exists for the connected model and belongs to the same test; disconnected validation and `put` dry runs check structure only.

For `DaxQueryResult` snapshots:
- Newly captured `hash` and `aggregate` baselines sort canonical row JSON with ordinal comparison before applying `row_cap`. Engine row-order changes therefore do not cause regressions, while duplicate rows and selected-column order remain significant.
- `hash` and `aggregate` operate on canonicalized row data, not transport-specific CLR numeric types.
- Whole numbers are normalized so Desktop and Service/XMLA transports hash and compare consistently when the result values are semantically equal.
- `topn` and `full` remain order-sensitive and preserve the same envelope shape, but row cell values are canonicalized before comparison.
- `validate` and `put` return a non-blocking warning when the final top-level `EVALUATE` clause has no `ORDER BY`. Add `ORDER BY` for intentional, repeatable `topn` or `full` snapshots; `hash` and `aggregate` still canonicalize row order.
- Existing bare-string `hash` baselines and `aggregate` baselines without `canonical_order: "v2"` retain their legacy ordered comparison semantics. Recapture them when you want order-insensitive comparisons; no automatic rewrite is performed.
- A v2 hash mismatch reports both baseline and current row counts. A legacy hash mismatch reports `baseline_rows=unknown` because the historical payload does not contain a count; aggregate mismatches continue reporting row counts and checksums.
- New `hash` and `aggregate` captures and v2 comparisons fail closed when the query result is truncated, because sorting an engine-selected prefix cannot guarantee order-insensitive behavior. Reduce the DAX result or increase the `max_query_rows` preference so the complete result reaches canonical sorting; `row_cap` is applied afterward and does not make an earlier truncation safe.

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
  - no params: assert that the model has some marked date table or non-system modern calendar
  - with `params.table`: assert that a specific table is the marked date table or owns a non-system modern calendar

Discovery:
- `operation: "capabilities"` returns `metadata_compliance.rules` with the built-in rule catalog, `metadata_compliance.entity_types`, and canonical `assertion_list_field: "rules"`.
- `spec.rules` is the assertion list field. `spec.checks` and shorthand rule types are rejected.

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

When a model is connected, each returned pack includes `applied`, `applied_test_count`, nullable `generated_test_count`, and nullable `applied_generated_test_count`. Applied counts are based on stored tests whose `source_pack` matches the pack id, while generated counts represent the current model-specific tests that the pack would generate. `applied_generated_test_count` is the intersection used for current coverage such as `2/3`. Without enough model context, packs remain listable but the generated coverage fields are omitted from the response (they are nullable and serialized with `WhenWritingNull`, so clients should treat missing and `null` equivalently).

### Apply a pack

```json
{ "operation": "packs_apply", "spec": { "pack_id": "metadata-quality" } }
```

`spec.pack_id` is required; `spec.pack` is rejected.

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

Each engine host executes at most one model-bound test operation at a time. Runs, snapshot capture, test bundle export, connected pack list/application, connected import preview/application, and valid `put` mutations acquire one model-session lease before resolving the canonical model scope and hold it through their final model-derived response or persistence step. A queued operation therefore resolves its model only after it acquires the lease; callers waiting for another operation may cancel their request without blocking later work.

While a protected test operation owns the lease, `manage_model_connection` operations that would replace or alter its session (`select`, `reload`, `sign_out`, and `set_impersonation`, including clearing impersonation) fail immediately with `code: "test_run_in_progress"`, `retryable: true`, and an instruction to retry after the operation finishes. Read-only discovery and status operations remain available. Storage-only test history, baseline, and catalog operations retain ordinary concurrency; separate engine hosts remain independent; and the `parallelism` option continues to control concurrent tests within an active run.

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

Use `spec.test_ids`; `spec.ids` is rejected.

### Stop on first failure

```json
{ "operation": "run", "spec": { "stop_on_first_failure": true } }
```

When enabled, the run stops after the first failed assertion or execution error. Within-run parallelism is disabled so the stopping point and returned result order remain deterministic. Passed, skipped, and expected-failure (`xfail`) results do not stop the run.

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

`export` is for run results and reports. To move test definitions between models or machines, use `export_tests` / `import_tests` instead.

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

## Transferring Test Definitions

Use `export_tests` and `import_tests` for portable test-definition bundles. Bundles contain test definitions only: no run history, no snapshot baselines, and no persisted model identity. Imported tests are saved against the currently connected model so the same bundle can be reused across Desktop and Service models.

When upgrading a legacy test database, SemanticOps exports any global definitions before removing them. The recovery bundle is written to the configured tests export/import root as `global-tests-recovery-<timestamp>.json` with user-only local permissions (beside `tests.db` when no separate export root is configured). Connect to the intended destination model, review the bundle, then apply it with `import_tests`; recovery definitions are never silently assigned to a model.

Bundles include full `spec`, `assert`, and `context` payloads. These fields are authored configuration and remain available when numeric or PII masking is enabled. Large inline bundles are saved to the configured tests export root and return `saved_to` with `content_omitted: true`.

### Export all tests inline

```json
{ "operation": "export_tests" }
```

### Export selected tests

```json
{
  "operation": "export_tests",
  "spec": {
    "test_ids": ["total-sales-2024", "metadata-quality-descriptions"]
  }
}
```

### Export by filters

```json
{
  "operation": "export_tests",
  "spec": {
    "tags": ["smoke"],
    "types": ["measure_assertion", "metadata_compliance"]
  }
}
```

### Save a definition bundle to disk

```json
{
  "operation": "export_tests",
  "spec": {
    "tags": ["release"],
    "save_to_path": "./test-bundles/release-tests.json"
  }
}
```

When `save_to_path` is set, or when a bundle exceeds `MCP_ENGINE_TESTS_MAX_EXPORT_BYTES`, the response returns `saved_to`, `content_omitted: true`, and `count` instead of echoing the full bundle.

### Preview an import

```json
{
  "operation": "import_tests",
  "dry_run": true,
  "spec": {
    "read_from_path": "./test-bundles/release-tests.json",
    "mode": "add_update"
  }
}
```

### Apply an import

```json
{
  "operation": "import_tests",
  "dry_run": false,
  "spec": {
    "bundle": {
      "kind": "mcp_engine_test_bundle",
      "bundle_version": "1.0",
      "exported_at": "2026-05-13T00:00:00Z",
      "count": 0,
      "tests": []
    },
    "mode": "add_skip"
  }
}
```

Import modes:
- `add_update` (default): add new tests and update existing ids.
- `add_skip`: add new tests and skip existing ids.
- `fail_on_conflict`: treat existing ids as conflicts and do not apply until resolved.

Validation notes:
- Schema-invalid tests and malformed bundles block import.
- Model-reference checks are warnings only. For example, missing measures, roles, tables, or columns do not block the bundle from being previewed or applied.
- Existing ids owned by another model are not overwritten by `add_update`; use a new test id or `add_skip` to leave the existing row unchanged.
- `read_from_path` uses the same export-root policy as `save_to_path`: relative paths resolve under `MCP_ENGINE_TESTS_EXPORT_ROOT` when set, otherwise under the managed tests directory.

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
- Existing `hash` baselines should be recaptured when you need Desktop and Service/XMLA numeric parity or order-insensitive comparison after upgrading to the canonicalized snapshot behavior.
- Existing `aggregate` baselines captured before this fix remain valid with their original ordered semantics; recapture them to opt in to order-insensitive checksums.

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
    "baseline_id": "baseline:stable:11111111-1111-1111-1111-111111111111:monthly-sales-snapshot@2026-08-12T10:15:30.0000000Z"
  }
}
```

Valid snapshot `spec.sub_operation` values:
- `capture`: capture a new baseline for a `regression_snapshot` test
- `list`: list baselines for a test
- `delete`: delete a baseline by `baseline_id`

Snapshot notes:
- Set `spec.sub_operation` explicitly. `spec.action`, `create`, and `save` are rejected.
- `capture` and `list` require `test_id`.
- `delete` requires `baseline_id`.

End-to-end baseline flow:
1. Save a draft `regression_snapshot` test with `operation: "put"`, omitting `assert` or `assert.baseline_id`.
2. Capture its baseline with `operation: "snapshot"` and `spec.sub_operation: "capture"`; retain the returned `baseline_id`.
3. Save the complete test with a second `put`, setting `assert.baseline_id` to the returned identifier. Capture does not activate or replace a test's baseline automatically.
4. Run the test with `operation: "run"` to compare current results against the stored baseline.

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
- `get` returns the complete authored definition, including `context`, `spec`, `assert`, and `meta.owner`
- `export_tests` returns complete authored definition bundles without a sensitive-content opt-in
- Assertions evaluate on raw values internally (stable behavior)
- Run results omit assertion `expected`/`actual` values and replace non-null assertion messages with stable pass/fail text
- Value-bearing and unclassified `error_message` details are replaced with stable status text; known structural failures use fixed masking-safe guidance without echoing submitted operators, assertion paths, object names, query errors, or model data
- Structural guidance covers malformed definitions, unsupported scalar operators, unsupported DAX assertion context, invalid or unresolved DAX assertion paths, multi-row/multi-column result shapes, and truncated DAX assertion results; legacy results without a recognized diagnostic identity continue to fail closed with generic masked text
- The known-safe `Test execution timed out` and `Test execution failed` infrastructure messages remain unchanged
- Run identifiers, model and test identifiers, timestamps, summary, durations, assertion kind/status, and capability diagnostics remain available
- Known skip categories and built-in capability identifiers remain available as stable guidance; arbitrary categories and capability names are removed, skip messages are replaced with stable type-based masked text (except the exact known-safe `Test is disabled` message), and all workarounds are cleared
- Newly executed runs are projected before persistence, and stored history is projected again on list and JSON/JUnit/Markdown/HTML export so older or unmasked sidecar-created records are safe to read from a masking-enabled host. Internal definition identities survive that projection for stability classification and remain absent from client-visible payloads
- `diagnostics_level: full` is blocked
- Snapshot modes `topn`/`full` are blocked unless `allow_sensitive_storage: true`

Bundled Test Runner note:
- The desktop Test Runner sidecar force-disables numeric and PII masking for its own engine process so the app can display raw query and test result values.
- It still uses the same `~/.mcp-engine` tests, baselines, and persisted run storage as the main SemanticOps MCP installation, formerly MCP Engine.
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

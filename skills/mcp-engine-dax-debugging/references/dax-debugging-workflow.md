# DAX Debugging Workflow

Use this workflow when a measure returns a wrong value, an unexpected blank, an inflated total, or numbers that disagree with the source system. It reproduces the systematic loop experienced modelers run manually in DAX Studio: reproduce, establish ground truth, decompose, inspect filter propagation, confirm the cause numerically.

This is correctness debugging. For slow-but-correct queries, use the performance guides instead (`query-performance-guide.md (from the mcp-engine-dax-performance skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`).

## Related Tools and Resources

- Reproduce and probe: `run_query` (`execute`, `test_access`)
- Definitions and metadata: `list_model` (`list` with `include_expression`, `search`, relationship and partition listings)
- Consumer and dependency context: `manage_dependencies` (`used_by`)
- Regression protection after the fix: `manage_tests` (`measure_assertion`, `dax_assertion`, `referential_integrity`)
- Related guides: `dax-query-guide.md (from the mcp-engine-query skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`, `relationships-guide.md (from the mcp-engine-schema-authoring skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`, `security-roles-guide.md (from the mcp-engine-security-governance skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`, `measure-authoring-guide.md (from the mcp-engine-semantic-authoring skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`

## Operating Rules

- Stay read-only until the root cause is proven. Do not "try a fix and see" — a fix that changes the number without an explained cause hides the bug instead of removing it.
- Change one variable per probe. Every query differs from the previous one in exactly one filter, one sub-expression, or one relationship assumption.
- Keep a written trail: expected value, actual value, and each probe with its result. The final explanation must account for the discrepancy numerically (for example, "total is 3x because each fact row matches 3 dimension rows").
- Keep probes aggregated and small. Never dump raw rows from tables that may contain sensitive values.

## Workflow

### Step 1 — State the discrepancy precisely

Record before querying anything:

- The measure and the exact filter context (slicers, page filters, visual grouping) where it is wrong.
- Actual value, expected value, and where the expectation comes from (source system, old report, user knowledge).
- Whether the user sees the problem always or only in specific contexts (specific period, specific role, totals row only).

If the user cannot state an expected value, first agree on the ground-truth query of step 3 — a discrepancy is only debuggable against a definition of correct.

### Step 2 — Reproduce minimally

Rebuild the wrong number outside the report with the smallest equivalent query:

```dax
EVALUATE
CALCULATETABLE(
    ROW("Actual", [Problem Measure]),
    'Dim Product'[Category] = "Accessories",
    'Dim Date'[Year] = 2025
)
```

Run it with `run_query` `{ "operation": "execute", "query": "<the query above>" }` (`query` is required). If this does not reproduce the report's number, the difference is in report-layer context (visual filters, interactions, field parameters, RLS as a viewer) — enumerate those with the user before continuing.

### Step 3 — Establish ground truth

Agree on the business definition and grain, then compute that exact aggregation from the most primitive fields while bypassing the suspect measure. Use `SUM` only when the metric is additive; preserve `DISTINCTCOUNT`, averages, ratios, semi-additive balances, and other non-additive semantics. The following example applies only to an additive sales-amount definition:

```dax
EVALUATE
ROW(
    "Expected",
    CALCULATE(
        SUM('Fact Sales'[Amount]),
        'Dim Product'[Category] = "Accessories",
        'Dim Date'[Year] = 2025
    )
)
```

- Ground truth equals the expected value but the measure disagrees → the defect is in the measure expression. Go to step 4.
- Ground truth is also wrong → the defect is in the model or data (relationships, grain, refresh). Go to step 5.
- Both match the report but the user expected otherwise → the defect is a definition mismatch; reconcile the business definition before touching anything.

### Step 4 — Decompose the measure

1. Get the definition: `list_model` `{ "operation": "list", "spec": { "type": "measures", "include_expression": true, "table": "<table>" } }`.
2. Evaluate its parts stepwise in the reproduction context — each `VAR`, each `CALCULATE` argument, each referenced measure — one `ROW` query per part.
3. Binary-search for the divergence: find the innermost sub-expression whose value is already wrong. Referenced measures recurse: extract the `[Measure Name]` references from the fetched expression, fetch each referenced measure's definition the same way, and debug the deepest wrong measure first. (`manage_dependencies` `used_by` returns downstream consumers of a target — useful for blast radius, not for finding a measure's inputs.)
4. When the divergent part is a filter argument, print what it selects: `EVALUATE CALCULATETABLE(VALUES('Dim'[Col]), <same filters>)` shows the surviving filter values.

### Step 5 — Inspect the filter path

List the relationships on the path from each filtered table to the fact: `list_model` `{ "operation": "list", "spec": { "type": "relationships" } }`.

For every hop, check in order: does the relationship exist; is it active; does the cross-filter direction point from the filter table toward the fact; is the cardinality what the data actually holds; do the key columns have matching data types (`{ "operation": "list", "spec": { "type": "columns", "include_details": true } }`).

Key integrity probes:

```dax
// Duplicate keys on the one-side inflate results on the many-side
EVALUATE
FILTER(
    SUMMARIZECOLUMNS('Dim Product'[ProductKey], "n", CALCULATE(COUNTROWS('Dim Product'))),
    [n] > 1
)

// Orphan fact rows land in the blank dimension member
EVALUATE
ROW(
    "OrphanRows",
    COUNTX(FILTER('Fact Sales', ISBLANK(RELATED('Dim Product'[ProductKey]))), 1)
)
```

`manage_tests` offers `referential_integrity` as a durable version of the orphan probe.

### Step 6 — Match the symptom

| Symptom | Most likely causes | Deciding probe |
| --- | --- | --- |
| Value inflated or duplicated | Duplicate dimension keys; many-to-many or bidirectional propagation; two facts joined through a shared dimension | Duplicate-key probe above; relationship listing; expected/actual ratio equals the duplication factor |
| Unexpected blank | No relationship path; wrong filter direction; orphan keys; key data type mismatch; RLS | Path check in step 5; orphan probe; step 7 |
| Total row disagrees with the sum of detail rows | Iterator or non-additive semantics (often correct by design); calculation group interference; bidirectional filtering | Decompose at total vs. detail context; list calculation groups (`{ "operation": "list", "spec": { "type": "calculation_groups" } }`) |
| Slicer has no effect | Inactive relationship without `USERELATIONSHIP`; missing path; measure removes the filter | `list_model` `{ "operation": "search", "spec": { "mode": "dax", "query": "REMOVEFILTERS" } }` (repeat for `ALL`) on the measure chain; relationship listing |
| Disagrees with the source system | Fact loaded at a different grain; incomplete refresh or partition gaps; upstream Power Query filters | `EVALUATE ROW("n", COUNTROWS('Fact Sales'))` vs. source count; `{ "operation": "list", "spec": { "type": "partitions", "table": "<fact>" } }` for ranges and refresh times |

A symptom can have several contributing causes; the loop ends only when step 8 closes numerically.

### Step 7 — Rule out security and masking

- Reproduce under the affected user's role: `run_query` `{ "operation": "test_access", "query": "<the step 2 reproduction query>", "spec": { "roles": ["<role name>"] } }` (both the top-level `query` and `spec.roles` are required) — RLS makes numbers differ per viewer by design.
- Masked display values are a rendering layer, not a data defect: check for masking flags (`display_columns`, `numeric_masked`) in responses before concluding data is wrong.

### Step 8 — Close the loop numerically

State the root cause with its evidence, and show the arithmetic that reconciles actual to expected (duplication factor, orphaned row count, missing partition's contribution). If the numbers do not reconcile fully, the diagnosis is incomplete — return to the narrowest unexplained probe.

### Step 9 — Fix and protect

1. Hand the fix to the matching authoring guide (measure logic → `measure-authoring-guide.md (from the mcp-engine-semantic-authoring skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`; relationships or keys → `relationships-guide.md (from the mcp-engine-schema-authoring skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`; upstream data → the user's ETL).
2. After the fix, re-run the step 2 reproduction and step 3 ground truth — they must now agree.
3. Add a regression test: build the complete definition (`measure_assertion` pinning the expected value in the reproduction context, or `referential_integrity` when the cause was orphan keys; payload shapes are in `unit-testing-guide.md (from the mcp-engine-testing-changes skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`), check it with `manage_tests` `{ "operation": "validate", "spec": { <definition> } }`, save the same definition with `{ "operation": "put", "spec": { <definition> } }`, then `{ "operation": "run" }`.

## Report to the user

1. The discrepancy: measure, context, actual vs. expected.
2. The root cause, with the probe evidence and the arithmetic that reconciles the numbers.
3. Whether it is a measure defect, model defect, data defect, security behavior, or definition mismatch.
4. The fix applied or recommended, and the verification result.
5. The regression test added, with its id.

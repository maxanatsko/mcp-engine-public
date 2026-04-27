# Performance Remediation Playbook (Pro)

This is a practical “symptom → diagnosis → tool → fix” guide for Power BI model performance. Use it when a report is slow, a visual times out, or refresh/performance regressions appear after changes.

## Related Tools and Resources

- `run_query` with `operation: "analyze"` (Pro): Storage Engine / Formula Engine metrics; optional query plan capture
- `run_query` with `operation: "vertipaq"` (Pro): VertiPaq storage statistics, optional exact cardinality
- `run_query` with `operation: "execute"`: Validate correctness quickly while iterating
- `list_model` with `operation: "search"`: Find expensive measures/calculation items
- `dax-query-plan-reference.md` (Pro): how to read plan operators
- `vertipaq-optimization-guide.md` (Pro)
- `query-performance-guide.md` (Pro)

## A. Triage: What Kind of Slowness?

Start by measuring a representative query (ideally the one a slow visual issues).

```json
{
  "operation": "analyze",
  "query": "EVALUATE TOPN(10, 'Sales')",
  "spec": {
    "runs": 3,
    "clear_cache": true,
    "use_xevents": true
  }
}
```

Interpretation shortcut:

- **SE-heavy**: Storage Engine scans/joins/aggregations dominate.
- **FE-heavy**: Formula Engine (DAX iterators/context transitions) dominates.
- **Both heavy**: often model bloat + complex measures.

When you need deeper insight, capture a plan:

```json
{
  "operation": "analyze",
  "query": "EVALUATE SUMMARIZECOLUMNS('Date'[Year], \"Sales\", [Total Sales])",
  "spec": {
    "runs": 3,
    "include_query_plan": true
  }
}
```

## B. Symptom → Diagnosis → Fix

### 1) High Storage Engine time (SE)

Symptoms:

- `run_query` with `operation: "analyze"` shows high SE duration and/or many SE queries.

Diagnosis steps:

1. Check model bloat:
   - `run_query` with `operation: "vertipaq"` for the involved tables.
2. Check relationship layout:
   - `list_model` with `operation: "list"`, `spec: { type: "relationships" }`
3. Check key columns:
   - high-cardinality, text keys, GUIDs, wide fact tables.

Typical fixes:

- Reduce fact table width: remove unused columns, especially high-cardinality strings.
- Replace text/GUID relationships with integer surrogate keys (where feasible).
- Ensure star schema with one-direction filtering.
- Avoid bidirectional relationships unless necessary (can increase join work).
- If query is scanning many rows, push filters earlier (dimension filters instead of fact filters).

Verification:

- Re-run `run_query` with `operation: "analyze"` with `clear_cache=true` to compare.

### 2) High Formula Engine time (FE)

Symptoms:

- FE duration dominates; SE is relatively low.

Common causes:

- Heavy iterators (`SUMX`, `FILTER`, `ADD COLUMNS`) over large tables.
- Repeated context transitions (`CALCULATE` inside iterators).
- Poorly scoped filters / row context “explosions”.
- Non-additive measures computed at granularities larger than necessary.

Diagnosis steps:

1. Search for expensive patterns:
   - `list_model` with `operation: "search"` and `spec: { mode: "dax", query:"SUMX("`
   - `list_model` with `operation: "search"` and `spec: { mode: "dax", query:"FILTER("`
2. Identify the specific measure used in the slow visual and inspect its dependencies.
3. If you captured a plan, look for expensive FE operators and large intermediate rowsets.

Typical fixes:

- Use variables (`VAR`) to avoid recomputing expressions.
- Replace iterators with simpler aggregations where possible.
- Reduce cardinality of the iteration table (iterate over dimensions, not facts).
- Prefer `SUMMARIZECOLUMNS` patterns over manual table shaping when appropriate.
- Avoid measure-in-boolean-filter patterns; store measure results in variables first.

Verification:

- Use `run_query` to validate results on a small set of slices.
- Re-run `run_query` with `operation: "analyze"`.

### 3) Slow visuals only when many slicers are applied

Symptoms:

- Query is fast “unfiltered”, slow under heavy slicer selections.

Common causes:

- Bidirectional relationships creating ambiguous paths.
- Many-to-many relationships or complex bridge tables.
- High-cardinality slicers on large fact tables.

Fixes:

- Move slicers to dimension tables.
- Simplify relationships: one-direction, star schema.
- Use aggregation tables or precomputed dimensions where appropriate.

### 4) VertiPaq model is large / memory pressure

Symptoms:

- Long refresh times, slow queries after data growth, or large storage sizes.

Diagnosis:

- `run_query` with `operation: "vertipaq"` across the model.
- Optionally enable exact `include_cardinality=true` for suspect tables (can be slow).

Fixes:

- Remove unused columns.
- Prefer numeric keys over long text.
- Reduce precision where safe (e.g., avoid `Double` if `Decimal` or integer fits).
- Split large text columns into separate dimensions or exclude them.

### 5) Regression after refactor or bulk edits

Symptoms:

- Performance worsened after measure/relationship changes.

Best workflow (Pro):

- Use `manage_model_changes`:
  - pin a checkpoint before major refactors
  - diff the transaction that introduced regression
  - rollback if necessary

Verification:

- Keep a small set of “golden” performance queries to re-run after changes.

### 6) Query plan capture is huge / hard to read

Fix:

- Start with metrics-only runs first (`include_query_plan=false`).
- Narrow the query to the smallest reproducer.
- Use `dax-query-plan-reference.md` to interpret the operators.

### 7) Direct Lake fallback to DirectQuery (Fabric)

Symptoms:

- Queries in Direct Lake mode are slower than expected
- Fabric capacity metrics show high fallback rate
- SE/FE metrics show DirectQuery patterns instead of Direct Lake

Common causes:

- Complex iterators (`SUMX`, `AVERAGEX`, `FILTER` over large tables)
- Certain DAX patterns not supported in Direct Lake mode
- Cross-filter bidirectional relationships
- `CALCULATE` with complex filter arguments

Diagnosis:

1. Check Fabric capacity admin portal for fallback metrics
2. Identify which measures trigger fallback by testing individually
3. Look for iterator patterns in measure expressions:
   - `list_model` with `operation: "search"` and `spec: { mode: "dax", query:"SUMX("`
   - `list_model` with `operation: "search"` and `spec: { mode: "dax", query:"FILTER("`

Typical fixes:

- Simplify measure expressions to avoid iterators
- Pre-aggregate in the lakehouse (create summary tables)
- Use aggregation tables in the semantic model
- Consider hybrid model: Direct Lake for simple queries, aggregations for complex
- Move complex calculations to lakehouse SQL views

Verification:

- Monitor fallback rate in Fabric capacity metrics
- Re-test representative queries after changes

### 8) Aggregation tables not being used

Symptoms:

- Queries hit detail table instead of aggregation
- Performance doesn't improve after adding aggregations

Common causes:

- Aggregation mappings not configured correctly
- Query grain is below aggregation grain
- Filters on columns not in aggregation

Diagnosis:

- Check aggregation configuration in Power BI Desktop
- Test with queries at aggregation grain
- Review which columns are included in aggregation

Fix:

- Ensure aggregation mappings cover common query patterns
- Include frequently filtered columns in aggregation
- Consider multiple aggregation tables at different grains

## C. Practical "Golden Queries" for Baselines

Build a small set of representative queries for:

- a key KPI by a common axis (e.g., year/month)
- a common slicer combination
- the slowest visual in the report

Store them externally and rerun with:

- `clear_cache=true` for apples-to-apples comparisons
- a consistent `runs` setting (3 or 5)

## D. Change Safely

When changing performance-sensitive parts:

1. Make one change at a time.
2. Validate correctness (`run_query`) before optimizing further.
3. Measure (`run_query` with `operation: "analyze"`) before/after.
4. If doing a batch refactor, use checkpoints/changesets (`../../mcp-engine-testing-changes/references/model-changes-guide.md`).

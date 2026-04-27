# DAX Query Plan Operator Reference (Pro)

This reference describes operators and patterns found in DAX query plans captured by the `run_query` tool with `operation: "analyze"` (Pro feature).

Use it as a practical guide for interpreting **logical** and **physical** plans and producing actionable, low-hallucination recommendations.

## Related Tools and Resources

- `run_query` (`operation: "analyze"`, `spec: { include_query_plan: true }`)
- `query-performance-guide.md` (high-level performance patterns)
- `performance-remediation-playbook.md` (end-to-end diagnostic workflow)

## Scope and Caveats

- **Plans are diagnostic artifacts, not guarantees.** Names, shapes, and fields can vary across Power BI Desktop/engine versions.
- **Do not guess unknown operators.** If an operator/token is unfamiliar, quote it and describe where it appears; don’t invent a meaning.
- **Performance issues are rarely “one operator.”** Look for *root cause chains* (e.g., filter not pushed down → large scan → large hash join → spill/spool).
- **Not all “scary” operators are bad.** `Spool`/`Cache` can be legitimate optimizations; treat them as a prompt to verify cardinality and reuse.

## Logical Plan Operators

| Operator | Description | Performance Impact |
|----------|-------------|-------------------|
| Scan_Vertipaq | Column store scan | O(rows) - check cardinality |
| Filter_Vertipaq | Predicate pushdown | Reduces scan - good when early |
| GroupBy_Vertipaq | Aggregation | Memory proportional to groups |
| Join_Inner | Table relationship | Check cardinality on both sides |
| Sum_Vertipaq | SUM aggregation | Efficient columnar operation |
| Count_Vertipaq | COUNT aggregation | Efficient columnar operation |

## Physical Plan Operators

| Operator | Description | Watch For |
|----------|-------------|-----------|
| VertiPaq | Direct engine query | Good - uses columnar storage |
| CallbackDataID | Row context callback | Bad - very slow, row-by-row |
| Spool | Intermediate materialization | Memory pressure, check row count |
| CrossJoin | Cartesian product | Cardinality explosion risk |
| Cache | Result caching | Good for repeated subexpressions |

## How to Read the Plan (Workflow)

1. **Start with the logical plan**: identify the high-level intent (grouping, joins, iterators, filters).
2. **Switch to the physical plan**: find how the engine executes (batch vs row-by-row, materialization, joins).
3. **Identify the “largest data moments”**:
   - the biggest scans,
   - the biggest intermediate results,
   - where cardinality explodes (joins/crossjoins),
   - any row-by-row callbacks (`CallbackDataID`).
4. **Trace why they happen**: missing filters, many-to-many relationship behavior, iterator misuse, context transition.
5. **Propose minimal changes first**: filter pushdown, reduce cardinality, avoid iterators, pre-aggregate, change relationship strategy.
6. **Ask 1–2 targeted questions** if the root cause depends on model context (relationship cardinality, column cardinality, DirectQuery).

## How to Read Numbers (Cardinality and Cost Signals)

Query plans often include numeric metadata as attributes or adjacent text. Exact field names vary, but look for:

- **Row/record counts**: `Rows`, `#Records`, `Cardinality`, `EstimatedRows`, `Recs`, `Records`.
- **Group counts**: `Groups`, `Distinct`, `Cardinality`.
- **Memory-related signals**: `Memory`, `Spool` size, “spill”/“temp” hints (if present).
- **Join inputs/outputs**: look for counts on both sides and the output. A small input joined to a huge input is fine; huge×huge is often not.

Interpretation heuristics:

- **Early filters are good.** If you see a large scan and then a filter later, the engine couldn’t push the predicate down.
- **Estimated rows matter even if the query is “fast on my machine.”** Large estimated intermediates usually become slow at scale.
- **Watch multiplicative patterns.** If one step outputs roughly the product of two earlier counts, that’s a cardinality explosion.

## Common Operators (Frequently Seen, Not Exhaustive)

Operator naming varies. Treat this as a “most likely meanings” map, not a strict spec.

### Scans and Filters

| Operator (examples) | Meaning | Practical notes |
|---|---|---|
| `Scan_Vertipaq`, `VertiPaqScan` | Column scan | Check which columns are scanned; high-cardinality columns are more expensive. |
| `Filter_Vertipaq`, `Filter` | Predicate application | Best when it appears close to the scan; late filters indicate blocked pushdown. |
| `Distinct`, `HashDistinct` | Deduplication | Often used for relationship traversal; can be expensive with high cardinality. |

### Aggregation and Grouping

| Operator (examples) | Meaning | Practical notes |
|---|---|---|
| `GroupBy_Vertipaq`, `Aggregate`, `Sum_Vertipaq`, `Count_Vertipaq` | Aggregation | Cost roughly proportional to scanned rows and number of groups. High groups → memory pressure. |
| `Sort`, `TopN` | Ordering/limiting | `TopN` can be cheap if it is pushed down early; expensive if applied after huge intermediates. |

### Joins and Relationship Traversal

| Operator (examples) | Meaning | Practical notes |
|---|---|---|
| `Join_Inner`, `HashMatch`, `MergeJoin` | Join | Verify join keys, relationship cardinality, and input sizes; hash joins with huge inputs are costly. |
| `GroupSemiJoin`, `SemiJoin` | Filtering join | Often appears when applying filters from a related table; can be good, but expensive if it forces large `Distinct`. |
| `CrossJoin` | Cartesian product | Almost always demands strong filters on both sides or a redesign (TREATAS/relationship). |

### Iteration / Row-by-Row

| Operator (examples) | Meaning | Practical notes |
|---|---|---|
| `CallbackDataID` | Row context callback | Strong signal of row-by-row execution; typically triggered by iterators and context transition patterns. |
| `IterPhyOp`, `Iterator`, `Spool_Iterator` | Iterator-like execution | Investigate `SUMX/AVERAGEX/FILTER` and nested `CALCULATE` patterns. |

### Materialization / Reuse

| Operator (examples) | Meaning | Practical notes |
|---|---|---|
| `Spool`, `ProjectionSpool` | Materialize intermediate | Validate intermediate size. If huge, reduce upstream cardinality; if small and reused, it may be beneficial. |
| `Cache` | Cached sub-result | Usually good. If cache is huge, check why the cached set is so large. |

## DirectQuery Signals (When to Consider DQ)

Even when you’re looking at DAX plans, DirectQuery can be involved. Consider DQ when:

- The tool shows `DirectQueryEnd` events (if surfaced in your result/logs).
- The physical plan contains operators/tokens indicating external queries (names vary by engine).
- You observe long Storage Engine time with few VertiPaq scan operators and repeated “remote” activity.

Recommended guidance:

- **Treat the SQL as a separate plan.** DAX plan issues can be “fine” while the generated SQL is slow (missing indexes, non-sargable predicates).
- **Reduce roundtrips.** Iterators with DQ can cause repeated remote queries (worst case: N queries for N rows).
- **Push filters early.** Ensure the filter context is expressible for pushdown to the source.

## Common Anti-Patterns

### 1. Row-by-Row Processing (CallbackDataID)

```xml
<PhysicalQueryPlan>
  ...
  <CallbackDataID>...</CallbackDataID>
  ...
</PhysicalQueryPlan>
```

**Cause**: Iterator functions (SUMX, FILTER, AVERAGEX) forcing row context evaluation
**Impact**: Orders of magnitude slower than batch operations
**Fix**:

- Use CALCULATE with filter context instead of iterators
- Pre-aggregate with SUMMARIZE/SUMMARIZECOLUMNS
- Use variables to cache intermediate results

### 2. Missing Filter Pushdown

**Symptom**: Large `Scan_Vertipaq` followed by late `Filter_Vertipaq`
**Cause**: Filters not in optimal position for engine optimization
**Fix**:

- Move filters to CALCULATE's filter arguments
- Use KEEPFILTERS to preserve filter context
- Avoid complex filter expressions that block pushdown

### 3. Expensive Context Transition

**Symptom**: High estimated rows on inner iterators
**Cause**: CALCULATE inside row context forcing context transition per row
**Fix**:

- Pre-aggregate using SUMMARIZE or variables
- Use ALLSELECTED instead of ALL when possible
- Consider restructuring DAX to avoid nested CALCULATE

### 4. Cardinality Explosion (CrossJoin)

**Symptom**: `CrossJoin` operator with large tables
**Cause**: Many-to-many relationships or missing filters
**Fix**:

- Add appropriate filters before join
- Review relationship cardinality
- Consider using TREATAS for virtual relationships

## Common False Positives (Don’t Over-Flag)

- **`Spool` present**: can be an optimization for reuse. Only treat it as a problem if the spooled set is large or repeated unnecessarily.
- **`Cache` present**: typically good; the question is why the cached set is large or why reuse is limited.
- **Iterators in logical plan**: not always slow; they become problematic when they force `CallbackDataID` or large nested loops.

## Reading Query Plans

1. **Start with Logical Plan**: Understand the DAX operations
2. **Check Physical Plan**: See how the engine executes
3. **Look for Red Flags**: CallbackDataID, large Spools, CrossJoins
4. **Check Cardinality**: Estimated rows at each stage
5. **Trace Data Flow**: Follow data from scans to final result

## Suggested Output Format (For Consistent, Actionable Guidance)

When reporting findings, prefer deterministic structure:

1. **Summary**: one sentence on where time/memory likely goes.
2. **Key Signals**: 3–6 bullets quoting plan tokens (operators/attributes) that support the conclusion.
3. **Likely Root Causes**: 1–3 bullets (filters blocked, relationship/cardinality, iterator/context transition, DQ roundtrips).
4. **Recommendations**: 1–5 concrete changes, ordered by impact and safety.
5. **Questions** (optional): 1–2 model-context questions needed to choose between fixes.

## Worked Examples (Minimal)

### Example A: Row-by-row iterator forcing callbacks

**Symptoms**
- Physical plan contains `CallbackDataID`
- Logical plan contains iterators like `SUMX`/`AVERAGEX`

**Likely cause**
- Iterator forces row context evaluation; nested `CALCULATE` may be triggering context transition per row.

**Recommendation ideas**
- Replace iterator with a set-based aggregation using `CALCULATE` and explicit filters.
- Pre-aggregate with `SUMMARIZECOLUMNS` and aggregate the smaller set.
- Use variables to avoid recomputing the same intermediate table.

### Example B: CrossJoin / join explosion

**Symptoms**
- Physical plan contains `CrossJoin`
- Large estimated row counts before/after the join

**Likely cause**
- Missing filters or many-to-many/bi-directional relationship behavior inflates the intermediate set.

**Recommendation ideas**
- Add/strengthen filters before join (or rewrite to avoid `CrossJoin`).
- Use `TREATAS` or a relationship-based approach to avoid cartesian products.
- Validate relationship cardinality and filter direction; reduce high-cardinality join keys where possible.

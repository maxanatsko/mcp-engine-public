# VertiPaq Optimization Guide (`run_query`) (Pro)

This guide explains how to use VertiPaq storage stats to identify model bloat and performance issues.

## Related Tools

- `run_query`: Storage stats per column (`operation: "vertipaq"` - dictionary/data sizes, encoding, segments)
  - Filters Power BI system artifacts by default; set `include_system_artifacts: true` to inspect raw `RowNumber-*` or auto date table stats
- `run_query`: Validate performance improvements after changes (`operation: "analyze"` - Pro for query plan)
- `list_model`: Inspect visibility, types, and column count (`operation: "list"`, `spec: { type: "columns" }`)

## Basic Usage

All tables summary:

```json
{ "operation": "vertipaq" }
```

One table:

```json
{ "operation": "vertipaq", "spec": { "table": "Sales" } }
```

## Cardinality (Optional, Expensive)

`include_cardinality=true` runs DAX to compute exact distinct counts per column.

```json
{
  "operation": "vertipaq",
  "spec": {
    "table": "Sales",
    "include_cardinality": true,
    "cardinality_batch_size": 25
  }
}
```

Guidance:

- Use only when you need exact counts.
- Expect 100–500ms per column (can be tens of seconds for wide models).

## What to Look For

### Large dictionary size

Often indicates high-cardinality text columns. Remediations:

- Remove unused columns.
- Replace long text keys with surrogate integers (where appropriate).
- Split columns or normalize into dimensions.

### Large data size / many segments

Remediations:

- Reduce row count (filter at source, aggregation tables, incremental patterns).
- Reduce precision / change data types where safe.
- Remove columns not used in visuals/measures.

### Encoding hints

Use encoding + size patterns to spot columns that don't compress well (e.g., GUIDs, free-form strings).

## Remediation Playbook

1. Identify top offenders by size (table then columns).
2. Determine whether each column is required for:
   - relationships
   - slicers/visuals
   - measure logic
3. Remove/replace columns and validate:
   - correctness (via `run_query` with `operation: "execute"`)
   - performance (via `run_query` with `operation: "analyze"`)

## Alternatives to Pure VertiPaq

### User-Defined Aggregations

For large fact tables where full import isn't practical, consider aggregation tables:

**What they are:**
- Pre-aggregated summary tables at a higher grain (e.g., daily instead of transactional)
- Power BI automatically routes queries to aggregations when possible
- Falls back to detail table for drill-through or filters below aggregation grain

**When to use:**
- Fact tables with hundreds of millions of rows
- Common queries aggregate to a predictable grain
- Acceptable trade-off between storage and query flexibility

**Implementation pattern:**
1. Create aggregation table with summarized data
2. Configure aggregation mappings in Power BI Desktop
3. Hide the aggregation table from report authors
4. Test with representative queries to verify automatic routing

### Direct Lake Mode (Microsoft Fabric)

Direct Lake is a storage mode available in Microsoft Fabric that bypasses VertiPaq entirely:

**How it works:**
- Semantic model reads directly from Delta tables in OneLake
- No data copy or import refresh required
- Queries run against columnar Parquet files in the lakehouse

**When to consider Direct Lake:**
- Data already exists in Fabric Lakehouse or Warehouse
- Very large datasets that exceed practical import sizes
- Need for near-real-time data without refresh latency
- Fabric capacity is available

**Trade-offs:**
- Some DAX patterns may cause "fallback" to DirectQuery (slower)
- Requires careful measure design to stay in Direct Lake mode
- Not all Power BI features are supported
- Dependent on Fabric capacity performance

**Checking for fallback:**
- Monitor Fabric capacity metrics for Direct Lake fallback counts
- Simplify iterators and complex calculations that trigger fallback
- Consider hybrid approaches: Direct Lake for recent data, aggregations for historical

### Composite Models

Combine Import, DirectQuery, and Direct Lake tables in a single model:

- Import small dimension tables for best performance
- DirectQuery or Direct Lake for large fact tables
- Use aggregations to optimize common query patterns

**Note**: Composite models require careful relationship and performance testing.

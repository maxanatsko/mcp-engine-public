# Query Performance Guide (`run_query`)

This guide explains how to measure DAX query performance and interpret results using `run_query` with `operation: "analyze"`.

## Related Tools

- `run_query`: Run repeated tests and return Storage Engine / Formula Engine metrics (`operation: "analyze"`)
- `run_query`: Validate correctness and result shape (`operation: "execute"`)
- `dax-query-plan-reference.md`: Query plan operator reference

## Basic Usage

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

Recommended defaults:

- `runs`: 3–5 for a quick signal; 10+ for stabilizing noisy results
- Desktop: set `clear_cache: true` when comparing alternatives, but call the runs cold-cache only when the response has no `clear_cache failed` limitation. If clearing fails, resolve it or start a separate run with `clear_cache: false`, discard run 1, and compare the median of runs 2..N; do not mix successful and failed cache-clear attempts.
- Power BI Service XMLA: cache clearing is unavailable. With `runs >= 3`, the response records this limitation; discard run 1 as warm-up and compare the median of runs 2..N from `all_durations_ms`. Keep the query, run count, and connection kind identical between alternatives, and label the result warm-cache. The Storage Engine / Formula Engine metrics come from discarded run 1, so use them only for diagnostic triage, not as the warm baseline or before/after evidence.
- `use_xevents`: true when available (best metrics)

## Interpreting Results

The tool returns multiple runs and computed statistics, typically including:

- total duration
- Storage Engine (SE) duration / query count
- Formula Engine (FE) duration

Common heuristics:

- High SE time: push filters earlier, reduce scanned rows, improve relationships/keys, reduce cardinality.
- High FE time: simplify iterators, reduce row context, avoid repeated CALCULATE patterns, use variables.

## Include Query Plan (Pro parameter)

Set `include_query_plan=true` in spec to capture logical/physical plans:

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

Notes:

- Query plans can be very large (thousands of lines).
- Use plans to spot expensive operators and unexpected scans/joins.

## Troubleshooting

- If Extended Events are unavailable, the tool can fall back to DMV-based metrics with less fidelity.
- For very fast queries (<10ms), measurement noise can dominate; increase `runs`.

---
name: mcp-engine-query
description: Use when writing or fixing DAX query text, choosing between run_query operations, validating results, testing RLS access for roles, or discovering schema before querying. For diagnosing why a query is slow, use mcp-engine-dax-performance; for wrong values, unexpected blanks, or inflated totals, use mcp-engine-dax-debugging; to persist logic as measures, use mcp-engine-semantic-authoring.
---

# PBI Query

Query authoring and execution through `run_query`.

## Query workflow

1. Discover schema first: `list_model` `{ "operation": "list", "spec": { "type": "tables" } }` — one call per type (`tables`, `columns`, `measures`, `relationships`; `spec.type` takes exactly one value) — or `{ "operation": "search", "spec": { "query": "<name>", "mode": "name" } }` for a named object. Never invent table, column, or measure names.
2. Pick the `run_query` operation:
   - `execute` — results and validation queries.
   - `test_access` — role-by-role RLS validation (requires top-level `query` and `spec.roles`).
   - `analyze` and `vertipaq` belong to performance work — hand off to `mcp-engine-dax-performance`.
3. Keep the first `execute` scoped: `TOPN` of at most 100 rows with an explicit `ORDER BY`, or a targeted aggregation. Expand only after the shape is confirmed correct.
4. Treat `null` in results as DAX `BLANK` semantics, and `truncated: true` as a row cap, not missing data.

## Author safely

- Read [dax-query-guide](references/dax-query-guide.md) before writing new or corrected query text — quoting, naming, and structure rules live there.
- Add `ORDER BY` to any multi-row output so results are stable and comparable across runs.

## Report results

After query work, report:

1. The exact DAX text run and the `run_query` operation used.
2. Row count returned and whether results were truncated.
3. The answer to the user's question, stated separately from raw output.

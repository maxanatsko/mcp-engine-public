---
name: mcp-engine-dax-performance
description: Use when a DAX query, measure, or visual is slow, when interpreting analyze timings, Storage Engine / Formula Engine splits, or query plans, when VertiPaq storage size or cardinality drives cost, or when the user wants a tuning pass or before/after benchmark. For wrong values, use mcp-engine-dax-debugging; for writing new queries, use mcp-engine-query; for a whole-model assessment, use mcp-engine-model-quality.
---

# PBI DAX Performance

Optimization of correct-but-slow queries: measure, change one thing, re-measure. Never claim a speedup without an identical before/after comparison.

## Optimization loop

1. **Baseline** — `run_query` `{ "operation": "analyze", "query": "<slow query>", "spec": { "runs": 5, "clear_cache": true } }`. On Desktop, call this cold-cache only when the response has no `clear_cache failed` limitation; otherwise fix cache clearing or start a separate warm-cache run with `clear_cache: false`, discard run 1, and use the median of runs 2–5. On Service XMLA, cache clearing is unavailable: record that limitation, discard run 1 as warm-up, and use the median of runs 2–5 as the warm-cache baseline. The Service Storage Engine / Formula Engine split comes from discarded run 1, so use it only as diagnostic triage, not as baseline or before/after evidence. Record run variance. For very fast queries (<10ms) increase `runs` — noise dominates.
2. **Triage** — classify with [performance-remediation-playbook](references/performance-remediation-playbook.md) section A: SE-heavy, FE-heavy, or model-size pressure. Interpretation rules for timings live in [query-performance-guide](references/query-performance-guide.md).
3. **Capture the plan when operators matter** — add `"include_query_plan": true` to spec (Pro); decode unfamiliar operators with [dax-query-plan-reference](references/dax-query-plan-reference.md).
4. **Check storage when SE-heavy or memory-bound** — `run_query` `{ "operation": "vertipaq", "spec": { "table": "<table>", "include_cardinality": true } }`; remediation options in [vertipaq-optimization-guide](references/vertipaq-optimization-guide.md).
5. **Hypothesize and change one thing** — the playbook's symptom table maps each diagnosis to its fix. Route the edit: measure rewrite → `mcp-engine-semantic-authoring`; model shape, columns, or relationships → `mcp-engine-schema-authoring` (or `mcp-engine-refactoring` when consumers are involved). If a named skill is not installed, make the edit from the tool's `inputSchema` and the model's conventions, or ask the user to add that skill.
6. **Re-measure identically** — same query text, connection kind, `runs`, and cache treatment. On Service XMLA, again discard run 1 and compare the median of the remaining identical warm runs. Keep the change only when the comparison shows it; revert otherwise and try the next hypothesis.
7. **Protect the win** — pin the result with a connection-appropriate `performance_budget` definition. On Desktop with verified cache clearing, validate `manage_tests` `{ "operation": "validate", "spec": { "id": "<test-id>", "name": "<name>", "type": "performance_budget", "spec": { "query": "<tuned query>", "runs": { "cold": 1, "warm": 3 } }, "assert": { "kind": "performance", "total_ms": { "max": <cold-and-warm-budget> } } } }`. On Service XMLA, validate `{ "operation": "validate", "spec": { "id": "<test-id>", "name": "<name>", "type": "performance_budget", "spec": { "query": "<tuned query>", "runs": { "cold": 1, "warm": 3 } }, "assert": { "kind": "performance", "warm_ms": { "max": <warm-budget> } } } }`: the unsupported `cold` slot is the unasserted warm-up, while `warm_ms.max` covers only the following three runner timings. Derive the Service threshold from the maximum post-warm-up timing plus approved headroom, not from the discarded first run or the analyze median. Then call `{ "operation": "put" }` with the same complete `spec`, followed by `{ "operation": "run" }` so regressions surface in test runs. Never apply a Desktop `total_ms` budget to Service results.

## Guardrails

- One change per iteration; a two-change iteration cannot attribute its result.
- On Desktop, use `clear_cache: true` when comparing alternatives and verify it succeeded; if not, resolve the failure or start a new run with `clear_cache: false`, discard run 1, and compare the remaining warm samples. On Service XMLA, state that cache clearing is unavailable and discard run 1 before comparing warm samples. Never mix successful and failed cache-clear attempts or label unavailable clearing as cold-cache results.
- Optimize the model shape before micro-optimizing DAX when the playbook triage points at structure.
- Plans and traces can be large: summarize findings, do not paste raw dumps to the user.

## Report results

After a tuning pass, report:

1. The query tuned and the baseline numbers (total, SE/FE split, runs).
2. Each hypothesis tried, the change made, and its measured effect.
3. The final before/after comparison for kept changes.
4. Changes reverted and why.
5. The `performance_budget` test added, with its id.

## References

- [query-performance-guide](references/query-performance-guide.md) — analyze usage, timing interpretation, run statistics
- [dax-query-plan-reference](references/dax-query-plan-reference.md) — plan operator definitions
- [vertipaq-optimization-guide](references/vertipaq-optimization-guide.md) — storage footprint and cardinality remediation
- [performance-remediation-playbook](references/performance-remediation-playbook.md) — triage, symptom-to-fix table, golden baseline queries

---
name: mcp-engine-dax-debugging
description: Use when a Power BI measure or query returns wrong values, unexpected blanks, inflated or duplicated totals, a total row that disagrees with its detail rows, a slicer that has no effect, or numbers that disagree with the source system. For slow-but-correct queries, use mcp-engine-dax-performance; for a whole-model assessment, use mcp-engine-model-quality.
---

# PBI DAX Debugging

Correctness debugging for wrong numbers. The full loop lives in [dax-debugging-workflow](references/dax-debugging-workflow.md) — follow it step by step; stay read-only until the root cause is proven and reconciles the discrepancy numerically.

## Workflow

1. **State the discrepancy** — measure, exact filter context, actual vs. expected, and where the expectation comes from.
2. **Reproduce minimally** — `run_query` `{ "operation": "execute", "query": "<CALCULATETABLE(ROW(...), <filters>) reproduction query>" }` that rebuilds the wrong number outside the report.
3. **Establish ground truth** — agree on the business definition and grain, then compute that same aggregation from the most primitive fields while bypassing the suspect measure. Use raw `SUM` only for additive metrics; preserve `DISTINCTCOUNT`, averages, ratios, semi-additive balance logic, and other non-additive semantics. Where the two diverge decides the branch: measure defect, model/data defect, or definition mismatch.
4. **Decompose** — fetch the expression with `list_model` `{ "operation": "list", "spec": { "type": "measures", "include_expression": true } }` and binary-search its parts, one `ROW` probe per sub-expression, recursing into referenced measures by extracting the `[Measure Name]` references from the expression itself (`manage_dependencies` `used_by` lists consumers, not inputs).
5. **Inspect the filter path** — `list_model` relationships listing for every hop between filter table and fact: existence, active state, direction, cardinality, key data types; duplicate-key and orphan-key probes from the guide.
6. **Match the symptom** — the guide's symptom table (inflated, blank, total mismatch, dead slicer, source mismatch) names the deciding probe for each cause.
7. **Rule out security and masking** — `run_query` `{ "operation": "test_access", "query": "<reproduction query>", "spec": { "roles": ["<role>"] } }` under the affected role; check masking flags before calling data wrong.
8. **Close numerically, then fix and protect** — the explanation must reconcile actual to expected arithmetically; the fix routes to the authoring skill (if it is not installed, apply the fix from the tool's `inputSchema` or ask the user to add it), and a `manage_tests` `measure_assertion` or `referential_integrity` test pins the corrected behavior.

## Guardrails

- No fixes during diagnosis — a change that "makes it right" without an explained cause hides the bug.
- One variable per probe; keep a written trail of every probe and result.
- Keep probes aggregated; never dump raw rows that may contain sensitive values.
- A total-vs-detail mismatch can be correct by design for non-additive measures — verify intent before declaring a defect.

## Report results

After a debugging session, report:

1. The discrepancy: measure, context, actual vs. expected.
2. The root cause with probe evidence and the reconciling arithmetic.
3. Its class: measure defect, model defect, data defect, security behavior, or definition mismatch.
4. The fix applied or recommended, with the re-run verification result.
5. The regression test added, with its id.

## References

- [dax-debugging-workflow](references/dax-debugging-workflow.md) — the step-by-step loop, probe queries, and symptom table

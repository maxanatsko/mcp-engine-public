---
name: mcp-engine-query
description: Write, review, scope, execute, and analyze DAX queries in Power BI models. Use when creating or fixing DAX queries, deciding between run_query operations, investigating performance or VertiPaq issues, testing RLS access, or discovering schema before querying.
---

# PBI Query

Use this skill for query authoring and query analysis work. Read only the bundled references needed for the current task.

## Query workflow

1. Inspect schema before guessing table, column, or measure names.
2. Decide whether the task is execution, performance analysis, VertiPaq inspection, or access testing.
3. Keep the first query scoped and small enough to inspect safely.
4. Expand only after the initial query shape is correct.

## Choose the operation

- Use `execute` when the user needs query results or a validation query.
- Use `analyze` when the user needs timings, operator-level diagnosis, or performance findings.
- Use `vertipaq` when the user needs storage footprint, table size, or cardinality clues.
- Use `test_access` when the user needs role-by-role access validation.

## Author safely

- Read [dax-query-guide](references/dax-query-guide.md) before writing new or corrected query text.
- Prefer schema discovery with `list_model` before inventing names or relationships.
- Keep the default query scoped with filters, `TOPN`, or targeted grouping.
- Add ordering for multi-row outputs so results are stable and readable.
- Treat nulls in result rows as query semantics until the model proves otherwise.

## Diagnose performance

- Read [query-performance-guide](references/query-performance-guide.md) before interpreting timings or trace output.
- Read [dax-query-plan-reference](references/dax-query-plan-reference.md) when a plan contains unfamiliar operators.
- Read [vertipaq-optimization-guide](references/vertipaq-optimization-guide.md) when the issue looks like storage bloat or high-cardinality pressure.
- Read [performance-remediation-playbook](references/performance-remediation-playbook.md) when the user wants a remediation sequence instead of isolated findings.

## Reference selection

- Read [dax-query-guide](references/dax-query-guide.md) first for syntax, quoting rules, and DAX query structure.
- Read [query-performance-guide](references/query-performance-guide.md) for timings, runs, and performance interpretation.
- Read [dax-query-plan-reference](references/dax-query-plan-reference.md) for plan operator definitions.
- Read [vertipaq-optimization-guide](references/vertipaq-optimization-guide.md) for storage optimization guidance.
- Read [performance-remediation-playbook](references/performance-remediation-playbook.md) for end-to-end tuning workflow.

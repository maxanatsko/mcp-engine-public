---
name: mcp-engine-schema-authoring
description: Use when creating, updating, renaming, or deleting tables, columns, calculated columns, relationships, hierarchies, calendars, or partitions, or changing refresh strategy and incremental refresh policy. For measures, calculation groups, or named expressions, use mcp-engine-semantic-authoring; for RLS roles or perspectives, use mcp-engine-security-governance; for multi-object refactors or renames with downstream consumers, use mcp-engine-refactoring.
---

# PBI Schema Authoring

Physical model structure work through `manage_schema`. Keep edits explicit, ordered, and dependency-aware.

## Safe edit order

1. Inspect current state: `list_model` `{ "operation": "list", "spec": { "type": "tables" } }`, then repeat for `columns` and `relationships` (`spec.type` takes exactly one value per call).
2. Before renaming or deleting, check consumers: `manage_dependencies` `{ "operation": "used_by", "spec": { "target": { "type": "column", "table": "<table>", "name": "<column>" } } }` (`spec.target` is required on every call, with the object's singular type); `{ "operation": "summary", "spec": { "target": ... } }` to gauge impact breadth.
3. For multi-object or risky edits, pin a rollback point first: `manage_model_changes` `{ "operation": "pin_checkpoint", "name": "before-<edit>" }` (`name` is required), and prefer `dry_run: true` on the write.
4. Apply the `manage_schema` operation (`create_table`, `update_column_properties`, `create_calc_column`, `create_relationship`, `create_hierarchy`, `create_partition`, `refresh_partition`, …) with per-item identifiers inside each bulk item.
5. Validate after the edit: a scoped `run_query` `{ "operation": "execute", "query": "<validation query>" }` that exercises the changed relationship, column, or partition.

## Branch by task

- Tables, columns, calculated columns, visibility, formatting, date-table marking: [column-and-table-authoring-guide](references/column-and-table-authoring-guide.md).
- Relationships — re-check filter direction, inactive relationships, and ambiguity before applying: [relationships-guide](references/relationships-guide.md).
- Hierarchies: [hierarchies-guide](references/hierarchies-guide.md). Calendars and date tables: [calendar-guide](references/calendar-guide.md).
- Partitions and refresh scope: [partitions-refresh-guide](references/partitions-refresh-guide.md). Incremental policy design: [incremental-refresh-policy-guide](references/incremental-refresh-policy-guide.md).

## Guardrails

- Confirm destructive changes and refresh-impacting edits with the user before execution.
- Keep schema changes separate from semantic rewrites unless the task explicitly spans both.

## Report results

After schema work, report:

1. Each operation applied, with table and target names.
2. Whether `dry_run` or `transaction` was used and the outcome.
3. Dependency-check results for renamed or deleted objects.
4. The validation query run and its result.
5. The checkpoint or transaction available for rollback.

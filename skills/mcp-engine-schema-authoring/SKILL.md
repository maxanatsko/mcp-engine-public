---
name: mcp-engine-schema-authoring
description: Plan and execute Power BI physical model changes. Use when creating or updating tables, columns, relationships, hierarchies, calendars, partitions, or refresh strategy, especially when changes may affect dependencies, data shape, or refresh behavior.
---

# PBI Schema Authoring

Use this skill for physical model structure work. Keep schema edits explicit, ordered, and dependency-aware.

## Safe edit order

1. Inspect the current schema and dependency surface before editing.
2. Choose the correct schema area: table or column shape, relationships, hierarchies, calendars, or partitions.
3. Validate downstream impact before destructive changes.
4. Prefer a dry run or checkpoint when the change touches many objects.
5. Re-check refresh implications after the structural edit lands.

## Branch by task

### Tables and columns

- Read [column-and-table-authoring-guide](references/column-and-table-authoring-guide.md) for create, update, rename, and delete flows.
- Use it when the user changes schema shape, visibility, formatting, or date-table marking.

### Relationships

- Read [relationships-guide](references/relationships-guide.md) before adding, removing, or changing relationships.
- Re-check filter direction, inactive relationships, and ambiguity before applying the change.

### Hierarchies and calendars

- Read [hierarchies-guide](references/hierarchies-guide.md) for user-facing browsing structure.
- Read [calendar-guide](references/calendar-guide.md) when the task involves date tables or time-intelligence support.

### Partitions and refresh

- Read [partitions-refresh-guide](references/partitions-refresh-guide.md) for partition design, refresh scope, and refresh safety.
- Read [incremental-refresh-policy-guide](references/incremental-refresh-policy-guide.md) when a fact table needs incremental policy design or changes.

## Guardrails

- Check dependencies before deleting or renaming objects that may be referenced elsewhere.
- Keep schema changes separate from semantic rewrites unless the task explicitly requires both.
- Confirm destructive changes and large refresh-impacting edits before execution.
- Re-run targeted validation queries after relationship or partition changes.

## References

- [column-and-table-authoring-guide](references/column-and-table-authoring-guide.md)
- [relationships-guide](references/relationships-guide.md)
- [hierarchies-guide](references/hierarchies-guide.md)
- [calendar-guide](references/calendar-guide.md)
- [partitions-refresh-guide](references/partitions-refresh-guide.md)
- [incremental-refresh-policy-guide](references/incremental-refresh-policy-guide.md)

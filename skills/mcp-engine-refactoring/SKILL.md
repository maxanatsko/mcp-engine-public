---
name: mcp-engine-refactoring
description: Use when renaming, consolidating, restructuring, or batch-editing Power BI model objects that have downstream consumers — measure consolidation, table or column renames, splitting a flat table toward a star schema, moving repeated logic into calculation groups, or executing a model-quality remediation backlog. For a single-object edit with no consumers, use mcp-engine-schema-authoring or mcp-engine-semantic-authoring directly.
---

# PBI Refactoring

Safe multi-object change orchestration: dependencies, baselines, checkpoints, staged edits, verification, rollback. The phases and their order live in [refactoring-workflow](references/refactoring-workflow.md) — follow it phase by phase; do not edit before its phases 1-4 are complete.

## Workflow

1. **Scope** — object list and change class, confirmed with `list_model` `{ "operation": "list", "spec": { "type": "<type>" } }` / `{ "operation": "search", "spec": { "query": "<name>", "mode": "name" } }`.
2. **Baseline** — 3-5 representative `run_query` `{ "operation": "execute", "query": "<aggregate>" }` calls recorded verbatim, or durable `manage_tests` `regression_snapshot` / `measure_assertion` tests that pass before any edit.
3. **Impact** — `manage_dependencies` `{ "operation": "used_by", "spec": { "target": { "type": "<type>", "table": "<table>", "name": "<name>" } } }` per object (`spec.target` is required on every call) plus `list_model` expression search (`mode: "dax"` and `"m"`), repeated until no new consumers appear; blast radius reported to the user first.
4. **Safety net** — `manage_model_changes` `{ "operation": "pin_checkpoint", "name": "before-<goal>" }`; multi-object edits staged as a named changeset (`create_changeset` returns the `changeset_id` that `add_to_changeset`, `preview_changeset`, and `apply_changeset` require).
5. **Execute in order** — create replacements → update consumers → rename or delete originals; `dry_run: true` on the first attempt of each operation shape.
6. **Verify** — baselines re-run and compared exactly; `diff_transaction` (with the `transaction_id` from `list_transactions`) reviewed against the plan; old names searched one final time.
7. **Decide** — keep, or `rollback_transaction` / `restore_checkpoint` on any unintended mismatch.

Recipes for the common cases (rename with consumers, measure consolidation, calc-group relocation, hygiene sweep) are in the workflow guide.

## Guardrails

- One refactor goal per run; never mix renames with logic changes.
- Dependency analysis cannot see report visuals or external consumers that bind by name — warn the user on every rename of a visible object.
- An intended value change discovered mid-refactor must be confirmed by the user, never silently absorbed.
- The actual edits follow the authoring guides of `mcp-engine-schema-authoring` and `mcp-engine-semantic-authoring`; this skill owns the sequence around them. If those skills are not installed, make the edits from the tools' `inputSchema` or ask the user to add them.

## Report results

After a refactor, report:

1. Scope, change class, and final consumer count per object.
2. Checkpoint and changeset ids created.
3. Each edit applied, in order.
4. Baseline comparison outcome: identical, intended differences confirmed, or rolled back.
5. Remaining risks (report-layer renames, deferred deletions) and the rollback point still available.

## References

- [refactoring-workflow](references/refactoring-workflow.md) — the phase-by-phase workflow, ordering rules, and recipes

---
name: mcp-engine-testing-changes
description: Use when creating or running model tests, applying test packs, capturing baselines or snapshots, exporting test results, checking dependency impact before a refactor, or using checkpoints, changesets, undo, redo, and rollback. For assessing overall model quality, use mcp-engine-model-quality; for authoring the changes themselves, use mcp-engine-schema-authoring or mcp-engine-semantic-authoring; for orchestrating a full multi-object refactor around these safety tools, use mcp-engine-refactoring.
---

# PBI Testing Changes

Proof that model changes are safe, reproducible, and reversible, through `manage_tests`, `manage_model_changes`, and `manage_dependencies`.

## Preferred workflow

1. Inspect what exists: `manage_tests` `{ "operation": "list" }`; `manage_model_changes` `{ "operation": "list_checkpoints" }` and `{ "operation": "list_transactions" }`.
2. Validate definitions before saving: `manage_tests` `{ "operation": "validate", "spec": { <complete test definition> } }`, then save the same definition with `{ "operation": "put", "spec": { <same definition> } }` — both operations require the definition (`put` takes `spec` or bulk `items`; payload shapes are in the unit-testing guide). Never save a large generated batch unvalidated.
3. Run the smallest useful scope first: `{ "operation": "run", "spec": { "tags": [...] } }` or `{ "operation": "run", "spec": { "test_ids": [...] } }` before a full-suite run.
4. Export when the user needs a shareable result: `{ "operation": "export", "spec": { "format": "markdown" } }`, or `export_tests` for a portable definition bundle.
5. Before broad edits, pin a rollback point: `manage_model_changes` `{ "operation": "pin_checkpoint", "name": "before-<edit>" }` (`name` is required), or stage the batch as a changeset (`create_changeset` with a `name` → `add_to_changeset` → `preview_changeset` → `apply_changeset`, threading the returned `changeset_id`).

## Branch by task

- Test types, canonical payload shapes, packs, baselines, export: [unit-testing-guide](references/unit-testing-guide.md).
- Checkpoints, changesets, transaction history, `diff_transaction`, `rollback_transaction`, undo/redo: [model-changes-guide](references/model-changes-guide.md).
- Impact before rename or delete: `manage_dependencies` `{ "operation": "used_by", "spec": { "target": { "type": "<type>", "table": "<table>", "name": "<name>" } } }` (`spec.target` is required) — [dependencies-guide](references/dependencies-guide.md). Use the output to narrow test scope after a planned change.

## Guardrails

- Preserve stable model identity behavior when work touches persisted tests or baselines.
- Apply test packs with `{ "operation": "packs_apply", "spec": { "pack_id": "<pack>" }, "dry_run": true }` first; apply for real only after reviewing the preview.
- Pair risky edits with a checkpoint or changeset before, and a targeted test run after.

## Report results

After testing or change-tracking work, report:

1. Tests created, changed, or run, with pass/fail counts and the names of failing tests.
2. Validation outcomes for saved definitions.
3. Baselines or snapshots captured or compared.
4. Checkpoint, changeset, or transaction ids available for rollback.

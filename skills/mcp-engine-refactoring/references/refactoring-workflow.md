# Model Refactoring Workflow

Use this workflow to change multiple model objects — or any object with downstream consumers — without breaking measures, reports, refresh, or security. It orchestrates the dependency, checkpoint, changeset, and testing tools into the sequence experienced modelers follow manually.

Single-object edits with no consumers do not need this workflow; use the relevant authoring guide directly.

## Related Tools and Resources

- Impact analysis: `manage_dependencies` (`used_by`, `summary`, `graph`)
- Safety net and staging: `manage_model_changes` (`pin_checkpoint`, `create_changeset`, `preview_changeset`, `apply_changeset`, `diff_transaction`, `rollback_transaction`, `restore_checkpoint`)
- Baselines and regression proof: `manage_tests` (`put`, `run`, `snapshot`), `run_query` (`execute`)
- Edits: `manage_schema`, `manage_semantic`, `manage_security` per the relevant authoring guide
- Related guides: `dependencies-guide.md (from the mcp-engine-testing-changes skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`, `model-changes-guide.md (from the mcp-engine-testing-changes skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`, `unit-testing-guide.md (from the mcp-engine-testing-changes skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`, `modeling-best-practices-guide.md (from the mcp-engine-semantic-authoring skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`, `semantic-layer-style-guide.md (from the mcp-engine-semantic-authoring skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`

## Operating Rules

- Never edit before phases 1-4 are complete. The order exists because each phase produces the evidence the next one needs.
- One refactor goal per run. Do not mix a rename sweep with logic changes; separate runs keep diffs and rollback meaningful.
- Model-internal dependency analysis cannot see report visuals, external tools, or downstream datasets that reference objects by name. Treat every rename of a visible object as a potential report-breaking change and say so to the user.
- If any phase produces a surprise (unknown dependent, failing baseline), stop and re-plan instead of pushing through.

## Workflow

### Phase 1 — Scope

1. Write down the refactor goal and the explicit object list (tables, columns, measures, roles).
2. Confirm each object exists and get its current definition: `list_model` `{ "operation": "list", "spec": { "type": "measures", "include_expression": true } }` (or the matching type), or `{ "operation": "search", "spec": { "query": "<name>", "mode": "name" } }`.
3. Classify the change: rename, consolidate, relocate logic, restructure, or hygiene sweep. The recipes below adjust the edit order per class.

### Phase 2 — Baseline

Capture the current behavior you promise not to change:

1. Pick 3-5 representative aggregates that exercise the affected objects (core measure totals, one sliced breakdown, one row count).
2. Run each with `run_query` `{ "operation": "execute", "query": "<the aggregate>" }` and record query text and results verbatim.
3. For refactors touching many measures or a fact table, prefer durable baselines: build `regression_snapshot` or `measure_assertion` definitions (payload shapes in `unit-testing-guide.md (from the mcp-engine-testing-changes skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`), save them with `manage_tests` `{ "operation": "put", "spec": { <definition> } }` (or bulk `items`), then `{ "operation": "run" }` to confirm they pass before any edit.

### Phase 3 — Impact analysis

1. For every object in scope: `manage_dependencies` `{ "operation": "used_by", "spec": { "target": { "type": "<type>", "table": "<table>", "name": "<name>" } } }` — `spec.target` is required on every call; record each consumer.
2. Gauge breadth with `{ "operation": "summary", "spec": { "target": ... } }` (same required target); for tangled areas use `{ "operation": "graph", "spec": { "target": ... } }`.
3. Find expression-level references that dependency output may not cover: `list_model` `{ "operation": "search", "spec": { "mode": "dax", "query": "<object name>" } }` and `{ "operation": "search", "spec": { "mode": "m", "query": "<object name>" } }`.
4. Add every discovered consumer to the scope list. Repeat until no new consumers appear.
5. Report the blast radius to the user before editing, including the report-layer caveat for renames of visible objects.

### Phase 4 — Safety net

1. Pin a named rollback point: `manage_model_changes` `{ "operation": "pin_checkpoint", "name": "before-<refactor-goal>" }` (`name` is required).
2. For multi-object edits, stage as a changeset: `{ "operation": "create_changeset", "name": "<refactor-goal>" }` returns a `changeset_id`; add each planned edit with `{ "operation": "add_to_changeset", "changeset_id": "<id>", "tool_name": "manage_semantic", "tool_args": { ... } }` and review with `{ "operation": "preview_changeset", "changeset_id": "<id>" }` before anything executes. See `model-changes-guide.md (from the mcp-engine-testing-changes skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)` for the full payload shapes.
3. For direct edits, use `dry_run: true` on the first attempt of each operation shape.

### Phase 5 — Ordered execution

Apply edits in dependency-safe order:

1. Create replacements before touching originals (new measure, new table, new relationship).
2. Update consumers to point at the replacement (`update_measure`, `update_calc_group`, role filters) and re-validate them.
3. Only then rename or delete originals. After any rename, re-run the phase 3 expression search for the old name — remaining hits are consumers you missed.
4. Keep batches small enough to review: apply the changeset with `{ "operation": "apply_changeset", "changeset_id": "<id>" }`, or run direct edits in reviewed groups.

### Phase 6 — Verify

1. Re-run every phase 2 baseline query and compare values exactly; re-run baseline tests with `manage_tests` `{ "operation": "run" }`.
2. Review what actually changed: `manage_model_changes` `{ "operation": "list_transactions" }`, pick the transactions the refactor created, and run `{ "operation": "diff_transaction", "transaction_id": "<id>" }` for each against the plan.
3. Search for the old names one final time (`list_model` search, `mode: "dax"` and `"m"`); any hit means an incomplete refactor.

### Phase 7 — Decide

- All baselines match and the diff equals the plan: keep, and tell the user which checkpoint remains available.
- Any mismatch you did not intend: roll back with `{ "operation": "rollback_transaction", "transaction_id": "<id from list_transactions>" }` or `{ "operation": "restore_checkpoint", "checkpoint_id": "<id from pin_checkpoint or list_checkpoints>" }` — add `"confirm": true` when the client does not support confirmation prompts — then report the mismatch and re-plan.
- An intended value change (for example, a bug fixed during consolidation) must be named as such and confirmed by the user, never silently absorbed into the refactor.

## Refactor Recipes

### Rename an object with consumers

Phases 1-4, then: rename via the authoring operation → search old name in DAX and M → update remaining consumers → verify. Warn the user that report visuals and external tools using the old name break regardless of model-internal fixes.

### Consolidate duplicate measures

Pick the canonical measure with the user. Update consumers of the duplicates to reference the canonical one → hide the duplicates and mark descriptions as deprecated → verify → delete the duplicates in a later run once nothing references them.

### Relocate repeated logic into a calculation group or base measure

Create the calculation group or base measure first → re-express one consumer and validate it against its baseline value → migrate remaining consumers → delete superseded expressions. See `modeling-best-practices-guide.md (from the mcp-engine-semantic-authoring skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)` for placement decisions.

### Hygiene sweep (hide keys, descriptions, folders)

Batch metadata-only edits per table with `dry_run: true` first. Baselines are metadata listings rather than query results: capture `list_model` output for the affected types before and after, and skip phases 2 and 6 query comparisons.

## Report to the user

1. Scope and classification, with the final consumer count per object.
2. Checkpoint and changeset ids created.
3. Each edit applied, in order.
4. Baseline comparison: identical, intended differences (confirmed), or rolled back.
5. Remaining risks (report-layer renames, deferred deletions) and the rollback point that remains available.

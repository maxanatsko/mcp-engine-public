---
name: mcp-engine-testing-changes
description: Validate Power BI changes with tests and checkpoints. Use when creating or running model tests, exporting results, managing baselines or snapshots, checking dependency impact before refactors, or using checkpoints, changesets, rollback, and stable model identity.
---

# PBI Testing Changes

Use this skill when the user wants proof that model changes are safe, reproducible, or reversible.

## Preferred workflow

1. Inspect current tests, packs, baselines, or change history.
2. Validate test definitions before saving large batches.
3. Run the smallest useful test scope first.
4. Export or summarize results in the format the user needs.
5. Create a checkpoint or changeset before broad edits when rollback matters.

## Branch by task

### Tests and baselines

- Read [unit-testing-guide](references/unit-testing-guide.md) for test types, canonical payload shapes, validation, and export.
- Use it when creating tests, applying packs, capturing baselines, or exporting results.

### Change tracking and rollback

- Read [model-changes-guide](references/model-changes-guide.md) when the task involves checkpoints, changesets, transaction history, diffs, or rollback.
- Prefer a checkpoint before large refactors or batch operations.

### Dependency impact

- Read [dependencies-guide](references/dependencies-guide.md) before renaming or deleting objects with downstream consumers.
- Use dependency output to narrow test scope after a planned change.

## Guardrails

- Preserve stable model identity behavior when work touches persisted tests or baselines.
- Validate before saving large generated test batches.
- Run targeted checks before broad suites when the user is iterating quickly.
- Pair risky edits with checkpoints or reversible change tracking.

## References

- [unit-testing-guide](references/unit-testing-guide.md)
- [model-changes-guide](references/model-changes-guide.md)
- [dependencies-guide](references/dependencies-guide.md)

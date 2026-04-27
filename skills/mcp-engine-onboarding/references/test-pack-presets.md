# Test Pack Presets

Use built-in test packs before generating custom tests. Preview with `dry_run: true` before saving generated tests, and ask for approval before persistent `manage_tests` writes.

## Built-In Packs

Use `manage_tests` with:

```json
{
  "operation": "packs_apply",
  "spec": { "pack_id": "metadata-quality" },
  "dry_run": true
}
```

Available pack IDs:

- `metadata-quality`: documentation, naming, formatting, folders, calculated column usage, and relationship metadata.
- `documentation-baseline`: conservative documentation checks for governance-heavy models.
- `presentation-hygiene`: report-consumption quality, format strings, and display folders.
- `referential-integrity`: orphan key and relationship integrity checks based on model relationships.
- `relationship-governance`: relationship shape and governance checks.
- `time-intelligence`: date table and time-intelligence validation checks.

## Recommendation Matrix

For first-time setup:

- Start with `metadata-quality`.
- Add `presentation-hygiene` for report-facing models.

For handoff or governance:

- Use `documentation-baseline`.
- Add `relationship-governance` and `referential-integrity` when relationships are important.

For active development:

- Use `metadata-quality`.
- Add `referential-integrity` before schema or relationship refactors.
- Add `time-intelligence` when the model has date tables, calendars, or time-intelligence measures.

For AI readiness:

- Use `metadata-quality`, `documentation-baseline`, and `presentation-hygiene`.
- Route deeper artifact drafting and validated-answer backlog work to `mcp-engine-ai-readiness`.

For security workflows:

- Inspect roles and object security before proposing custom `rls_validation` or `ols_validation` tests.
- Require one-to-one expectation coverage for declared principals and targets.
- Keep starter security tests narrow and reviewable.

## Starter Custom Tests

Generate custom tests only after inspecting model metadata or receiving exact business rules from the user. Good candidates:

- Critical measure smoke tests.
- RLS principal row-count or query-success checks.
- OLS table/column access expectations.
- Performance budgets for known expensive queries.

Avoid inventing expected business values without user input or a trusted baseline.

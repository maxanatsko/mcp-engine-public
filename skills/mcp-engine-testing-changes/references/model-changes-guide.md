# Model Changes Guide (`manage_model_changes`)

This guide explains how to use the Pro Model Changes feature to track write operations, diff schema changes, pin checkpoints, and manage/apply changesets.

## Related Tools

- `manage_model_changes`: Pro feature for transaction history, diffs, rollback, checkpoints, and changesets
- `list_model`: Inspect current model state (`operation: "list"`, `spec: { type: "tables|measures|relationships" }`)
- `list_model`: Find dependencies before refactors (`operation: "search"`)
- `run_query`: Validate results after changes (`operation: "execute"`)

## What Gets Tracked

When Pro Model Changes is enabled, write tools are executed inside a transaction recorder that captures:

- Tool name and arguments
- Before/after model schema snapshots (for diffing)
- Optional "before" snapshots for rollback support

This allows you to list recent write operations and inspect how they changed the model.

### Model Write Coverage Matrix

Tracked model-mutating operations:

| Tool | Operation scope | Transaction behavior |
| --- | --- | --- |
| `manage_schema` | tables, field parameters, partitions, source columns, calculated columns, relationships, hierarchies, calendars | Captures before/after schema and diffs object plus metadata changes |
| `manage_semantic` | measures, KPIs, calculation groups/items, UDFs, named expressions, Power Query parameters | Captures before/after schema and diffs semantic object plus metadata changes |
| `manage_security` | roles, RLS filters, OLS permissions, perspectives/members | Captures before/after schema and diffs security changes |
| `manage_localization` | cultures, culture annotations, translations | Captures before/after schema and diffs localization changes |
| `manage_model_properties` | description, culture, discourage implicit measures, compatibility level, model annotations | Only `operation="update"` is transaction-wrapped |
| `manage_refresh` | partition, table, model processing | `status_only=true` is not recorded; actual refresh writes create a processing transaction even when schema is unchanged |
| `manage_model_changes` | `apply_changeset`, `rollback_transaction`, `restore_checkpoint`, `undo`, `redo` | History operations are traceable and record the restored snapshot or source transaction |

Out of scope for model-change transactions:

- local/admin writes such as preferences, policies, test definitions, audit maintenance, license state, and connection state

## Operations Requiring Confirmation

These operations require confirmation. If the client does not support elicitation, or if `MCP_ENGINE_DISABLE_ELICITATION=true` disables prompts for the process, pass `confirm=true`:

- `rollback_transaction`
- `restore_checkpoint`
- `delete_checkpoint`
- `undo`
- `redo`

## Storage and Configuration

Model changes are stored locally (per user) under:

- Default: `~/.mcp-engine/changes`
- Override: `MCP_ENGINE_CHANGES_DIR`

Additional environment knobs:

- `MCP_ENGINE_CHANGES_MAX_DISK_MB` (default: 200)
- `MCP_ENGINE_CHANGES_RETENTION_DAYS` (default: 30)
- `MCP_ENGINE_CHANGES_CLEANUP_INTERVAL_HOURS` (default: 1)
- `MCP_ENGINE_CHANGES_FINALIZATION_TIMEOUT_MS` (default: 30000; bounded recovery after a model mutation starts)

## Mutation outcomes and recovery

Tracked model writes, changeset batches, undo, redo, transaction rollback, and checkpoint restore include an additive `operation_outcome` object. Existing operation-specific response fields remain unchanged.

- `model_mutation` is `not_applied`, `applied`, or `unknown`.
- `audit_outcome` is `complete` or `incomplete`.
- `retry_safe` states whether repeating the original model mutation is safe. Never infer safety from the MCP error flag alone.
- `failed_phase` and `failure_reason` identify the sanitized finalization failure.
- `recovery.action` tells an automated client to retry the original request, retry finalization, inspect model/history state, or take no action.

When `recovery.action` is `retry_finalization`, call:

```json
{ "operation": "retry_finalization", "operation_id": "<operation_id>" }
```

This operation resumes only the recorded idempotent audit/finalization phase. It never re-executes the model write or snapshot restore. Repeating `retry_finalization` after completion is safe and reports the completed outcome. If `model_mutation="unknown"`, inspect the current model and transaction history instead of replaying the original operation.

Caller cancellation stops work before the mutation boundary. Once a mutation or restore starts, rollback and mandatory finalization use the bounded internal recovery timeout so caller cancellation cannot cancel its own safety recovery.

### Snapshot ownership and legacy data

Every snapshot capture and destructive restore requires a trustworthy model owner. Starting with version 3.9.2, capture fails before persistence when the current model identity is unresolved; restores reject snapshots whose `model_id` is blank, `unknown`, or a legacy `*:unknown` sentinel with `SNAPSHOT_MODEL_OWNERSHIP_UNKNOWN`, and reject snapshots owned by another model with `SNAPSHOT_MODEL_MISMATCH`. Transactional writes and batches do not execute when their mandatory before-snapshot cannot establish ownership.

Unowned snapshots cannot be claimed from the current connection and have no temporary compatibility window. The only supported migration is the existing stable-identity alias migration: when persisted model identity metadata positively identifies a legacy model ID as an alias of the current stable ID, the stored snapshot owner is rewritten to that real stable ID before restore. Otherwise, keep the legacy snapshot only as offline evidence and create a new checkpoint from the correctly connected model.

## Transaction History

### List transactions

```json
{ "operation": "list_transactions", "limit": 10, "offset": 0 }
```

Response includes a `pagination` object with `total`, `has_more`, `next_offset`. Default limit: 50, max: 1000 (configurable). Failed batch transactions also include durable rollback fields: `rollback_outcome`, `rollback_error`, and `rollback_completed_at`.

### Get a transaction record

```json
{ "operation": "get_transaction", "transaction_id": "txn_abc123" }
```

Transaction details include rollback state for failed batches. `rollback_outcome` is one of `notattempted`, `succeeded`, `failed`, or `unavailable`; `rollback_completed_at` is populated when a failed batch rollback path is finalized, and `rollback_error` is populated when a rollback failed or could not run.

### Diff a transaction

Use `verbosity="summary"` for a compact diff summary and `verbosity="full"` for a structured detailed diff:

```json
{ "operation": "diff_transaction", "transaction_id": "txn_abc123", "verbosity": "summary" }
```

`verbosity="full"` returns grouped structured changes for:

- `model_properties`
- `tables`
- `measures`
- `columns`
- `relationships`
- `calculation_groups`
- `hierarchies`
- `roles`
- `cultures`
- `perspectives`
- `kpis`
- `named_expressions`
- `udfs`

Schema hashes and diff summaries are computed from the same snapshot surface, so writes to security roles, translations, perspectives, hierarchies, KPIs, named expressions, calculation groups, UDFs, relationships, and model properties (including annotations) are reflected in transaction history. Table, partition, column, measure, relationship, and calendar metadata changes are also included in hashes and detailed diffs.

The `full` response includes:

- top-level transaction metadata (`transaction_id`, `diff_summary`, schema hashes)
- `counts` with:
  - `added`, `modified`, and `deleted` for entity changes
  - `model_properties` for model-level property changes
  - `total` for the overall number of returned changes, including model-level property changes
- `changes` grouped by entity type and change type

Unchanged objects and raw snapshot previews are omitted.

`manage_refresh` transactions use an explicit processing summary when the schema hash does not change:

- `No schema changes; processing operation recorded.`

## Rollback

Rollback restores the model to a previously captured "before" snapshot (when available).

```json
{ "operation": "rollback_transaction", "transaction_id": "txn_abc123" }
```

Guidance:

- Prefer rolling back to a pinned checkpoint for intentional restore points.
- Always validate critical measures and visuals after a rollback.
- Rollback requires confirmation. If the client doesn't support elicitation, re-run with `confirm=true`.
  The `confirm` flag is only a compatibility fallback for clients without elicitation; it does not bypass a client-advertised confirmation prompt.

```json
{ "operation": "rollback_transaction", "transaction_id": "txn_abc123", "confirm": true }
```

## Undo / Redo (Single-level)

`manage_model_changes` supports a single-level undo/redo flow:

- `undo` restores a transaction's "before" snapshot and creates a redo point automatically.
- `redo` restores the redo point created by the most recent undo.
- Any subsequent successful model write invalidates the redo point.

### Check redo availability

```json
{ "operation": "status" }
```

Response includes `redo_available` and (when true) a `redo` object with `snapshot_id`, `undone_transaction_id`, and `created_at`.

### Undo

Undo the latest rollbackable transaction:

```json
{ "operation": "undo" }
```

If the client doesn't support elicitation:

```json
{ "operation": "undo", "confirm": true }
```

Undo a specific transaction:

```json
{ "operation": "undo", "transaction_id": "txn_abc123" }
```

If the client doesn't support elicitation:

```json
{ "operation": "undo", "transaction_id": "txn_abc123", "confirm": true }
```

Notes:

- Undo requires that the transaction has a captured `before_snapshot_id` (auto-snapshot).
- Undo creates a redo snapshot of the current model state before restoring the "before" snapshot.
- Undo/redo operations require confirmation. If the client doesn't support elicitation or prompts are disabled by `MCP_ENGINE_DISABLE_ELICITATION=true`, re-run with `confirm=true`.
- If a client advertises elicitation but the confirmation prompt cannot be completed, the operation fails closed rather than trusting model-supplied arguments.

### Redo

```json
{ "operation": "redo" }
```

If the client doesn't support elicitation:

```json
{ "operation": "redo", "confirm": true }
```

## Checkpoints (Pinned Snapshots)

Model-change history from SemanticOps MCP v3.8.2 remains within the supported identity-migration window. The server migrates legacy model identity aliases through one shared resolver; this migration is removed when the minimum supported release advances beyond data that requires it.

Local model-change storage supports database schemas v5 through v10. Schemas v5-v9 upgrade to v10; older databases must first be opened with SemanticOps MCP v3.8.2, and newer schemas are rejected to prevent downgrade corruption. Incomplete restore finalizations durably protect their pre-restore recovery snapshot from retention and explicit deletion until finalization completes. Snapshot content is stored by checkpoint ID, not by a database-persisted path, and restore accepts only the `{ "create": { "database": ... } }` envelope produced by the server.

### Pin a checkpoint

```json
{ "operation": "pin_checkpoint", "name": "Before refactor" }
```

### List checkpoints

```json
{ "operation": "list_checkpoints", "limit": 50, "offset": 0 }
```

Response includes a `pagination` object with `total`, `has_more`, `next_offset`.

### Restore a checkpoint

```json
{ "operation": "restore_checkpoint", "checkpoint_id": "snap_xyz789" }
```

If the client doesn't support elicitation:

```json
{ "operation": "restore_checkpoint", "checkpoint_id": "snap_xyz789", "confirm": true }
```

### Delete a checkpoint

Deleting a checkpoint requires confirmation. If the client doesn't support elicitation or prompts are disabled by `MCP_ENGINE_DISABLE_ELICITATION=true`, re-run with `confirm=true`.
If a client advertises elicitation, the prompt must complete successfully; `confirm=true` is not a security boundary and does not override a failed prompt.

```json
{ "operation": "delete_checkpoint", "checkpoint_id": "snap_xyz789" }
```

If the client doesn't support elicitation:

```json
{ "operation": "delete_checkpoint", "checkpoint_id": "snap_xyz789", "confirm": true }
```

## Changesets (Queue and Apply Multiple Operations)

Changesets let you stage multiple tool calls and apply them as a batch.

Changesets are scoped to the model that created them. Direct `changeset_id` operations such as add, preview, apply, and delete are rejected when the active connection is a different model. Connection-management operations, read-only refresh probes, and model changes/history/checkpoint/changeset control tools such as `manage_model_changes` cannot be queued or replayed inside changesets.

For multiple related Power BI Desktop M mutations, always use one changeset instead of calling the write tools directly one operation at a time. Supported M-table, M-partition, named-expression, Power Query parameter, query-group, and applicable formatting operations are planned in order against one staged TOM model. `apply_changeset` validates the final graph and calls `Model.SaveChanges()` exactly once when the staged graph contains a mutation; an all-no-op changeset performs no commit. Unsupported or unsafe combinations fail before mutation; the server does not silently fall back to sequential per-operation commits.

This single-commit contract is specific to eligible Desktop M changesets. Service/XMLA connections and non-M changesets retain their normal execution behavior.

Lifecycle:
- `draft`: can be updated, previewed, applied, or deleted
- `applied`: immutable history entry; cannot be deleted

### Create a changeset

```json
{ "operation": "create_changeset", "name": "Batch updates" }
```

### Add an operation to a changeset

Provide:

- `tool_name`: the tool to run (e.g., `manage_semantic`)
- `tool_args`: the arguments object for that tool

```json
{
  "operation": "add_to_changeset",
  "changeset_id": "cs_123",
  "tool_name": "manage_semantic",
  "tool_args": {
    "operation": "create_measure",
    "table": "Sales",
    "target": "Total Sales",
    "spec": {
      "expression": "SUM('Sales'[Amount])",
      "format_string": "$#,0"
    }
  }
}
```

### Preview a changeset

```json
{ "operation": "preview_changeset", "changeset_id": "cs_123" }
```

### Apply a changeset

```json
{ "operation": "apply_changeset", "changeset_id": "cs_123" }
```

A mutating eligible Desktop M changeset that completed its single TOM commit reports:

```json
{
  "commit_strategy": "single_tom_commit",
  "power_query_adoption_status": "user_verification_required",
  "post_commit_validation_required": true,
  "next_steps": [
    "Open Power Query and confirm the final query tree and M text match the intended changeset.",
    "If Power BI Desktop shows pending query changes, use Apply or Close & Apply, then validate the affected model objects and representative queries.",
    "Do not start another Desktop M changeset until Power Query adoption and post-commit validation are complete."
  ]
}
```

The presence of `commit_strategy: "single_tom_commit"` confirms that one TOM commit and mandatory server-side finalization completed. `status: "applied"` alone does not prove a commit occurred because an all-no-op changeset also completes as applied without calling `SaveChanges`. TOM readback cannot prove that the Power Query editor adopted the committed state, so `power_query_adoption_status` remains `user_verification_required`; it does not mean Power Query rejected the changes.

Before another Desktop M changeset:

1. Open Power Query and confirm the final query tree, group placement, parameter metadata, and M text match the intended changeset.
2. If Power BI Desktop shows pending query changes, use **Apply** or **Close & Apply** as appropriate for the current editor workflow. Do not press Apply blindly when Desktop shows no pending query changes.
3. Re-read the affected model objects and run representative validation queries or test packs.
4. Continue only after the Power Query and model results agree. If they diverge, stop and preserve the session for diagnosis instead of replaying the changeset.

### Delete / list changesets

```json
{ "operation": "delete_changeset", "changeset_id": "cs_123" }
```

`delete_changeset` only works for draft changesets. Once a changeset has been applied, it remains in history and cannot be deleted.

```json
{ "operation": "list_changesets", "limit": 50, "offset": 0 }
```

Response includes a `pagination` object with `total`, `has_more`, `next_offset`.

## Recommended Workflow for Risky Refactors

1. Pin a checkpoint: `pin_checkpoint`.
2. Create a changeset and add your planned operations.
3. Preview the changeset.
4. Apply changeset.
5. For a Desktop M changeset, confirm Power Query adoption; use Apply or Close & Apply only if Desktop shows pending query changes.
6. Validate: `list_model` with `operation: "search"` for broken references + `run_query` for key validation queries.
7. If needed, restore the checkpoint.

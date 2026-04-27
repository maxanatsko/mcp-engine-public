# Model Changes Guide (`manage_model_changes`) (Pro)

This guide explains how to use the Pro Model Changes feature to track write operations, diff schema changes, pin checkpoints, and manage/apply changesets.

## Related Tools

- `manage_model_changes` (Pro): Transaction history, diffs, rollback, checkpoints, changesets
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
| `manage_schema` | tables, partitions, source columns, calculated columns, relationships, hierarchies, calendars | Captures before/after schema and diffs object plus metadata changes |
| `manage_semantic` | measures, KPIs, calculation groups/items, UDFs, named expressions, Power Query parameters | Captures before/after schema and diffs semantic object plus metadata changes |
| `manage_security` | roles, RLS filters, OLS permissions, perspectives/members | Captures before/after schema and diffs security changes |
| `manage_localization` | cultures, culture annotations, translations | Captures before/after schema and diffs localization changes |
| `manage_model_properties` | description, culture, discourage implicit measures, compatibility level, model annotations | Only `operation="update"` is transaction-wrapped |
| `refresh` | partition, table, model processing | `status_only=true` is not recorded; actual refresh writes create a processing transaction even when schema is unchanged |
| `manage_model_changes` | `apply_changeset`, `rollback_transaction`, `restore_checkpoint`, `undo`, `redo` | History operations are traceable and record the restored snapshot or source transaction |

Out of scope for model-change transactions:

- local/admin writes such as preferences, policies, test definitions, audit maintenance, license state, and connection state

## Storage and Configuration

Model changes are stored locally (per user) under:

- Default: `~/.mcp-engine/changes`
- Override: `MCP_ENGINE_CHANGES_DIR`

Additional environment knobs:

- `MCP_ENGINE_CHANGES_MAX_TRANSACTIONS` (default: 50)
- `MCP_ENGINE_CHANGES_MAX_DISK_MB` (default: 200)
- `MCP_ENGINE_CHANGES_RETENTION_DAYS` (default: 30)
- `MCP_ENGINE_CHANGES_CLEANUP_INTERVAL_HOURS` (default: 1)

## Transaction History

### List transactions

```json
{ "operation": "list_transactions", "limit": 10, "offset": 0 }
```

Response includes a `pagination` object with `total`, `has_more`, `next_offset`. Default limit: 50, max: 1000 (configurable).

### Get a transaction record

```json
{ "operation": "get_transaction", "transaction_id": "txn_abc123" }
```

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

`refresh` transactions use an explicit processing summary when the schema hash does not change:

- `No schema changes; processing operation recorded.`

## Rollback

Rollback restores the model to a previously captured "before" snapshot (when available).

```json
{ "operation": "rollback_transaction", "transaction_id": "txn_abc123" }
```

Guidance:

- Prefer rolling back to a pinned checkpoint for intentional restore points.
- Always validate critical measures and visuals after a rollback.
- Rollback requires confirmation. If the client doesn't support elicitation, re-run with `confirm=true`:

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
- Undo/redo operations require confirmation. If the client doesn't support elicitation, re-run with `confirm=true`.

### Redo

```json
{ "operation": "redo" }
```

If the client doesn't support elicitation:

```json
{ "operation": "redo", "confirm": true }
```

## Checkpoints (Pinned Snapshots)

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

```json
{ "operation": "delete_checkpoint", "checkpoint_id": "snap_xyz789" }
```

If the client doesn't support elicitation:

```json
{ "operation": "delete_checkpoint", "checkpoint_id": "snap_xyz789", "confirm": true }
```

Backward compatibility: older clients used `transaction_id` as the identifier for checkpoints. This is still accepted.

## Changesets (Queue and Apply Multiple Operations)

Changesets let you stage multiple tool calls and apply them as a batch.

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
5. Validate: `list_model` with `operation: "search"` for broken references + `run_query` for key validation queries.
6. If needed, restore the checkpoint.

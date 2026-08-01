# Audit Logging & `manage_audit` (Enterprise)

`manage_audit` provides a tamper-evident audit trail for MCP tool usage using a SHA-256 hash chain stored in a local SQLite database.

Identity fields:
- `operator_id`, `operator_name`, `operator_source`, and `audit_session_id` are the primary accountability fields.
- `user_id` is historical-only. New intent, read, and completion rows write `NULL`; existing rows remain readable and exportable.
- `hash_version` indicates which hash input format was used for each entry; legacy rows remain verifiable.

Related:
- `tool-invocation-conventions.md (from the mcp-engine-bootstrap skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`
- `pii-masking-guide.md` (redaction basics)
- `troubleshooting-guide.md (from the mcp-engine-bootstrap skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`

## Feature gating

- Requires **Enterprise tier** to execute audit operations and to record tool activity.
- The tool itself is hidden from `tools/list` unless `MCP_ENGINE_AUDIT_TOOL_ENABLED=true`.

## Storage and configuration

### Database

- Default DB path: `~/.mcp-engine/audit.db`
- Override: `MCP_ENGINE_AUDIT_DB=/path/to/audit.db` (supports `~` expansion)

### Environment variables

- `MCP_ENGINE_AUDIT_ENABLED=true|false` (default: enabled unless explicitly `false`)
- `MCP_ENGINE_AUDIT_RETENTION_DAYS=90` (default: 90)
- `MCP_ENGINE_AUDIT_INCLUDE_READS=true|false` (default: false; write-only)
- `MCP_ENGINE_AUDIT_CLEANUP_HOURS=24` (default: 24)
- `MCP_ENGINE_AUDIT_WRITE_TIMEOUT_MS=30000` (default: 30000; timeout for each direct durable intent or completion write)
- `MCP_ENGINE_AUDIT_TOOL_ENABLED=true|false` (default: false; tool hidden unless `true`)

Write-path behavior:
- Eligible write operations are `fail_closed`: MCP directly persists a `write_intent` entry, then either executes or blocks the handler, then directly persists exactly one deterministic `write` outcome for that intent.
- If pre-write intent persistence cannot be confirmed before execution, the tool returns `AUDIT_LOG_WRITE_FAILED` and the write handler is not called.
- If final write audit persistence fails after handler execution, the tool returns `AUDIT_LOG_WRITE_FAILED`; the underlying write may already be durable, but the pre-write intent remains available for correlation by `request_id`.
- Nested direct writes inherit the outer write lifecycle and do not create another intent/outcome pair.
- For Pro model-change operations, the failure also carries `operation_outcome` with the exact mutation state and a `retry_finalization` recovery action. Retrying finalization completes the deterministic audit entry correlated with the durable write intent and cannot duplicate the model mutation or audit-chain entry.
- `retry_finalization` reserves and completes its own lifecycle while the returned `operation_outcome` continues to describe the recovered operation.
- `manage_audit` status, list, verify, and inline export suppress self-auditing. File export remains an audited write.
- Read operations remain best-effort when `MCP_ENGINE_AUDIT_INCLUDE_READS=true`. They use an internal fixed-capacity buffer of 1000 entries and may be dropped under load; read pressure never delays durable write persistence.

### Startup export (one-time)

- `MCP_ENGINE_AUDIT_EXPORT_ON_START=/path/to/export.json` will write an export once on server startup.
- `MCP_ENGINE_AUDIT_EXPORT_MAX_ENTRIES=100000` (set `0` for unlimited).
- The maximum is applied at the database read boundary. Unlimited startup exports still read and serialize in bounded pages rather than materializing the complete log.

### Compatibility and migration (3.9.56)

- Inline exports now stop before enumeration and return `AUDIT_EXPORT_INLINE_LIMIT_EXCEEDED` when the matching count exceeds the effective `manage_audit` pagination maximum (10,000 by default). Use `save_to_path` to stream larger exports.
- Export format v2, entry fields, SQLite storage, retention anchors, and v1/v2 hash bytes are unchanged. No database or export-file migration is required.
- The internal audit store query API was replaced in place. There is no legacy store facade, compatibility interface, queue setting, or old/new dual API for operators to configure.

## Operations

Requests are validated strictly. Unknown fields, blank filters or paths, malformed dates, unsupported operation types, reversed date ranges, and fields that do not apply to the selected operation are rejected instead of being ignored.

Operation-specific fields:
- `status` and `verify`: `operation` only.
- `list`: `model_id`, `start_date`, `end_date`, `operation_type`, `limit`, `offset`, and `include_arguments`.
- `export`: `model_id`, `start_date`, `end_date`, `operation_type`, `include_arguments`, and `save_to_path`.

Date filters must be ISO-8601. Date-only values and timestamps without an offset are interpreted as UTC; timestamps with an explicit offset are normalized to UTC before querying and reporting.

### `status`

Returns audit status and DB metadata.

The aggregate response includes `total_entries` and `anchor_count` without loading entry or anchor collections. When entries exist, `oldest_entry` and `newest_entry` provide the UTC timestamp bounds; empty logs omit both nullable fields.

Example:
```json
{ "operation": "status" }
```

### `list`

Lists audit log entries with optional filters and pagination.

Supported filters:
- `model_id`
- `start_date` / `end_date` (ISO-8601)
- `operation_type`: `"read"`, `"write"`, or `"write_intent"`
- `include_arguments` (default false)

Pagination:
- `limit`: default 100, max 10000 (configurable via `MCP_ENGINE_PAGINATION_*` env vars or `appsettings.json`)
- `offset`: default 0

Response includes `count` (total entries) and a `pagination` object:
```json
{
  "entries": [...],
  "count": 500,
  "pagination": { "limit": 50, "offset": 0, "returned": 50, "total": 500, "has_more": true, "next_offset": 50 }
}
```

Example:
```json
{ "operation": "list", "operation_type": "write", "limit": 50, "offset": 0 }
```

### `verify`

Verifies hash-chain integrity for the full audit log (in insertion order).

Notes:
- Verification checks both:
  1) `previous_hash` linkage
  2) recomputed `hash` matches stored `hash`
- Verification reads the log in bounded insertion-order pages and returns a stable invalid result for malformed hashes, unsupported hash versions, invalid stored operation values, and broken chain links.
- `verify` returns `success=true` even when the chain is invalid; check `chain_valid` in the result.

Example:
```json
{ "operation": "verify" }
```

### `export`

Exports entries (and optionally verifies the chain for **unfiltered** exports).

Important:
- Chain verification only runs for **unfiltered** exports. Any filter (model/date/type) produces a subset that does not necessarily form a contiguous chain.

Inline export:
```json
{ "operation": "export" }
```

- Inline exports are limited to the effective `manage_audit` pagination maximum (10,000 entries by default) because MCP structured content must be materialized as one response.
- When the matching entry count exceeds that limit, the operation returns `AUDIT_EXPORT_INLINE_LIMIT_EXCEEDED`; set `save_to_path` to stream the full export to a file.

File export:
- Provide `save_to_path` to write JSON to disk.
- Absolute paths are allowed.
- Relative paths are saved under `~/.mcp-engine/exports/`.
- Entries are read and serialized in bounded insertion-order pages; full file exports do not materialize the complete log in memory.
- The destination is replaced atomically from a same-directory, owner-only temporary file. Reparse-point paths are rejected, and temporary files are cleaned up after failed or cancelled writes.
- File generation reserves a durable `write_intent` before writing and records its final outcome after completion. The generated file may contain that intent, while the final outcome appears in subsequent lists and exports.

Example:
```json
{ "operation": "export", "save_to_path": "./audit-export.json" }
```

Retention context:
- Unfiltered exports include `anchor_hash` when retention established the trusted predecessor for the first exported row.
- Filtered exports set `anchor_hash` to `null` because omitted rows mean the filtered subset is not a contiguous verifiable chain.

## Retention and administrative boundary

- Audit entries and anchors cannot be cleared or reset through MCP.
- Configured retention cleanup removes only an old prefix of the chain and records the terminal hash in an anchor so later entries remain verifiable.
- Deleting or replacing the SQLite database is an offline operating-system administration action outside the SemanticOps MCP contract. Stop the server and follow your organization's evidence-retention and access-control procedures before changing the database file.

## Privacy and redaction

- Audit entries store a serialized argument payload.
- Known sensitive argument keys are redacted before hashing and persistence (e.g., `license_key`, `password`, `token`).
- Operator identity is resolved from the active execution context:
  - `service_auth` for authenticated Power BI Service sessions
  - `os_user` for Desktop/local sessions
  - `unknown` when no stronger identity can be resolved
- Consider enabling masking policies for outputs depending on your environment (see `pii-masking-guide.md`).

# Audit Logging & `manage_audit` (Enterprise)

`manage_audit` provides a tamper-evident audit trail for MCP tool usage using a SHA-256 hash chain stored in a local SQLite database.

Identity fields:
- `operator_id`, `operator_name`, `operator_source`, and `audit_session_id` are the primary accountability fields.
- `user_id` is retained only as a legacy license identity field for backward compatibility.
- `hash_version` indicates which hash input format was used for each entry; legacy rows remain verifiable.

Related:
- `../../mcp-engine-bootstrap/references/tool-invocation-conventions.md`
- `pii-masking-guide.md` (redaction basics)
- `../../mcp-engine-bootstrap/references/troubleshooting-guide.md`

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
- `MCP_ENGINE_AUDIT_QUEUE_SIZE=1000` (default: 1000; write queue capacity, writes wait when full)
- `MCP_ENGINE_AUDIT_WRITE_TIMEOUT_MS=30000` (default: 30000; write audit persistence timeout)
- `MCP_ENGINE_AUDIT_TOOL_ENABLED=true|false` (default: false; tool hidden unless `true`)

Write-path behavior:
- Write operations are `fail_closed`: the tool result is not considered complete until the audit entry is durably stored.
- If write audit persistence cannot be confirmed before the timeout, the tool returns `AUDIT_LOG_WRITE_FAILED`.
- Read operations remain best-effort when `MCP_ENGINE_AUDIT_INCLUDE_READS=true`; read audit entries may still be dropped under load.

### Startup export (one-time)

- `MCP_ENGINE_AUDIT_EXPORT_ON_START=/path/to/export.json` will write an export once on server startup.
- `MCP_ENGINE_AUDIT_EXPORT_MAX_ENTRIES=100000` (set `0` for unlimited).

## Operations

### `status`

Returns audit status and DB metadata.

Example:
```json
{ "operation": "status" }
```

### `list`

Lists audit log entries with optional filters and pagination.

Supported filters:
- `model_id`
- `start_date` / `end_date` (ISO-8601)
- `operation_type`: `"read"` or `"write"`
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

File export:
- Provide `save_to_path` to write JSON to disk.
- Absolute paths are allowed.
- Relative paths are saved under `~/.mcp-engine/exports/`.

Example:
```json
{ "operation": "export", "save_to_path": "./audit-export.json" }
```

### `clear`

Deletes all audit entries and anchors (destructive).

Example:
```json
{ "operation": "clear" }
```

## Privacy and redaction

- Audit entries store a serialized argument payload.
- Known sensitive argument keys are redacted before hashing and persistence (e.g., `license_key`, `password`, `token`).
- Operator identity is resolved from the active execution context:
  - `service_auth` for authenticated Power BI Service sessions
  - `os_user` for Desktop/local sessions
  - `unknown` when no stronger identity can be resolved
- Consider enabling masking policies for outputs depending on your environment (see `pii-masking-guide.md`).

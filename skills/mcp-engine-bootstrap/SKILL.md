---
name: mcp-engine-bootstrap
description: Use when starting a SemanticOps MCP session, connecting to a Power BI model, checking the current connection, applying saved preferences, composing bulk or write payloads, or recovering from empty results, stale metadata, or argument validation errors. For task work after the session is healthy, use the matching mcp-engine skill (query, schema-authoring, semantic-authoring, testing-changes, security-governance).
---

# PBI Bootstrap

Session setup and recovery for SemanticOps MCP. Use the bundled references as the working documentation.

## Quick Start

1. Check the connection: `manage_model_connection` `{ "operation": "get_current" }`.
2. If no model is connected — Desktop: `{ "operation": "list" }`, present the choices, then `{ "operation": "select", "model_id": "<model_id>" }` with the user's pick. Service: `{ "operation": "authenticate", "source": "service" }` first when needed, then `{ "operation": "list_workspaces" }` for the workspace choice, then `{ "operation": "list", "source": "service", "workspace": "<workspace>" }` to list its datasets, and finally `select` with the returned `model_id` (workspaces themselves are not selectable).
3. Load saved preferences with `manage_preferences` `{ "action": "list", "resource": "rendered" }` and apply them within their stated trust boundary.
4. Read [tool-invocation-conventions](references/tool-invocation-conventions.md) before composing any non-trivial write or bulk request.
5. If a tool returns empty or stale results, switch to [troubleshooting-guide](references/troubleshooting-guide.md) before retrying.

## Route to the right tool family

- `list_model` (`list`, `search`, `analyze`, `info`, `report`) for discovery, search, previews, and metadata inspection.
- `run_query` (`execute`, `analyze`, `vertipaq`, `test_access`) for DAX execution, performance analysis, storage inspection, and RLS checks.
- Write tools (`manage_schema`, `manage_semantic`, `manage_security`) only after the target object and operation are explicit.
- Once connected and healthy, hand off to the task skill: query authoring → `mcp-engine-query`; slow queries → `mcp-engine-dax-performance`; wrong values → `mcp-engine-dax-debugging`; tables, columns, relationships, partitions → `mcp-engine-schema-authoring`; measures and calc groups → `mcp-engine-semantic-authoring`; multi-object refactors → `mcp-engine-refactoring`; tests, checkpoints, rollback → `mcp-engine-testing-changes`; RLS, policy, masking, audit → `mcp-engine-security-governance`. If a named skill is not installed, continue with the tool's `inputSchema` or ask the user to add it.

## Guard bulk and write requests

- Keep `transaction`, `dry_run`, `include_items`, and `include_details` as top-level request controls.
- Put per-item identifiers and `spec` values inside each item; items do not inherit from the request.
- Confirm destructive intent before deletes, renames, broad refreshes, or model-wide rewrites.

## Recover from common failures

- Wrong or stale connection state: re-check with `{ "operation": "get_current" }`, then `{ "operation": "reload" }` after external model changes.
- Object seems missing: broaden discovery with `list_model` `{ "operation": "search", "spec": { "query": "<object name>", "mode": "name" } }` (`spec.query` is required) before concluding it does not exist.
- Argument validation errors: fix key names and payload shape per [tool-invocation-conventions](references/tool-invocation-conventions.md) before changing business logic.
- Full recovery flows: [troubleshooting-guide](references/troubleshooting-guide.md).

---
name: mcp-engine-bootstrap
description: Prepare and recover Power BI MCP Engine sessions. Use when starting a session, connecting to a model, checking current connection, applying preferences, selecting the right tool family, troubleshooting empty results or stale metadata, or fixing malformed tool arguments and bulk payloads.
---

# PBI Bootstrap

Use the bundled references in this skill as the working documentation for session setup and recovery.

## Quick Start

1. Check the current connection state first.
2. If no model is connected, list available models or service workspaces and wait for user choice before selecting.
3. Apply portable preferences before doing substantial work.
4. Read the payload conventions reference before composing non-trivial write or bulk requests.
5. If a tool returns empty or stale results, switch to the troubleshooting workflow before retrying.

## Workflow

### Connect first

- Start with the connection tool and confirm whether a model is already selected.
- If multiple models are available, present the choices instead of auto-selecting.
- If the user is on Power BI Service, authenticate before listing service resources when needed.
- Re-check the current connection before using model-dependent tools.

### Route to the right tool family

- Use `list_model` for discovery, search, previews, and model metadata inspection.
- Use `run_query` for DAX execution, performance analysis, VertiPaq inspection, and RLS query checks.
- Use write tools only after the target object and operation are explicit.
- Use the testing or changes workflows before broad refactors or risky edits.

### Guard bulk and write requests

- Read [tool-invocation-conventions](references/tool-invocation-conventions.md) before composing any bulk payload.
- Keep `transaction`, `dry_run`, `include_items`, and `include_details` as top-level request controls.
- Put per-item identifiers and `spec` values inside each item rather than assuming inheritance.
- Confirm destructive intent before submitting deletes, renames, broad refreshes, or model-wide rewrites.

### Recover from common failures

- Read [troubleshooting-guide](references/troubleshooting-guide.md) when connection state looks wrong, metadata is stale, or argument validation fails.
- Reload metadata before retrying after external model changes.
- Broaden discovery before assuming an object is missing.
- Fix key names and argument shape issues before changing business logic.

## References

- [tool-invocation-conventions](references/tool-invocation-conventions.md) — payload shapes, bulk rules, pagination, and naming
- [troubleshooting-guide](references/troubleshooting-guide.md) — connection recovery, stale metadata, and argument mistakes

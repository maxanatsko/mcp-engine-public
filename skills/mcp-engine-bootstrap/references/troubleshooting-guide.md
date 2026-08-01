# Troubleshooting Guide

This guide lists common issues when using SemanticOps MCP, formerly MCP Engine, and practical steps to resolve them for Power BI Desktop workflows.

## Related Tools and Resources

- `manage_model_connection`: `operation="list"|"list_workspaces"|"select"|"get_current"|"reload"|"accept_current_model_state"|"authenticate"|"cancel_authentication"|"sign_out"|"set_impersonation"` for model selection, service sign-in, authentication cancellation, metadata refresh, and Full-mode manual Desktop verification resolution
- `list_model`: `operation="list"|"search"|"analyze"|"info"` for discovery and diagnosis
- `run_query`: `operation="execute"|"analyze"|"vertipaq"|"test_access"` for validation and performance checks
- `tool-invocation-conventions.md`: argument shape and bulk patterns

## Connection and Discovery

### Power BI Service sign-in flow fails

Use `manage_model_connection` with `operation="authenticate"`, `source="service"`, and one of:

- `auth_flow="interactive"` to open a browser on the machine running SemanticOps.
- `auth_flow="device_code"` for remote or headless hosts.
- `auth_flow="silent"` to restore only a reusable in-memory, cached, or configured noninteractive session. It returns `status: "unauthenticated"` when none exists and never starts user interaction.
- `auth_flow="auto"`, or omit `auth_flow`, to preserve `MCP_ENGINE_AUTH_MODE` behavior.

Silent MSAL acquisition may contact Microsoft Entra to refresh a cached token, but it never opens a browser, issues a device code, or falls through to an interactive flow. Interactive flow returns `interactive_auth_unavailable` when the host cannot open a system browser and does not fall back to device code. Explicit interactive or device-code public-client flows return `auth_flow_not_supported_for_configured_mode` when the deployment uses an access token or service principal. These flow choices do not bypass Microsoft Entra Conditional Access.

If an authentication attempt is already pending, repeat the same flow to retrieve its current state or call `operation="cancel_authentication"`, `source="service"` before switching flows. Cancellation succeeds with `cancelled=false` when no attempt is pending.

### “Not connected” / no model selected

Symptoms:

- Tools that require a live model connection return an error about connection/model selection.

Fix:

1. Run `manage_model_connection`:
   - `operation="list"` to discover available models
   - `operation="select"` to select the intended model
   - `operation="get_current"` to confirm
2. Then retry the tool.

### Tools return empty results unexpectedly

Common causes:

- Wrong model selected (multiple Power BI Desktop instances).
- Filtering too aggressively (`list_model` with incorrect `table` or `name_filter`).
- `list_model` `operation="list"` filters internal/system artifacts by default. For raw diagnostics, set `spec.include_system_artifacts: true`.

Fix:

- Re-check selected model with `manage_model_connection operation="get_current"`.
- Use broader discovery:
  - `list_model` with `operation: "list"`, `spec: { type: "tables" }`
  - `list_model` with `operation: "search"`, `spec: { query: "...", mode: "name" }` (wildcards supported)
  - `list_model` with `operation: "search"`, `spec: { query: "...", mode: "text" }` (substring over metadata; optionally `include_fields: ["folders","formatting","members","security"]`)

### "Model changed externally" / "Operation blocked"

Symptoms:

- Tools fail with an error mentioning "model changed externally" or "Operation blocked".
- A prompt may appear asking to reload metadata.

Cause:

- The model was modified outside of MCP (e.g., in Power BI Desktop UI or Tabular Editor) since MCP last read the metadata.
- MCP detects this via a best-effort mechanism:
  - Power BI Desktop: uses an Analysis Services server trace when available (more reliable for metadata edits).
  - Fallback: compares the model's last-modified timestamp with a stored baseline.

Fix:

1. Run `manage_model_connection` with `operation="reload"` to refresh TOM metadata.
2. Retry your original tool call.

Alternatively, if prompted to reload via elicitation:

- Accept the reload prompt to refresh metadata automatically.
- Decline if you want to continue with potentially stale metadata (only works if `ExternalChangeFailClosed=false`).

Configuration:

- Set `MCP_ENGINE_EXTERNAL_CHANGE_AUTO_RELOAD=true` to automatically reload without prompting.
- Set `MCP_ENGINE_EXTERNAL_CHANGE_FAIL_CLOSED=false` to allow operations on stale metadata.

### Desktop verification remains pending after reload

`reload` is the normal resolution path. It refreshes Desktop metadata and clears the pending revision only when the saved expectation and model-health checks pass.

If the current Desktop model is correct but automatic verification cannot prove the active revision, pending responses with available evidence include `manual_resolution_action`. Review its warning and use the exact `model_id` and `verification_revision_id` it supplies:

```json
{
  "operation": "accept_current_model_state",
  "model_id": "<exact model ID>",
  "verification_revision_id": "<exact revision ID>",
  "confirm": true
}
```

This Full-mode-only operation refreshes the same Desktop model again, confirms that neither identity nor revision changed, and then clears only that exact revision. When elicitation is available, approve the default-false prompt; otherwise `confirm=true` is required. Success reports `accepted_current_state` and warns that the original mutation was not proven. It does not report the saved expectation as reflected or verified.

Do not copy IDs from an older response. A model switch, concurrent write, repeated acceptance, unavailable evidence, disconnected Desktop session, declined or malformed confirmation, cancellation, refresh failure, or stale revision leaves the guard active. Reconnect or run `reload` first when Desktop is disconnected. Service/XMLA, read-only, and browse-only modes do not expose this operation.

## Confirmation Prompts Are Disabled

Cause:

- `MCP_ENGINE_DISABLE_ELICITATION=true` disables interactive MCP elicitation prompts for the server process.

Impact:

- Operations with explicit non-interactive fallbacks may ask you to re-run with `confirm=true`.
- Prompt-only destructive/admin operations remain blocked.
- Policy `require_confirm` rules behave like an unsupported client. Use `MCP_ENGINE_POLICY_CONFIRMATION_UNSUPPORTED=deny` when disabled prompts should fail closed.

Fix:

- Unset `MCP_ENGINE_DISABLE_ELICITATION`, set it to `false`, or use the documented `confirm=true` fallback for operations that support it.

## Policy Configuration Fails During Startup

Cause:

- A policy enum uses an undocumented alias, a numeric limit is outside its documented inclusive range, the policies directory is not writable, or a configured path cannot be normalized.

Behavior:

- SemanticOps rejects explicit invalid policy configuration before policy runtime services are constructed, regardless of mode, connection source, or license tier.
- The error names the exact configuration property and environment variable. Invalid bundle-path errors include only a safe filename and reason, never the full directory.
- A syntactically valid bundle path may point to a missing file; the existing bundle load and lockdown path handles that after startup.

Fix:

1. Use only `fail_closed|fail_open` for `MCP_ENGINE_POLICY_FAIL_MODE` and `proceed|deny` for `MCP_ENGINE_POLICY_CONFIRMATION_UNSUPPORTED`.
2. Check the accepted numeric ranges in the policy guide or admin operations reference. Defaults apply only when a variable is absent.
3. Use `~`, `~/...`, or `~\...` only at the beginning of a policy path. Embedded tildes are preserved literally.
4. Correct the reported value and restart; policy configuration is immutable for the process lifetime and is not re-read after startup.

## `POLICY_PROCESSING_FAILED` / Local Policy Store Failed

Cause:

- A local policy file could not be read or secured, contains empty/corrupt/`null` JSON, has `rules: null`, or contains an invalid store or rule.
- A condition or required context-enrichment step failed while evaluating a rule.

Behavior:

- The default `MCP_ENGINE_POLICY_FAIL_MODE=fail_closed` blocks tools with `POLICY_PROCESSING_FAILED`; `manage_policy` and `manage_policy_ui` stay available for recovery.
- Invalid stores are rejected atomically. SemanticOps does not publish a filtered subset of valid rules.
- Explicit `fail_open` is intended for development. It records the structured failure and continues through another valid scope or later unrelated rules.
- Enterprise bundle failure retains its separate lockdown status and recovery allowlist.

Fix:

1. Call `manage_policy` with `operation="status"` and inspect `local_store_state` and sanitized `local_store_failures`.
2. Repair or reset only the failed scope with `manage_policy`; do not rely on the rejected file's apparently valid rules.
3. Restart after changing `MCP_ENGINE_POLICY_FAIL_MODE` or `MCP_ENGINE_POLICY_CONFIRMATION_UNSUPPORTED`. Invalid explicit values fail startup.
4. Keep `fail_closed` for production. If temporarily using `fail_open`, verify the audit entry `_policy` annotation reports `proceeded` and return to `fail_closed` after recovery.

## Argument Shape and Naming Issues

### “Unknown property” / “missing required field”

Cause:

- Tool argument keys are strict, and the server expects `snake_case`.
- Some clients default to `camelCase` field names (or auto-transform keys), which can cause schema mismatches.

Fix:

- Inspect the tool schema via the MCP `tools/list` method and use the exact key names.
- All tools use `snake_case` consistently:
  - `manage_semantic`: spec contains `format_string`, `format_string_expression`, `display_folder`, `is_hidden`
  - `manage_schema`: spec contains `format_string`, `display_folder`, `is_hidden`, `new_name`

See: `tool-invocation-conventions.md`.

### Bundled reference path treated as a web URL

Cause:

- Bundled reference paths in this skill are local markdown files, not HTTP(S) URLs.

Fix:

- Open the corresponding markdown file in this skill's `references/` directory directly.
- Use `resources/list` when you need to discover available documentation resources.
- Do not send bundled reference paths to `web_fetch`.

### Bulk mode “items must be an array”

Cause:

- Bulk-enabled write tools expect `items: [ { ... }, { ... } ]`.

Fix:

```json
{
  "operation": "update_measure",
  "transaction": true,
  "dry_run": true,
  "items": [
    { "table": "Sales", "target": "Total Sales", "spec": { "expression": "..." } }
  ]
}
```

Calc-group note:

- `manage_semantic` calc-group bulk operations use top-level `items[]` like other bulk operations. Calculation group items belong inside each item's `spec.items` for create or `spec.items_upsert` / `spec.items_delete` for update.

Bulk validation notes:

- Top-level object identifiers such as `table` and `target` are single-item fields.
- In bulk mode, each item must include its own identifiers; missing fields are reported per item instead of inheriting top-level values.

## DAX and Expression Errors

### DAX syntax error / “The expression is not valid”

Fix:

- Validate quoting rules (tables in quotes, columns/measures in brackets).
- Use `dax-query-guide.md (from the mcp-engine-query skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)` patterns.
- If the failing expression is in a measure, search for dependencies:
  - `list_model` with `operation: "search"` and `spec: { mode: "dax", query:"MeasureName"` or query for `"[Measure]"`.

### DAX query returns no rows

Common causes:

- Filters eliminate all rows.
- Using `SUMMARIZECOLUMNS` without measures (may return no rows because it removes all-blank rows).

Fix:

- Start from a simpler query:
  - `EVALUATE TOPN(10, 'Table')`
- Add filters gradually.

### `COUNTROWS(...)` or scalar queries return `null`

Cause:

- `run_query` preserves DAX `BLANK` as JSON `null`.
- This can happen for genuinely blank results, not only for missing tables or objects.

Fix:

- Treat `null` as a DAX result first, then verify object existence separately with `list_model`.
- When debugging a newly created calculated table, inspect the create-table warnings and processing signals before assuming the table is missing.

## Relationship / Modeling Issues

### Ambiguous filter path / unexpected totals

Cause:

- Multiple paths between tables, bidirectional relationships, or incorrect cardinalities.

Fix:

- Inspect relationships:
  - `list_model` with `operation: "list"`, `spec: { type: "relationships" }`
- Prefer:
  - star schema
  - `cross_filter_direction="OneDirection"`
- Use inactive relationships for alternate date keys and `USERELATIONSHIP` in measures.

See: `relationships-guide.md (from the mcp-engine-schema-authoring skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)` and `modeling-best-practices-guide.md (from the mcp-engine-semantic-authoring skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`.

### Relationship was created but `RELATED()` or validation still fails

Cause:

- New relationships can require a `Calculate` refresh before validation-sensitive measures see the updated relationship state.

Fix:

1. Run `manage_refresh` with `scope: "Table"` or `scope: "Model"` and `refresh_type: "Calculate"`.
2. Retry the measure validation or DAX query.
3. If the error persists, inspect the relationship definition and key column types.

## Compatibility Level Issues

### “Compatibility level too low” (calc groups / incremental refresh / UDFs)

Cause:

- The feature requires a minimum model compatibility level.

Fix:

1. Check current level:
   - `manage_model_properties operation="get"`
2. Upgrade if appropriate:

```json
{
  "operation": "update",
  "compatibility_level": 1702,
  "allow_compatibility_upgrade": true
}
```

Notes:

- Upgrades are irreversible and may be blocked by the host.
- Keep a PBIX backup before upgrading.

## Incremental Refresh Policy Issues

### RangeStart/RangeEnd not found

Cause:

- Required DateTime parameters are missing.

Fix:

- Create Power Query parameters named `RangeStart` and `RangeEnd` (DateTime), or set `range_start_parameter` / `range_end_parameter` to the names you use.

See: `incremental-refresh-policy-guide.md (from the mcp-engine-schema-authoring skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`.

### “refresh_policy.source_expression is required”

Fix:

- Provide `refresh_policy.source_expression`, or ensure the first partition has an M expression the server can clone.

## Licensing and Pro Gating

### Parameter or feature requires Pro

Symptoms:

- Some tool parameters are gated (e.g., performance plan capture).
- A tool reports that its operation requires Pro tier.

Fix:

- Use non-Pro equivalents where possible, or upgrade your tier for the gated feature.
- For performance: run without `include_query_plan` first, then enable it when needed.

## Large Outputs / Token Pressure

### Responses are too large (query plans, expressions)

Fix:

- Prefer preview patterns:
  - `list_model` with `operation: "list"`, `spec: { type: "measures", include_expression: false, preview_chars: N }` (adjust `type` as needed)
- Only enable:
  - `include_expression=true`
  - `include_query_plan=true`
  - `include_details=true`
  when you actually need the full output.

## Expression and Reference Errors

### "Expression references a column that doesn't exist"

Cause:

- Column was renamed or deleted
- Relationship not established
- Typo in column/table name

Fix:

1. Use `list_model` with `operation: "search"` and `spec: { mode: "dax", query: "'TableName'[ColumnName]" }` to find broken references
2. Check column exists: `list_model` with `operation: "list"`, `spec: { type: "columns", table: "TableName" }`
3. Check relationship setup: `list_model` with `operation: "list"`, `spec: { type: "relationships" }`
4. Update the expression with correct column reference

### "Circular dependency detected"

Cause:

- Measure A references Measure B, which references Measure A
- Calculated column references itself
- Calculation group items with circular references

Fix:

1. Map the dependency chain:
   - `list_model` with `operation: "search"` and `spec: { mode: "dax", query: "[MeasureName]" }`
2. Refactor to break the cycle:
   - Use variables (VAR) to store intermediate results
   - Restructure measure logic to avoid mutual references

### "The expression for column cannot be determined"

Cause:

- Calculated column expression has row context issues
- Missing relationship for RELATED/RELATEDTABLE

Fix:

- For calculated columns, ensure the expression evaluates in row context
- Verify relationships exist: `list_model` with `operation: "list"`, `spec: { type: "relationships" }`
- Consider using measures instead of calculated columns for aggregations

## Direct Lake / Fabric Issues

### Direct Lake fallback to DirectQuery

Symptoms:

- Queries slower than expected
- Fabric capacity metrics show high fallback rate

Common causes:

- Complex iterators (SUMX, FILTER with many rows)
- Unsupported DAX patterns
- Cross-filter scenarios not optimized for Direct Lake

Fix:

- Simplify measure logic
- Pre-aggregate in the lakehouse where possible
- Check Fabric documentation for supported patterns
- Consider hybrid model with aggregation tables

### "Cannot connect to semantic model"

Cause (Fabric):

- XMLA endpoint not enabled
- Capacity paused or unavailable
- Permission issues

Fix:

- Verify XMLA read/write is enabled in capacity settings
- Check capacity is running
- Verify user has appropriate workspace permissions

## Refresh and Processing Errors

### "Refresh failed: Timeout expired"

Cause:

- Large data volume
- Slow data source
- Complex Power Query transformations

Fix:

- Increase timeout settings where available
- Optimize Power Query (reduce steps, use query folding)
- Consider incremental refresh for large tables
- Check data source performance

### "Memory allocation failure during refresh"

Cause:

- Model too large for available memory
- High-cardinality columns consuming memory

Fix:

- Reduce model size (see `vertipaq-optimization-guide.md (from the mcp-engine-dax-performance skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`)
- Remove unused columns
- Consider aggregations or Direct Lake mode
- Upgrade capacity tier if needed

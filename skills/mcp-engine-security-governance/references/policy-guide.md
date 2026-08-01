# Policy Guide

This guide covers policies for tool execution guardrails.

## Related Tools

- `manage_policy`: All policy operations (status, list, get, evaluate, recipes, render, validate, packs_list, packs_apply, put, delete, import, reset)

## Policies

Policies run before tool execution to allow, deny, or require confirmation.

Important:
- Use `deny` for high-risk operations that must be blocked reliably.
- Treat `require_confirm` as a best-effort UX behavior, not a portable compliance control.
- MCP hosts may not support elicitation/confirmation consistently enough for server-side enforcement.
- `MCP_ENGINE_DISABLE_ELICITATION=true` disables interactive confirmation prompts for the process and makes policy confirmation behave like an unsupported client.
- For locked-down deployments that disable elicitation, also set `MCP_ENGINE_POLICY_CONFIRMATION_UNSUPPORTED=deny` if `require_confirm` rules must fail closed instead of proceeding.
- Local policy processing defaults to `fail_closed`. Use `MCP_ENGINE_POLICY_FAIL_MODE=fail_open` only for development scenarios where continuing without a fully evaluated local policy set is acceptable.
- Local/user-authored `deny` rules cannot target `manage_policy` or `manage_policy_ui`. Use `require_confirm` for policy approval workflows.
- Admin lockdown of policy changes comes from bundle/config controls, not ordinary local policy rules.

### List Built-in Recipes

```json
{ "operation": "recipes" }
```

Recipe IDs are render-only templates for `render`. Use `render` to produce JSON for `import`. They are not valid `spec.pack_id` values for `packs_apply`.

### Render a Recipe

```json
{ "operation": "render", "recipe": "block-deletes" }
```

### List Built-in Packs

```json
{ "operation": "packs_list" }
```

Pack IDs are the apply-able bundle namespace for `packs_apply`. They are distinct from the recipe IDs returned by `recipes`.

### Preview a Pack

`packs_apply` defaults to preview mode.

```json
{
  "operation": "packs_apply",
  "scope": "global",
  "spec": { "pack_id": "semantic-quality" }
}
```

### Apply a Pack

```json
{
  "operation": "packs_apply",
  "scope": "global",
  "dry_run": false,
  "spec": { "pack_id": "semantic-quality" }
}
```

Notes:
- Use canonical `spec.pack_id`.
- `spec.pack_id` must come from `packs_list`, not from `recipes`.
- Preview resolves the selected scope and current store, then runs the same full-store planner and validation as apply. It does not use an empty synthetic store.
- A model-scoped preview resolves or provisions the model's stable policy identity before capturing the snapshot. This may create or backfill the model identity annotation after confirmation, but preview never persists policy rules.
- `packs_apply` merges or updates pack-owned rules by ID prefix. It does not reset unrelated rules.
- Re-applying a pack replaces that pack's existing `<packId>.*` rules with the current built-in definition set.
- In read-only or admin-bundle-locked modes, `packs_apply` is limited to `dry_run=true`.

### Create a Policy Rule

```json
{
  "operation": "put",
  "id": "no-delete-measure",
  "action": "deny",
  "tool": "manage_semantic",
  "condition_operation": "delete_measure",
  "message": "Measure deletion is blocked."
}
```

Notes:
- `put` supports only a simple condition via `condition_operation` (matches top-level tool arg `operation`).
- `put` cannot inspect nested payload fields (for example, `spec.description`).
- For advanced conditions (full AST, including `not`, `all_of`, and `arg_missing` on nested fields), use `import`.

### Confirm Renames (Recipe)

```json
{
  "operation": "render",
  "recipe": "confirm-renames"
}
```

### Import a Policy Rule (Advanced Conditions)

```json
{
  "operation": "import",
  "data": "{ \"rules\": [ { \"id\": \"deny-non-whitelisted-ops\", \"action\": \"deny\", \"tool\": \"manage_*\", \"condition\": { \"kind\": \"not\", \"condition\": { \"kind\": \"arg_in\", \"field\": \"operation\", \"values\": [\"create_measure\",\"update_measure\"] } }, \"message\": \"Operation not allowed by policy.\" } ] }"
}
```

Example (field-level validation with dot-notation):

```json
{
  "operation": "import",
  "data": "{ \"rules\": [ { \"id\": \"require-measure-description\", \"action\": \"deny\", \"tool\": \"manage_semantic\", \"condition\": { \"kind\": \"all_of\", \"conditions\": [ { \"kind\": \"arg_equals\", \"field\": \"operation\", \"value\": \"create_measure\" }, { \"kind\": \"arg_missing\", \"field\": \"spec.description\" } ] }, \"message\": \"spec.description is required for create_measure.\" } ] }"
}
```

Example (JSON-in-string inspection through synthetic parsed fields):

```json
{
  "operation": "import",
  "data": "{ \"rules\": [ { \"id\": \"confirm-policy-import-deny-rules\", \"action\": \"require_confirm\", \"tool\": \"manage_policy\", \"condition\": { \"kind\": \"all_of\", \"conditions\": [ { \"kind\": \"arg_equals\", \"field\": \"operation\", \"value\": \"import\" }, { \"kind\": \"arg_equals\", \"field\": \"parsed.data.rules[].action\", \"value\": \"deny\" } ] }, \"message\": \"Imported policy rules include deny actions: {arg:parsed.data.rules[].action}\" } ] }"
}
```

### Validate a Single Rule

```json
{
  "operation": "validate",
  "data": "{ \"id\": \"require-measure-description\", \"action\": \"deny\", \"tool\": \"manage_semantic\", \"condition\": { \"kind\": \"arg_missing\", \"field\": \"spec.description\" } }"
}
```

### Validate an Import Payload

`validate` also accepts the same import-style envelope as `import`.

```json
{
  "operation": "validate",
  "data": "{ \"rules\": [ { \"id\": \"require-measure-description\", \"action\": \"deny\", \"tool\": \"manage_semantic\", \"condition\": { \"kind\": \"arg_missing\", \"field\": \"spec.description\" } } ] }"
}
```

### Store Schema, Ordering, and Migration

Persisted local stores and Enterprise bundles use policy schema version `1.0`. The top-level `version`, `scope`, and `rules` fields must be present explicitly:

- `version` must be exactly `"1.0"`.
- A global store uses `"scope": "global"` and must omit `scopeId` or set it to `null`.
- A model store uses `"scope": "model"` and its `scopeId` must exactly match the persisted model scope ID for that file.
- `rules` must be an array. Every rule must be non-null and have a non-empty ID.
- Rule IDs are unique case-insensitively within a store. For example, `block-delete` and `BLOCK-DELETE` conflict.
- Rule-count and serialized-byte limits apply to the complete canonical store. Validation never drops invalid rules to make a store load.

Rules are evaluated and displayed in one canonical order. Model scope precedes global scope. Within each scope, lower numeric priority values run first; equal-priority rules retain their persisted array order. Administration list and pack preview output include disabled rules in that same stable order, while evaluation skips disabled rules.

`import` continues to accept the portable rules-only envelope `{ "rules": [...] }`. Omitted metadata is filled from the selected target. If an import supplies `version`, `scope`, or `scopeId`, each value must match version `1.0` and the selected target exactly; imported metadata never overrides the target.

There is no automatic policy migration. Before upgrading an older or manually edited store, keep a backup, add the required metadata, set `version` to `1.0`, correct the scope metadata, and remove case-insensitive duplicate IDs. Invalid local stores are rejected atomically and follow the configured fail-open/fail-closed recovery path; invalid bundles retain bundle lockdown behavior.

### Administration Generations and Mutation Plans

`status`, `list`, and `get` include `generation`, and each response is projected from one coherent runtime snapshot. In particular, model identity and model rules come from the same generation; administration does not resolve connection/model state a second time while formatting the response.

Successful mutations and pack previews include:

```json
{
  "plan": {
    "source_generation": 12,
    "result_generation": 13,
    "added_ids": ["new-rule"],
    "updated_ids": [],
    "removed_ids": []
  }
}
```

For previews, `result_generation` is `null` because no policy store is committed. A commit persists the exact validated candidate and timestamp produced by the planner. If the policy generation or selected model changes before persistence, the server rejects the candidate with nested `error.code="POLICY_PLAN_STALE"`, includes `expected_generation`, `actual_generation`, and `retryable=true`, and does not retry or overwrite the intervening state.

Policy administration, runtime denial, confirmation, and processing failures use the shared nested `error.code` / `error.message` envelope. Unexpected administration exceptions are sanitized. Request cancellation propagates instead of being converted to a policy error.

### Policy Control Plane Protection

- Local `deny` rules targeting `manage_policy` or `manage_policy_ui` are rejected by `validate`, `put`, and `import`.
- Wildcard patterns that would match `manage_policy*` (for example `manage_*`) are also rejected for local `deny` rules.
- Local `allow` and `require_confirm` rules for `manage_policy*` remain valid.
- If any rule in a local store is invalid, the entire affected store is rejected atomically. No valid subset is published.

### Local Failure Behavior

`MCP_ENGINE_POLICY_FAIL_MODE` accepts `fail_closed` (default) or `fail_open`. Invalid explicit values fail server startup.

Policy configuration is parsed and validated once during startup. Values are trimmed; documented enum values are case-insensitive; unlisted aliases and explicitly empty values are rejected. Defaults apply only when an environment variable is absent. Numeric bounds are inclusive:

| Environment variable | Default | Minimum | Maximum |
|---|---:|---:|---:|
| `MCP_ENGINE_POLICY_MAX_RULES` | 100 | 1 | 10,000 |
| `MCP_ENGINE_POLICY_MAX_ID_LENGTH` | 100 | 1 | 1,024 |
| `MCP_ENGINE_POLICY_MAX_MESSAGE_LENGTH` | 500 | 1 | 32,768 |
| `MCP_ENGINE_POLICY_MAX_DESC_LENGTH` | 500 | 1 | 32,768 |
| `MCP_ENGINE_POLICY_MAX_TOOL_LENGTH` | 100 | 1 | 1,024 |
| `MCP_ENGINE_POLICY_MAX_TOTAL_BYTES` | 524,288 | 1 | 10,485,760 |
| `MCP_ENGINE_POLICY_REGEX_TIMEOUT_MS` | 100 | 1 | 5,000 |
| `MCP_ENGINE_POLICY_REGEX_MAX_LENGTH` | 200 | 1 | 4,096 |
| `MCP_ENGINE_POLICY_MAX_NESTING_DEPTH` | 5 | 1 | 64 |

`MCP_ENGINE_POLICIES_DIR` defaults to `~/.mcp-engine/policies`. Policy paths expand only a leading `~`, `~/`, or `~\`; embedded tildes are preserved. The directory is normalized and checked for writability during startup. `MCP_ENGINE_POLICY_BUNDLE_PATH` is optional and normalized at startup; invalid syntax fails with a sanitized filename-only error. A normalized path may refer to a missing file because existing bundle loading and lockdown behavior handles missing or unreadable bundles at runtime.

- `fail_closed`: local read, JSON, I/O, storage-hardening, store/rule validation, condition, and required context-enrichment failures produce a `processing_failure` decision. Tool execution returns `POLICY_PROCESSING_FAILED`. Only `manage_policy` and `manage_policy_ui` remain available to inspect and repair local policy state.
- `fail_open`: the affected local store contributes no rules, and condition/context failures are treated as non-matches. Evaluation continues through another valid scope and later unrelated rules. A later real rule can still allow, deny, or require confirmation while retaining structured failure metadata.

Local store rejection is atomic per scope. A failed global store does not filter and publish its valid rules; a failed model store behaves the same way. Under explicit `fail_open`, another valid scope may still be evaluated. Enterprise bundle load failure keeps its separate lockdown behavior and recovery allowlist.

`manage_policy` `operation="status"` reports `local_failure_mode`, `confirmation_unsupported_behavior`, `local_store_state`, and sanitized `local_store_failures`. `operation="evaluate"` reports `decision`, nullable `action`, and structured `processing_failures`. A processing failure has no synthetic rule ID, matched rule, or policy action.

Policy audit entries add a redacted `_policy` annotation only when processing failures occur. It contains failure category/code, configured mode, and `blocked` or `proceeded`; it never contains filesystem paths, raw exception text, or tool arguments.

### Block Refresh Execution (Recipe)

```json
{
  "operation": "render",
  "recipe": "block-refresh"
}
```

### Test a Rule Before Enabling

```json
{ "operation": "evaluate", "tool": "manage_semantic", "operation_value": "delete_measure" }
```

Evaluation simulation resolves the same canonical tool security identity as runtime execution. For example, evaluating an app-internal `_ui` alias applies the policies of its published canonical tool while retaining the requested alias in the response.

## Built-in Packs

| Pack ID | Purpose |
|---------|---------|
| `safe-authoring-baseline` | Confirm renames, confirm M2M relationship creation, protect risky partition updates |
| `semantic-quality` | Require measure description, format, and display folder; block explicit display-folder clears |
| `delete-operation-denials` | Block delete and remove operations regardless of dependency impact |
| `dependent-delete-denials` | Block supported non-table deletes when downstream dependents exist |
| `table-delete-impact-review` | Require confirmation or deny table deletes based on dependency blast radius |
| `unsupported-operation-denials` | Block unsupported destructive operations that do not yet have dependency modeling |
| `masking-settings-approval` | Require confirmation before changing masking-related runtime settings through `manage_preferences` |
| `security-hardening` | Block role/perspective deletion and confirm security rule changes |
| `refresh-and-partition-safety` | Block refresh execution and partition refresh operations |

`masking-settings-approval` covers:
- `manage_preferences` `action="set"` for protected masking setting IDs
- `manage_preferences` `action="delete"` for protected masking setting IDs
- `manage_preferences` `action="reset"` for `resource="setting"` and `resource="all"`
- `manage_preferences` `action="import"` when the import payload includes masking setting IDs

`delete-operation-denials` covers:
- Broad delete denial rules for `manage_semantic`, `manage_schema`, `manage_security`, and `manage_localization`
- Schema remove operations, because those are destructive model changes

`dependent-delete-denials` covers:
- Dependency-aware delete rules for `manage_semantic` `delete_measure`, `delete_kpi`, `delete_calc_group`, `delete_udf`, `delete_named_expression`
- Dependency-aware delete rules for `manage_schema` `delete_partition`, `delete_column`, `delete_calc_column`, `delete_relationship`, `delete_hierarchy`, `delete_calendar`
- Dependency-aware delete rules for `manage_security` `delete_role`, `delete_perspective`
- Dependency-aware delete rules for `manage_localization` `delete_culture`

`table-delete-impact-review` covers:
- Table deletes with severe blast radius are denied
- Table deletes with smaller but non-zero blast radius require confirmation

`unsupported-operation-denials` covers:
- Static fallback deny rules for unsupported destructive operations: `manage_semantic` `delete_pq_parameter`, `manage_schema` `remove_time_unit`, `manage_schema` `remove_related_group`

### List Rules (Paginated)

```json
{ "operation": "list", "limit": 50, "offset": 0 }
```

Response includes `count` (total) and a `pagination` object with `total`, `has_more`, `next_offset`. Default limit: 50, max: 1000 (configurable via `MCP_ENGINE_PAGINATION_*` env vars).

## Condition Reference

Policy conditions support:

- `not` (logical negation)
- `arg_equals`, `arg_in`, `arg_present`, `arg_missing`
- `arg_gt`, `arg_gte`, `arg_lt`, `arg_lte` (numeric comparisons)
- `arg_regex`, `arg_regex_not`
- `bulk_any`, `bulk_all` (for `items[]` operations)
- `all_of`, `any_of` (logical composition)
- `tool_has_tag`, `model_path_pattern`, `model_hash_in`

Condition JSON uses a `kind` discriminator (e.g., `{ "kind": "arg_equals", "field": "operation", "value": "delete" }`).

For Power BI Desktop models on Windows, `model_path_pattern` uses case-insensitive, culture-invariant matching. Its `*` wildcard matches zero or more characters and `?` matches exactly one character. The condition does not normalize separators, resolve relative paths, or access the filesystem. Generic `arg_regex` and `arg_regex_not` conditions remain case-sensitive.

Field paths:
- `field` supports dot-notation for nested args (e.g., `spec.description`, `spec.format_string_expression`).
- JSON-in-string arguments that contain an object or array are exposed under `parsed.<argName>` during policy evaluation.
- Use `[]` to traverse arrays (for example, `parsed.data.rules[].action` or `parsed.data.scopes.global.settings[].id`).
- If a rule references `parsed.<argName>` and that string fails to parse as JSON, policy evaluation records a context-enrichment failure and follows the configured failure mode.

### Dependency Facts (Evaluation-Only)

For supported delete operations, policy evaluation injects a synthetic `dependency` namespace before rule matching. These fields are evaluation-only and are not part of the external tool schema:

- `dependency.has_dependents`: `true` when at least one downstream dependent was found
- `dependency.count`: total dependent hit count
- `dependency.types`: distinct dependent types, sorted case-insensitively
- `dependency.sample_ids`: deterministic sorted sample of up to 5 dependent IDs
- `dependency.sample_summary`: concise semicolon-separated summary of up to 5 sampled dependents, suitable for custom policy messages
- `dependency.samples`: structured deterministic sample of up to 5 dependent objects; each sample includes `type`, `name`, `table`, `id`, and `summary`
- `dependency.samples_truncated`: `true` when more dependency samples were found than are included in `dependency.samples`

Sample fields expose only object type, table/name, and ID details. Expression text and other dependency payloads are intentionally omitted from these policy facts.

Dependency facts currently apply to these operations:
- `manage_semantic`: `delete_measure`, `delete_kpi`, `delete_calc_group`, `delete_udf`, `delete_named_expression`
- `manage_schema`: `delete_table`, `delete_partition`, `delete_column`, `delete_calc_column`, `delete_relationship`, `delete_hierarchy`, `delete_calendar`
- `manage_security`: `delete_role`, `delete_perspective`
- `manage_localization`: `delete_culture`

Out of scope for dependency facts in this release:
- `manage_semantic` `delete_pq_parameter`
- `manage_schema` `remove_time_unit`
- `manage_schema` `remove_related_group`
- Non-model destructive operations such as changeset or checkpoint deletion

Dependency analysis for policy facts uses high-confidence structural/expression hits only. Metadata-only mentions are excluded from enforcement facts in this release.

Bulk semantics:
- Dependency facts are computed per effective `items[]` entry only
- No top-level aggregate `dependency.*` facts are added for the whole bulk request
- Use `bulk_any` / `bulk_all` when authoring bulk dependency-aware rules

Failure semantics:
- If dependency enrichment fails for a supported delete, any rule that dereferences `dependency.*` hits the normal condition-evaluation error path
- `fail_open`: dependency-aware rules become non-matches and unrelated rules continue
- `fail_closed`: the evaluation returns a distinct `processing_failure` decision with no matched rule
- Rules that do not reference `dependency.*` continue evaluating normally

Example (deny delete when a measure still has dependents):

```json
{
  "operation": "import",
  "data": "{ \"rules\": [ { \"id\": \"deny-dependent-measure-delete\", \"action\": \"deny\", \"tool\": \"manage_semantic\", \"condition\": { \"kind\": \"all_of\", \"conditions\": [ { \"kind\": \"arg_equals\", \"field\": \"operation\", \"value\": \"delete_measure\" }, { \"kind\": \"arg_equals\", \"field\": \"dependency.has_dependents\", \"value\": true } ] }, \"message\": \"Deleting this measure is blocked because it has {arg:dependency.count} downstream dependents: {arg:dependency.sample_summary}.\" } ] }"
}
```

Example (bulk delete guard using per-item dependency facts):

```json
{
  "operation": "import",
  "data": "{ \"rules\": [ { \"id\": \"deny-bulk-delete-when-any-item-has-dependents\", \"action\": \"deny\", \"tool\": \"manage_semantic\", \"condition\": { \"kind\": \"bulk_any\", \"predicate\": { \"kind\": \"arg_equals\", \"field\": \"dependency.has_dependents\", \"value\": true } }, \"message\": \"At least one bulk delete target still has downstream dependents.\" } ] }"
}
```

Example (numeric threshold for high-impact table deletes):

```json
{
  "operation": "import",
  "data": "{ \"rules\": [ { \"id\": \"deny-severe-table-delete\", \"action\": \"deny\", \"tool\": \"manage_schema\", \"condition\": { \"kind\": \"all_of\", \"conditions\": [ { \"kind\": \"arg_equals\", \"field\": \"operation\", \"value\": \"delete_table\" }, { \"kind\": \"arg_gte\", \"field\": \"dependency.count\", \"value\": 25 } ] }, \"message\": \"Deleting this table is blocked because it has {arg:dependency.count} downstream dependents.\" } ] }"
}
```

`model_hash_in` is kept as the condition name for compatibility, but it matches the current persisted model scope ID. On desktop models this is the stable model ID when available.

Message templates:
- `{arg:field.path}` supports the same dot-notation, `parsed.<argName>`, and `[]` traversal semantics as policy conditions.
- `{tool}`, `{model.path}`, and `{model.hash}` remain supported.
- Invalid placeholders are rejected during policy validation and import/write operations.

## Admin Policy Bundle

Administrators can deploy a centrally-managed policy bundle that locks out local policy modifications.

**⚠️ Requires Enterprise tier.** The bundle is only activated when the Enterprise license is present. On Pro or Free tiers, the bundle is ignored and normal local policy behavior applies.

### Configuration

Set the environment variable:
```
MCP_ENGINE_POLICY_BUNDLE_PATH=/path/to/policy-bundle.json
```

### Bundle Format

```json
{
  "version": "1.0",
  "scope": "global",
  "rules": [
    {
      "id": "block-deletes",
      "action": "deny",
      "tool": "manage_*",
      "condition": { "kind": "arg_equals", "field": "operation", "value": "delete" },
      "message": "Delete operations blocked by admin policy."
    }
  ]
}
```

### Behavior When Bundle Is Active (Enterprise Tier)

- **Local policies ignored**: The `global.json` and `model_*.json` files are not loaded.
- **Model scope disabled**: Requesting `scope: "model"` returns an error.
- **Write operations blocked**: `put`, `delete`, `import`, and `reset` return `POLICY_LOCKED_BY_BUNDLE`.
- **Status shows bundle info**: `manage_policy status` includes `bundle_active: true`, `bundle_source`, `bundle_file`.
- **Policy lockdown source**: Bundle/config controls enforce policy readonly behavior even if local policy rules would otherwise allow writes.

### Behavior When Bundle Is Configured but Not Enterprise

When `MCP_ENGINE_POLICY_BUNDLE_PATH` is set but the license is Pro or Free:
- **Bundle is ignored**: Local policies work normally (`global.json`, `model_*.json`).
- **Write operations allowed**: Policy CRUD operations proceed as usual.
- **Status shows ignored state**: `manage_policy status` includes:
  - `bundle_configured: true`
  - `bundle_active: false`
  - `bundle_ignored_reason: "requires_enterprise"`
  - `bundle_required_tier: "enterprise"`

### Bundle Load Failure (Enterprise Tier)

If the bundle file is missing, inaccessible, or contains invalid JSON:
- All tools are denied except an allowlist: `manage_policy`, `manage_policy_ui`, `manage_license`, `manage_license_ui`, `manage_model_connection`, `manage_model_connection_ui`, `list_model`, `run_query`
- This lockdown prevents model modifications until the admin bundle issue is resolved
- `bundle_error` field in `manage_policy status` shows the failure reason
- **No automatic retry**: The server attempts to load the bundle once at first use; a server restart is required after fixing the bundle file

## Notes

- Policy engine requires Pro tier licensing.
- Admin Policy Bundle requires Enterprise tier licensing.
- Model-scope rules are evaluated before global-scope rules.
- When admin bundle is configured with Enterprise tier, writes are blocked immediately (even before bundle loads).
- Tool schemas are cached at startup. If the license tier changes at runtime, a server restart is required for the `manage_policy` tool schema to reflect the new available operations.

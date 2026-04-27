# Policy Presets

Use built-in policy packs first. Preview with `dry_run: true` before applying, and ask for approval before any persistent policy write.

## Built-In Packs

Use `manage_policy` with:

```json
{
  "operation": "packs_apply",
  "scope": "global",
  "spec": { "pack_id": "safe-authoring-baseline" },
  "dry_run": true
}
```

Available pack IDs:

- `safe-authoring-baseline`: confirmation and safety rails for common authoring mistakes.
- `semantic-quality`: semantic model quality guardrails.
- `destructive-change-protection`: impact-aware protection for destructive deletes and unsupported destructive operations.
- `masking-settings-approval`: requires confirmation before masking preference changes.
- `security-hardening`: guardrails for role and perspective changes.
- `refresh-and-partition-safety`: blocks broad refresh execution and partition refresh paths.

## Recommendation Matrix

For confirm-heavy solo or developer setup:

- Start with `safe-authoring-baseline`.
- Add `semantic-quality` when model authoring quality is the main goal.
- Add `masking-settings-approval` when masking settings are enabled or likely to change.

For deny destructive operations:

- Use `destructive-change-protection`.
- Add `refresh-and-partition-safety` when refresh or partition operations are not part of the approved workflow.
- Add `security-hardening` for roles, OLS, RLS, perspectives, or team-admin settings.

For consultant/client work:

- Start with previews only.
- Prefer `masking-settings-approval` and `safe-authoring-baseline`.
- Apply `destructive-change-protection` only after confirming the client accepts server-side blocks.

For team/admin governance:

- Prefer deny-style rules for high-risk operations.
- Use `security-hardening`, `destructive-change-protection`, and `refresh-and-partition-safety`.
- Keep Enterprise-only policy bundle or audit recommendations separate from local Pro policy packs.

## Custom Rules

Only propose custom rules when built-in packs do not cover the need. Keep custom rules narrow:

- One intent per rule.
- Specific tool and operation.
- Clear message.
- Reviewable conditions.
- No broad deny-all policy unless the user explicitly asks for lockdown.

Validate custom policy payloads before import when possible. Do not apply custom policy rules until the user approves the exact JSON.

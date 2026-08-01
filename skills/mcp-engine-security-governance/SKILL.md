---
name: mcp-engine-security-governance
description: Use when creating or changing RLS roles, role filters, OLS permissions, perspectives, allow/deny/confirm policy rules, PII or numeric masking, or audit logging and evidence export, and when deciding whether a request needs security enforcement or only curation. For first-time guided setup of policies, masking, and guardrails, use mcp-engine-onboarding.
---

# PBI Security Governance

Security enforcement and governance through `manage_security`, `manage_policy`, and `manage_audit`. Keep curation separate from access control.

## Start with the right category

- Perspectives (`create_perspective`, `update_perspective`) are curation — they change what users see, not what they can access.
- Roles, OLS, policies, masking, and audit are enforcement — fail closed by default unless the user explicitly changes that posture.
- Confirm the current model with `manage_model_connection` `{ "operation": "get_current" }` and state the intended scope before modifying anything security-sensitive.

## Branch by workflow

- RLS and OLS: `manage_security` (`create_role`, `set_role_filters`, `set_role_permissions`) — read [security-roles-guide](references/security-roles-guide.md) first. For RLS, validate each affected role with `run_query` `{ "operation": "test_access", "query": "<small aggregate over the filtered data>", "spec": { "roles": ["<role>"] } }` (both `query` and `spec.roles` are required). For OLS, exercise every changed table or column directly under the affected principals and verify each expected allow or denial; use targeted `test_access` queries or a `manage_tests` `ols_validation` matrix. An unrelated aggregate is not OLS evidence.
- Perspectives: [perspectives-guide](references/perspectives-guide.md) — when the user wants curated field visibility, not restricted access.
- Policy rules and packs: `manage_policy` (`status`, `evaluate`, `put`, `packs_apply` with `dry_run: true` first) — [policy-guide](references/policy-guide.md). Prefer deny rules over `require_confirm` for high-risk operations; `require_confirm` is best-effort UX, not a portable control.
- Masking: [pii-masking-guide](references/pii-masking-guide.md) for sensitive-value redaction and masking behavior.
- Audit: [audit-logging-guide](references/audit-logging-guide.md) for durable trails, export verification, and integrity checks. Treat auditability as part of correctness for write and governance flows.

## Guardrails

- Confirm intent with the user before destructive or broad security changes.
- Never present a perspective as protection; say explicitly when a request needs RLS or OLS instead.
- Do not echo sensitive values, filter expressions over PII, or masked data in summaries, examples, or logs.

## Report results

After security or governance work, report:

1. Roles, permissions, policies, or masking rules created or changed.
2. `test_access` or `ols_validation` results for each affected role, principal, and protected object.
3. Policy evaluation or dry-run outcomes before application.
4. The enforcement posture after the change — what is now denied, confirmed, masked, or audited.

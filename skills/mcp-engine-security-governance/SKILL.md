---
name: mcp-engine-security-governance
description: Handle Power BI security and governance workflows. Use when defining or validating RLS or OLS, perspectives, policies, masking rules, or audit workflows, and when distinguishing between security enforcement and model curation.
---

# PBI Security Governance

Use this skill for security enforcement and governance work. Keep curation concerns separate from true access control.

## Start with the right category

- Treat perspectives as curation, not access control.
- Treat roles, object security, policies, masking, and auditability as enforcement concerns.
- Confirm the current model and intended scope before modifying anything security-sensitive.

## Branch by workflow

### RLS, OLS, and perspectives

- Read [security-roles-guide](references/security-roles-guide.md) before changing RLS or OLS.
- Read [perspectives-guide](references/perspectives-guide.md) when the user wants curated field visibility rather than restricted access.
- Validate role behavior with targeted test queries after making changes.

### Policies and masking

- Read [policy-guide](references/policy-guide.md) when the task involves allow, deny, or confirmation rules.
- Read [pii-masking-guide](references/pii-masking-guide.md) when the task involves sensitive-value redaction or masking behavior.
- Keep enforcement fail-closed by default unless the user explicitly changes that posture.

### Audit workflows

- Read [audit-logging-guide](references/audit-logging-guide.md) when the task involves durable audit trails, export verification, or integrity checks.
- Treat auditability as part of correctness for write and governance flows.

## Guardrails

- Confirm intent before destructive or broad security changes.
- Separate perspective work from RLS or OLS work so the user does not mistake curation for protection.
- Re-test the affected principals or policies after changes.
- Avoid exposing sensitive values in summaries, examples, or logs.

## References

- [security-roles-guide](references/security-roles-guide.md)
- [perspectives-guide](references/perspectives-guide.md)
- [policy-guide](references/policy-guide.md)
- [pii-masking-guide](references/pii-masking-guide.md)
- [audit-logging-guide](references/audit-logging-guide.md)

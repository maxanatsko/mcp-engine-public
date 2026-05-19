# Playbook Template

Use this structure for the final onboarding output. Keep it plain-language, short, and specific to the user's setup choices.

## Setup Profile

Summarize:

- License tier and activation status.
- Server mode: full, read-only, or browse-only.
- Connection target: Desktop, Service/XMLA, both, or deferred.
- Primary workflow: local development, shared review, governance, testing, AI readiness, performance troubleshooting, or support.
- Sensitivity posture.

## Applied Or Previewed Setup

Group by state:

- Applied: approved changes already made.
- Previewed only: packs or tests shown but not applied.
- Recommended next: useful setup not yet approved.
- Deferred: requires a model, Pro/Enterprise, admin setup, or user-provided details.
- Explicitly not changed: important for trust and safety.

## Masking

Include:

- PII masking choice.
- Numeric masking choice.
- Force-mask tables/columns, if any.
- Exclude tables/columns, if any.
- Reminder that masking helps with safe sharing but is not access control.

## Guardrails

Include:

- Built-in policy packs selected, previewed, or applied.
- Custom guardrails requested by the user.
- Whether deletes, renames, security edits, masking changes, refresh, or partition operations are blocked, confirm-required, or unchanged.
- Note when policy edits are unavailable due to mode, tier, or Enterprise admin bundle.

## Preferences And Memory

Include:

- Naming conventions saved or recommended.
- Glossary and aliases saved or recommended.
- Row limits, formatting defaults, or response style.
- Scope: global, workspace, or model.
- Reminder that secrets and row-level extracts were not stored.

## Change Safety

Include:

- Dependency-check preference.
- Checkpoint preference.
- Undo/redo or rollback posture.
- Whether broad edits should require a checkpoint and validation first.

## Tests

Include:

- Test packs previewed or applied.
- Custom tests requested.
- Tests deferred because expected business values, role identities, or a connected model are missing.
- How the user should run or export tests next.

## Reports, Diagnostics, And RLS

Include:

- Model report choice and intended audience.
- Advanced query diagnostics preference.
- VertiPaq/performance resources to use when needed.
- RLS effective-user testing choice and approved identities, if any.

## Enterprise Posture

Include only when relevant:

- Audit logging status or next action.
- Admin policy bundle status.
- Any admin handoff message needed.
- Reminder that centrally managed policy was not overridden.

## Next Session Prompts

Provide copy/paste prompts tailored to the user's choices, such as:

- "Show my current license tier, server mode, connected model, and active masking settings."
- "Before editing this model, check dependencies and create a checkpoint if the change is broad."
- "Preview the recommended policy packs for this environment; do not apply until I confirm."
- "Preview metadata-quality tests for this model and explain what each test covers."
- "Generate a concise model overview for a business owner; avoid data values."
- "Use advanced diagnostics to investigate this slow DAX query, but ask before collecting traces."
- "Test this RLS role as [user@domain.com] and summarize only aggregate results."

## Safety Notes

List only the notes relevant to the user:

- No license key was echoed or stored in preferences.
- No model changes were made during onboarding unless explicitly approved.
- No policy/test pack was applied without preview.
- No sensitive sample data was stored as memory.
- No audit evidence was exposed unless explicitly requested and approved.
- Desktop refresh status is best-effort diagnostics, not durable progress tracking.

---
name: mcp-engine-onboarding
description: Use when a user types onboarding, asks to set up SemanticOps MCP, wants help choosing Free vs Pro, or wants licensing, modes, masking, guardrails, preferences, model safety, tests, reporting, diagnostics, RLS testing, or Enterprise posture tailored to their workflow. For changing security, policy, or masking on an already-configured server, use mcp-engine-security-governance.
---

# PBI Onboarding

Use this skill to turn "onboarding" into a regular-user setup concierge. This is an orchestration workflow over existing SemanticOps MCP tools, not a runtime MCP tool.

## Start Here

1. Start with: "I'll ask a series of questions to tailor your SemanticOps MCP setup."
2. Speak to regular users. Avoid jargon like CRUD, AST, resource/action, internal feature, or raw payload unless the user asks.
3. Use [feature-setup-guide](references/feature-setup-guide.md) as the source-of-truth checklist for Pro and Enterprise coverage.
4. Ask one feature-area question at a time. Explain the feature, give examples, offer choices, recommend a default, and say what would be changed, previewed, or deferred.
5. Check current license, mode, connection, preferences, and policy state when SemanticOps MCP tools are available.
6. Produce a setup action plan before any activation, write, pack application, model selection, validation query, test creation, checkpoint, report generation, or audit operation.
7. Ask for explicit approval before calling tools that activate licenses, mutate preferences, policies, tests, model metadata, select a model, run validation queries, create checkpoints, generate reports, or access audit logs.

## Workflow

- Read [questionnaire](references/questionnaire.md) for the short interview, defaults, and follow-up questions.
- Read [feature-setup-guide](references/feature-setup-guide.md) to cover all user-configurable Pro and Enterprise feature areas.
- Read [setup-profiles](references/setup-profiles.md) to map answers to recommended connection, mode, workflow, and safety posture.
- Read [policy-presets](references/policy-presets.md) when recommending `manage_policy` packs or custom policy rules.
- Read [test-pack-presets](references/test-pack-presets.md) when recommending `manage_tests` packs or starter validation tests.
- Read [playbook-template](references/playbook-template.md) when producing the final user playbook.

## Tool Usage

Use existing SemanticOps MCP surfaces only:

- `manage_license` for license status, activation, refresh, and tier explanation.
- `manage_model_connection` for connection discovery, selection, and current state.
- `list_model` for metadata grounding after a model is connected.
- `manage_preferences` for approved runtime preference changes.
- `manage_policy` for approved policy pack previews or applications.
- `manage_tests` for approved test pack previews or applications.
- `manage_model_changes` for approved checkpoints, change history, undo/redo, and rollback planning.
- `manage_dependencies` for approved impact checks before risky changes.
- `run_query` for approved validation queries and Pro query diagnostics.
- `manage_audit` only when Enterprise is active and the user asks for audit posture or evidence.

Always preview pack application first when the tool supports it:

- `manage_policy` with `operation: "packs_apply"`, `spec.pack_id`, and `dry_run: true`.
- `manage_tests` with `operation: "packs_apply"`, `spec.pack_id`, and `dry_run: true`.

Full guided setup is allowed only after the user approves the exact batch. Approved setup can include:

- License activation through `manage_license`.
- Connection selection through `manage_model_connection`.
- Runtime preference changes through `manage_preferences`.
- Policy pack preview and application through `manage_policy`.
- Test pack preview and application through `manage_tests`.
- Checkpoint or change-safety setup through `manage_model_changes`.
- Impact checks through `manage_dependencies`.
- Targeted validation queries through `run_query`.

## Guardrails

- Do not auto-apply preferences, policies, tests, or model changes from interview answers alone.
- Do not echo license keys back to the user.
- Do not use or reintroduce the removed `onboarding_completed` preference.
- Do not collect or store sensitive business data in the onboarding notes.
- Respect SemanticOps MCP mode, license, policy, confirmation, and audit gates.
- Keep Enterprise-only recommendations clearly separated from local Pro workflows.
- Treat `require_confirm` policy rules as user-experience guardrails, not a portable compliance control across every MCP client.
- Prefer deny-style policy for high-risk operations when the user needs server-enforced protection.
- If no model is connected, produce the general plan and mark model-specific actions as deferred.
- Do not overclaim Desktop refresh observability. Desktop refresh status is best-effort diagnostics, not durable progress tracking.

## Output Standard

Return a compact onboarding packet:

1. License and mode posture.
2. Masking choices.
3. Guardrails previewed, applied, or deferred.
4. Preferences and memory saved or recommended.
5. Model-change safety and dependency-analysis choices.
6. Test setup.
7. Reporting, diagnostics, and RLS testing choices.
8. Enterprise audit or admin-bundle posture when relevant.
9. What was changed, previewed, deferred, and explicitly not changed.
10. Copy/paste prompts for the user's next sessions.

---
name: mcp-engine-onboarding
description: Guide new MCP Engine users through setup planning, safe defaults, preferences, policies, test packs, connection choices, masking posture, and a tailored Power BI workflow playbook. Use when a user asks to set up MCP Engine for their workflow, choose safe configuration, onboard a team/client/model, or discover the right MCP Engine tools and skills to use first.
---

# PBI Onboarding

Use this skill to turn an unclear MCP Engine starting point into a concrete setup plan and a practical operating playbook. This is an orchestration workflow over existing MCP Engine tools, not a runtime MCP tool.

## Start Here

1. Interview the user before recommending changes.
2. Check current connection and mode when MCP Engine tools are available, but do not require a connected model to produce a general setup plan.
3. Produce a setup plan before any write or validation call.
4. Ask for explicit approval before calling tools that mutate preferences, policies, tests, model metadata, or execute optional validation queries.
5. Prefer built-in policy and test packs before generating custom payloads.

## Workflow

- Read [questionnaire](references/questionnaire.md) for the short interview, defaults, and follow-up questions.
- Read [setup-profiles](references/setup-profiles.md) to map answers to recommended connection, mode, workflow, and safety posture.
- Read [policy-presets](references/policy-presets.md) when recommending `manage_policy` packs or custom policy rules.
- Read [test-pack-presets](references/test-pack-presets.md) when recommending `manage_tests` packs or starter validation tests.
- Read [playbook-template](references/playbook-template.md) when producing the final user playbook.

## Tool Usage

Use existing MCP Engine surfaces only:

- `manage_model_connection` for connection discovery, selection, and current state.
- `list_model` for metadata grounding after a model is connected.
- `manage_preferences` for approved runtime preference changes.
- `manage_policy` for approved policy pack previews or applications.
- `manage_tests` for approved test pack previews or applications.
- `run_query` only for approved, targeted validation queries.

Always preview pack application first when the tool supports it:

- `manage_policy` with `operation: "packs_apply"`, `spec.pack_id`, and `dry_run: true`.
- `manage_tests` with `operation: "packs_apply"`, `spec.pack_id`, and `dry_run: true`.

## Guardrails

- Do not auto-apply preferences, policies, tests, or model changes from interview answers alone.
- Do not collect or store sensitive business data in the onboarding notes.
- Respect MCP Engine mode, license, policy, confirmation, and audit gates.
- Keep Enterprise-only recommendations clearly separated from local Pro workflows.
- Treat `require_confirm` policy rules as user-experience guardrails, not a portable compliance control across every MCP client.
- Prefer deny-style policy for high-risk operations when the user needs server-enforced protection.
- If no model is connected, produce the general plan and mark model-specific actions as deferred.

## Output Standard

Return a compact onboarding packet:

1. Profile summary.
2. Recommended connection and mode posture.
3. Preference recommendations.
4. Policy pack and rule recommendations.
5. Test pack and starter validation recommendations.
6. Relevant docs, resources, and skills to use next.
7. Approval checklist for any tool calls.
8. Tailored daily, safe-change, validation, and troubleshooting playbook.

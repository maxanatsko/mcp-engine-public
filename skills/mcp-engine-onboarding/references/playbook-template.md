# Playbook Template

Use this structure for the final onboarding output. Keep it short, operational, and specific to the user's profile.

## Profile

Summarize:

- Persona.
- Connection target.
- Mode posture.
- Sensitivity level.
- Risk tolerance.
- Main workflows.
- Model scope.

## Recommended Setup

List:

- Preferences to inspect or set.
- Policy packs to preview or apply.
- Test packs to preview or apply.
- Model-specific actions that require a connection.
- Docs/resources/skills to use next.

Separate items into:

- Approved and applied.
- Approved preview only.
- Recommended but not applied.
- Deferred until a model is connected.

## Daily Workflow

Include the normal loop:

1. Confirm connection with `manage_model_connection`.
2. Inspect model context with `list_model`.
3. Use the relevant skill for the task: `mcp-engine-query`, `mcp-engine-semantic-authoring`, `mcp-engine-schema-authoring`, `mcp-engine-security-governance`, `mcp-engine-testing-changes`, or `mcp-engine-ai-readiness`.
4. Keep tool payloads aligned with `docs://tool-invocation-conventions`.

## Safe-Change Workflow

Include:

- Confirm mode and policy status.
- Create or verify a checkpoint when broad edits are planned.
- Preview policy or test pack changes first.
- Make one narrow change batch at a time.
- Run targeted validation before broader suites.

## Validation Workflow

Include:

- Preview recommended `manage_tests` packs with `dry_run: true`.
- Save generated tests only after approval.
- Run the smallest useful scope first.
- Export or summarize results when the user needs evidence.

## Troubleshooting Workflow

Include:

- Re-check current connection.
- Reload metadata after external Power BI changes.
- Use read-only discovery before write retries.
- Ask for a short failure window and exact tool payload for host/client differences.

## Example Prompts

Provide prompts tailored to the profile, such as:

- "Check my current MCP Engine connection and tell me what mode and model I am using."
- "Preview the recommended policy packs for this workflow, but do not apply them yet."
- "Preview starter test packs for this model and explain what each generated test would cover."
- "Help me make this model change safely, with dependency impact and validation before edits."
- "Assess this model for AI readiness and draft the next improvements."

## Avoid

List concrete cautions:

- Do not run broad refresh or destructive write operations without explicit approval.
- Do not treat perspectives as security controls.
- Do not rely on confirmation prompts as the only compliance control.
- Do not expose sensitive values in summaries or examples.
- Do not apply model-specific tests or preferences before the target model is confirmed.

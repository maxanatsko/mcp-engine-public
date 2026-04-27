# Setup Profiles

Use profiles to turn questionnaire answers into recommendations. A user can match more than one profile; combine conservatively and call out tradeoffs.

## Solo Analyst

Best fit:

- One PBIX or a small number of local models.
- Mostly asks questions, validates measures, and reviews metadata.
- Low need for shared governance.

Recommend:

- Start with Power BI Desktop connection.
- Use full mode locally if the user intends to author changes; otherwise read-only.
- Enable safe preference defaults for row limits, formatting, and masking based on sensitivity.
- Recommend `safe-authoring-baseline` only if the user will make model changes.
- Recommend `metadata-quality` or `presentation-hygiene` test packs after a model is connected.

## Power BI Developer

Best fit:

- Builds or refactors semantic models.
- Needs repeatable validation and reversible changes.
- Accepts controlled write workflows.

Recommend:

- Full mode for local development with explicit checkpoints and validation.
- `safe-authoring-baseline` for confirmation rails.
- `destructive-change-protection` when deleting or renaming model objects is possible.
- `semantic-quality` for semantic model quality guardrails.
- `metadata-quality`, `referential-integrity`, `relationship-governance`, and `time-intelligence` test packs depending on model features.
- Use `mcp-engine-testing-changes` before broad refactors and `mcp-engine-semantic-authoring` or `mcp-engine-schema-authoring` for authoring workflows.

## Consultant

Best fit:

- Works across many client models.
- Needs portable habits and minimal accidental data exposure.
- Often needs a client-ready playbook.

Recommend:

- Read-only or browse-only until the client approves write scope.
- Treat data as sensitive by default.
- Prefer global preferences for portable limits and masking, with model scope only when a specific client model is approved.
- `masking-settings-approval`, `safe-authoring-baseline`, and `destructive-change-protection` when write access is enabled.
- `documentation-baseline`, `metadata-quality`, and `presentation-hygiene` test packs for handoff quality.
- Keep final playbook explicit about what was changed, what was only recommended, and what needs client approval.

## Admin Or Team Lead

Best fit:

- Shared workspaces, team standards, governance, and support workflows.
- Needs durable guardrails more than ad hoc convenience.

Recommend:

- Read-only as the default shared posture; full mode only for trusted maintainers.
- Browse-only for broad discovery or support users.
- Separate local Pro recommendations from Enterprise/org-managed policy or audit expectations.
- `security-hardening`, `refresh-and-partition-safety`, `destructive-change-protection`, and `masking-settings-approval`.
- `documentation-baseline`, `relationship-governance`, `referential-integrity`, and security validation starter tests when roles or OLS are present.
- Use `mcp-engine-security-governance` and `mcp-engine-testing-changes` as follow-up skills.

## AI Readiness

Best fit:

- User wants Copilot, Fabric data agent, or natural-language Q&A readiness.

Recommend:

- Metadata-level inspection before data-value queries.
- `metadata-quality`, `documentation-baseline`, and `presentation-hygiene` test packs.
- `semantic-quality` policy pack when the user will make authoring changes.
- Route deeper readiness assessment to `mcp-engine-ai-readiness`.
- Keep Prep data for AI artifacts as drafts unless the user has a supported PBIP/Git/manual workflow.

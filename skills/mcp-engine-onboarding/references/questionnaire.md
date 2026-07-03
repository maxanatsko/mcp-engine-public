# Onboarding Questionnaire

Use this script when the user types "onboarding" or asks to set up SemanticOps MCP. The goal is to walk a regular user through setup choices in plain language. Ask one section at a time; do not dump the whole questionnaire at once.

Opening line:

"I'll ask a series of questions to tailor your SemanticOps MCP setup. I can help with the Free version, activate Pro or Enterprise if you have a key, and set up safety, masking, guardrails, memory, tests, and workflow defaults."

## Interview Rules

Each step must:

- Explain the feature in user language.
- Give a concrete example.
- Offer 2-5 choices with a recommended default.
- Say what will be changed, previewed, or deferred.
- Ask approval before any tool call that activates, writes, generates, selects, queries, checkpoints, rolls back, or reads audit evidence.

Avoid these unless the user asks for implementation detail:

- CRUD
- AST
- resource/action pattern
- internal feature
- raw JSON payloads

## 1. License And Tier

Say:

"Before we start, do you have your Pro or Enterprise key handy, or would you like to continue with the Free version?"

Explain:

- Free supports core semantic-model work where your server mode allows it: connecting, browsing, querying, refresh, and model editing.
- Pro adds the safety and workflow layer: masking, guardrails, dependency impact checks, checkpoints, tests, model/workspace memory, model reports, and advanced diagnostics.
- Enterprise adds compliance and centrally managed governance features like audit logs and admin policy bundles.

Choices:

- Check current tier first. Recommended.
- Continue with Free for now.
- Activate Pro or Enterprise.
- Explain what I get with Pro before deciding.

If activating:

- Tell the user not to paste the key into shared chats.
- Ask them to paste it only after confirming you will not echo it back.
- After activation, show tier and enabled features in plain English.

Allowed tools after approval:

- `manage_license` status, activate, refresh.

Links:

- Wiki: `docs/wiki/tools/manage_license.mdx`
- Pricing: `/pricing`

## 2. Server Mode

Say:

"Next, let's choose how much power the assistant should have in this environment: full, read-only, or browse-only."

Explain:

- Full mode: can read, query, refresh, and make approved model changes.
- Read-only mode: can inspect and query, but cannot change the model.
- Browse-only mode: can inspect structure safely, but cannot run queries or make edits.

Choices:

- Full mode for local development.
- Read-only for shared or production models. Recommended for shared models.
- Browse-only for regulated or first-look environments.
- Not sure; check current mode and explain what is allowed.

Allowed tools after approval:

- `manage_model_connection` get current state.

Links:

- Wiki: `docs/wiki/concepts/modes-and-restrictions.mdx`

## 3. Connection Context

Say:

"Do you want me to connect to a model now, or keep setup general until you open the right PBIX or Service dataset?"

Choices:

- Connect to the current Power BI Desktop model.
- Connect to a Power BI Service/XMLA dataset.
- List available models and let me choose.
- Keep setup general for now. Recommended when no model is ready.

Allowed tools after approval:

- `manage_model_connection` get_current, list, authenticate, select.
- `list_model` only after a model is connected.

## 4. Masking

Say:

"SemanticOps MCP supports masking so results are safer to share. PII masking hides things like emails, phone numbers, IDs, or payment-card-like values. Numeric masking hides sensitive numbers like revenue, margin, payroll, forecasts, or confidential metrics."

Example:

- PII masking can turn `alex@example.com` into a masked value.
- Numeric masking can hide sensitive finance values while still letting us discuss the shape of the result.

Ask:

"What masking would you like to use?"

Choices:

- Turn on PII masking.
- Turn on numeric masking.
- Turn on both. Recommended for customer, people, finance, or regulated models.
- Keep both off for now.
- Review the model first and decide force/exclude lists.

Follow-up:

"If you know certain tables or columns are always sensitive, I can force masking for them. If you know some lookup tables are safe, I can exclude them. Otherwise, we can leave lists empty for now."

Allowed tools after approval:

- `manage_preferences` settings for `pii_masking_enabled`, `numeric_masking_enabled`, force lists, and exclude lists.

Links:

- `docs://pii-masking-guide`
- Wiki: `docs/wiki/tools/manage_preferences-settings.mdx`

## 5. Guardrails

Say:

"Guardrails are hard rules for the assistant. Preferences are guidance; guardrails enforce what is allowed, blocked, or must be confirmed."

Examples:

- Block deletes.
- Ask before renaming objects.
- Require descriptions on new measures.
- Protect masking settings.
- Block broad refresh or partition operations.

Ask:

"Would you like to preview built-in guardrail packs?"

Choices:

- Safe authoring and destructive-change baseline. Recommended when edits are allowed.
- Semantic quality.
- Masking and security hardening. Recommended if masking or roles matter.
- Refresh and partition safety.
- Skip for now, or describe a custom guardrail you want.

Allowed tools after approval:

- `manage_policy` packs_list, packs_apply with dry_run first.
- `manage_policy` validate/import for approved custom rules.

Links:

- `docs://policy-guide`
- Wiki: `docs/wiki/tools/manage_policy.mdx`

## 6. Preferences And Memory

Say:

"Preferences are memory for how you want me to work. They do not change your model by themselves; they guide future sessions."

Examples:

- Measures use PascalCase.
- Always add descriptions and display folders.
- GM means Gross Margin.
- Keep query results to 200 rows.
- Use model-specific conventions for one dataset and different conventions elsewhere.

Ask:

"Do you have conventions or memory you want saved?"

Choices:

- Naming conventions for measures, tables, or columns.
- Business glossary or abbreviations.
- Default behavior such as row limits or formatting.
- Model or workspace-specific memory. Recommended when Pro is active and a model/workspace is connected.
- Skip memory for now.

Allowed tools after approval:

- `manage_preferences` set/list/export/import.

Rules:

- Do not store secrets, license keys, passwords, connection strings, tokens, or row-level data extracts.
- Use global scope for portable conventions.
- Use model/workspace scope only when Pro and current connection make it available.

Links:

- Wiki: `docs/wiki/tools/manage_preferences.mdx`
- Wiki: `docs/wiki/tools/manage_preferences-scopes.mdx`

## 7. Change Safety And Impact Checks

Say:

"If you plan to let the assistant edit models, Pro can add a safety net: checkpoints, change history, undo/redo, rollback, and dependency checks before risky edits."

Ask:

"How cautious should I be before model changes?"

Choices:

- Always check dependencies before deletes or renames. Recommended.
- Create checkpoints before broad edits.
- Ask me before each checkpoint or impact check.
- Use this only for shared or production models.
- Skip change safety for now.

Allowed tools after approval:

- `manage_dependencies` for impact checks.
- `manage_model_changes` for checkpoints, history, undo/redo, rollback planning.

Links:

- `docs://dependencies-guide`
- `docs://model-changes-guide`
- Wiki: `docs/wiki/tools/manage_dependencies.mdx`
- Wiki: `docs/wiki/tools/manage_model_changes.mdx`

## 8. Unit Tests

Say:

"SemanticOps can create tests for the model so changes are safer. Tests can check metadata quality, relationships, key measures, RLS/OLS behavior, performance budgets, and regression snapshots."

Ask:

"Would you like to preview starter tests?"

Choices:

- Metadata, documentation, and presentation hygiene. Recommended after a model is connected.
- Relationship and referential-integrity checks.
- RLS/OLS validation.
- Core KPI and performance tests; I will provide the expected business behavior or budgets.
- Skip tests for now.

Allowed tools after approval:

- `manage_tests` packs_list and packs_apply with dry_run first.
- `manage_tests` validate/put/run only after approval.

Rules:

- Do not invent expected business values.
- Preview generated tests before saving them.

Links:

- `docs://unit-testing-guide`
- Wiki: `docs/wiki/tools/manage_tests.mdx`

## 9. Model Reports

Say:

"SemanticOps can produce documentation and review reports for your semantic model. These help with handoff, governance, audits, and onboarding teammates."

Ask:

"Would you like model documentation or a review report?"

Choices:

- Executive overview.
- Technical handoff.
- Governance and quality review.
- Relationship diagram or model documentation resource.
- Skip reports for now.

Allowed tools after approval:

- `list_model` report operation.
- Read relevant model documentation resources when available.

Rules:

- Avoid including sensitive values.
- Ask who the report is for before generating it.

## 10. Advanced Query Diagnostics

Say:

"When a DAX query is slow, Pro diagnostics can collect query plans, timings, traces, and performance details so we can identify the bottleneck."

Ask:

"How should I handle advanced query diagnostics?"

Choices:

- Use diagnostics when a query is slow.
- Ask before collecting query plans or timings. Recommended.
- Keep diagnostics off unless I request them.

Allowed tools after approval:

- `run_query` with `include_query_plan`, `include_timings`, or `trace`.
- Read performance resources.

Rules:

- Running queries or traces needs approval for the exact purpose and scope.

## 11. RLS Effective-User Testing

Say:

"If you use Power BI Service/XMLA and RLS, Pro can test as a specific effective user to confirm what that person would see."

Ask:

"Do you need to test RLS as specific users?"

Choices:

- Yes, for Service/XMLA datasets.
- Not now, but remind me if roles are found.
- No, this model does not use RLS.

Allowed tools after approval:

- `manage_model_connection` effective user setup.
- `run_query` validation under the approved identity.

Rules:

- Never invent a user identity.
- Do not set impersonation without explicit approval.

Links:

- `docs://security-roles-guide`

## 12. Prompt And Completion Helpers

Say:

"Some Pro features are helpers rather than setup switches: better completions and built-in prompts for DAX best practices, naming conventions, and security guidance. Guide documents are available to all tiers."

Ask:

"Would you like me to use those helpers automatically when they fit?"

Choices:

- Use Pro helpers automatically when available. Recommended.
- Ask before using Pro-only prompts.
- Keep responses short unless I ask for depth.

Allowed tools after approval:

- None by default. This is a behavior preference, not a persistent setting unless the user asks to save it.

## 13. Enterprise Posture

Ask only when Enterprise is active or the user mentions audit, compliance, or central governance.

Say:

"Enterprise adds audit evidence and centrally managed policy bundles. These are usually admin-controlled, so I will review posture rather than casually change them."

Ask:

"Do you need audit or admin-policy setup reviewed?"

Choices:

- Check audit status.
- Summarize or export audit evidence.
- Check if admin policy bundle is active.
- Prepare an admin handoff note.
- Skip Enterprise setup.

Allowed tools after approval:

- `manage_audit` for audit status/evidence.
- `manage_policy` status for admin bundle posture.

Rules:

- Do not try to override a centrally managed policy bundle.
- Do not expose sensitive audit details unnecessarily.

Links:

- `docs://audit-logging-guide`
- `docs://policy-guide`

## Final Approval Batch

Before applying anything, summarize:

- License action, if any.
- Connection/model selection, if any.
- Preference settings to set.
- Masking force/exclude lists.
- Guardrail packs to preview or apply.
- Test packs to preview or apply.
- Checkpoints or dependency checks to run.
- Reports or diagnostics to generate.
- RLS effective user, if any.
- Audit/admin-bundle actions, if any.
- Items intentionally not changed.

Do not proceed until the user approves that exact batch.

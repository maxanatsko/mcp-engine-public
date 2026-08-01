# Feature Setup Guide

Use this guide as the source checklist for the onboarding concierge. It maps the SemanticOps MCP Pro and Enterprise features to plain-language setup questions. Ask about each feature area when it needs a user choice or workflow preference. Mention passive helpers without pretending they are settings.

## Free Baseline

Plain-language explanation:

SemanticOps MCP works on Free out of the box. Free is enough for core semantic-model work when the server mode allows it: connect to models, browse metadata, query data, refresh, and edit model objects.

Ask:

"Do you want to continue with Free for now, or do you have a Pro or Enterprise key handy?"

Choices:

- Continue Free for now: keep setup to core model work and explain Pro-only safety features as optional upgrades.
- Activate Pro or Enterprise: guide the user through activation without echoing the key.
- Check current tier first: call `manage_license` status and explain what is already enabled.

Recommended default:

Check current tier first.

Tool/docs surface:

- `manage_license`
- Wiki: https://semanticops.dev/wiki/tools/manage_license
- Pricing: `/pricing`

Approval requirement:

License activation requires explicit approval and key-handling warning. Never repeat the key back.

## Server Mode

Plain-language explanation:

Mode controls what the assistant is allowed to do. Full can read, query, refresh, and edit. Read-only can inspect and query but cannot change the model. Browse-only can safely inspect structure but cannot run queries or make edits.

Ask:

"Would you like SemanticOps MCP to run in full, read-only, or browse-only mode?"

Choices:

- Full: best for active development where the assistant may make approved edits.
- Read-only: best for shared or production models where you want analysis but no changes.
- Browse-only: best for regulated or first-look environments where query results should stay hidden.
- Not sure: check the current mode and explain what it allows.

Recommended default:

Read-only for shared models; full for local development.

Tool/docs surface:

- `manage_model_connection` current state
- Wiki: https://semanticops.dev/wiki/concepts/modes-and-restrictions

Approval requirement:

Do not claim the assistant can change server mode at runtime unless the user's deployment actually exposes that setup path. Provide setup docs instead.

## PII Masking

Plain-language explanation:

PII masking tries to hide personal or sensitive identifiers in returned results, such as emails, phone numbers, payment-card-like values, birth dates, or known ID patterns. It reduces accidental exposure in chat, tickets, and screenshots.

Ask:

"Would you like to turn on PII masking?"

Choices:

- Turn on PII masking: good for customer, employee, contact, or account data.
- Keep it off: acceptable for demo or public data.
- Classify specific fields: add `force` or `exclude` annotations to reviewed tables or columns.
- Decide after model review: inspect metadata first, then recommend annotations.

Recommended default:

Turn on PII masking when the model may contain people, customer, or account data.

Tool/docs surface:

- `manage_preferences`
- Setting ID: `pii_masking_enabled`
- `manage_schema` annotations: `McpEngine_PiiMasking=force|exclude`
- `pii-masking-guide.md (from the mcp-engine-security-governance skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`
- Wiki: https://semanticops.dev/wiki/tools/manage_preferences-settings

Approval requirement:

Changing masking settings or model annotations requires explicit approval. Repeat annotation targets without exposing sensitive sample values.

## Numeric Masking

Plain-language explanation:

Numeric masking hides sensitive numbers in returned results. Use it when revenue, margin, payroll, forecasts, medical amounts, identifiers, or other numeric values should not appear directly in chat.

Ask:

"Would you like numeric masking too?"

Choices:

- Turn on numeric masking: good for finance, payroll, forecasts, and confidential metrics.
- Keep numeric values visible: useful when exact numbers are needed for analysis.
- Classify specific fields: add `force` or `exclude` annotations to reviewed tables or columns.
- Exclude safe reference tables: annotate reviewed calendars or harmless lookup tables.

Recommended default:

Turn on numeric masking for finance or regulated models; otherwise ask first.

Tool/docs surface:

- `manage_preferences`
- Setting ID: `numeric_masking_enabled`
- `manage_schema` annotations: `McpEngine_NumericMasking=force|exclude`
- `pii-masking-guide.md (from the mcp-engine-security-governance skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`
- Wiki: https://semanticops.dev/wiki/tools/manage_preferences-settings

Approval requirement:

Changing masking settings requires explicit approval. Explain that masking is a sharing safety feature, not an access-control boundary.

## Preferences And Memory

Plain-language explanation:

Preferences are working agreements the assistant remembers for future sessions. They can store naming conventions, glossary terms, default behavior, row limits, formatting choices, and safety habits. Pro adds workspace and model-specific preferences.

Ask:

"Are there conventions or preferences you usually want the assistant to follow?"

Choices:

- Naming conventions: measures, tables, columns, display folders, descriptions.
- Business glossary: abbreviations and terms like GM, COGS, ARR, churn.
- Output defaults: row limits, preview sizes, summarize instead of dumping tables.
- Model or workspace-specific memory: use when conventions differ by dataset or workspace.
- Skip for now: continue without saved preferences.

Recommended default:

Ask for naming conventions and row-limit defaults. Use model-specific preferences when Pro is active and a model is connected.

Tool/docs surface:

- `manage_preferences`
- Pro feature: `preferences_model_scope`
- Wiki: https://semanticops.dev/wiki/tools/manage_preferences, https://semanticops.dev/wiki/tools/manage_preferences-scopes, https://semanticops.dev/wiki/tools/manage_preferences-settings

Approval requirement:

Preference writes require explicit approval and must not store secrets, tokens, connection strings, row-level data extracts, or sensitive business values.

## Policy Guardrails

Plain-language explanation:

Guardrails are enforceable rules for the assistant. Preferences are guidance; policy is enforcement. Policies can block deletes, require confirmation for renames, protect security changes, or prevent broad refresh and partition operations.

Ask:

"Would you like to add guardrails for what the assistant is allowed to do?"

Choices:

- Safe authoring and destructive-change baseline: confirmation rails plus delete protection.
- Semantic quality: require descriptions, formats, and display folders.
- Masking and security hardening: protect masking changes, roles, OLS/RLS, and perspectives.
- Refresh and partition safety: block broad refresh or risky partition operations.
- Skip or describe a custom guardrail in plain English.

Recommended default:

Preview safe authoring baseline and destructive-change protection for full-mode users. Preview masking-settings approval when masking is enabled.

Tool/docs surface:

- `manage_policy`
- Pro feature: `policy_enforcement`
- `policy-guide.md (from the mcp-engine-security-governance skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`
- Wiki: https://semanticops.dev/wiki/tools/manage_policy

Approval requirement:

Always preview built-in packs before apply. Custom rules must be rendered/validated and approved before import.

## Model-Change Safety

Plain-language explanation:

Pro can keep model change history, checkpoints, undo/redo, rollback, and queued change plans. This gives users a recovery path before risky edits.

Ask:

"When you make model changes, do you want SemanticOps MCP to create checkpoints and keep change history?"

Choices:

- Always create a checkpoint before broad edits.
- Ask before checkpointing.
- Use checkpoints only for production or shared models.
- Skip change history for now.

Recommended default:

Ask before checkpointing, and always checkpoint before broad refactors.

Tool/docs surface:

- `manage_model_changes`
- Pro feature: `model_changes`
- `model-changes-guide.md (from the mcp-engine-testing-changes skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`
- Wiki: https://semanticops.dev/wiki/tools/manage_model_changes

Approval requirement:

Checkpoint creation, undo, redo, restore, and rollback require explicit approval. Explain the impact before restoring or rolling back.

## Dependency Analysis

Plain-language explanation:

Dependency analysis shows what may break before renames, deletes, relationship changes, measure refactors, or schema cleanup. It is a safety check before edits.

Ask:

"Should I check dependencies before risky edits like renames, deletes, or measure refactors?"

Choices:

- Always check before risky edits.
- Check only before deletes or renames.
- Ask me each time.
- Skip by default.

Recommended default:

Always check before deletes or renames.

Tool/docs surface:

- `manage_dependencies`
- Pro feature: `dependencies`
- `dependencies-guide.md (from the mcp-engine-testing-changes skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`
- Wiki: https://semanticops.dev/wiki/tools/manage_dependencies

Approval requirement:

Dependency reads are non-mutating, but still explain when the check may inspect expressions or metadata.

## Unit Tests

Plain-language explanation:

Unit tests help catch regressions in measures, DAX queries, security behavior, metadata quality, referential integrity, and performance. Built-in packs can create starter tests.

Ask:

"Would you like to set up model tests?"

Choices:

- Metadata, documentation, and presentation hygiene.
- Relationship/integrity checks: useful before schema changes.
- RLS/OLS validation: useful when security roles matter.
- KPI and performance tests: user provides expected business behavior or budgets.
- Skip for now.

Recommended default:

Preview metadata-quality first after a model is connected.

Tool/docs surface:

- `manage_tests`
- Pro feature: `unit_tests`
- `unit-testing-guide.md (from the mcp-engine-testing-changes skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`
- Wiki: https://semanticops.dev/wiki/tools/manage_tests

Approval requirement:

Always preview generated tests before saving. Do not invent expected business values without user input or a trusted baseline.

## Model Reporting

Plain-language explanation:

Model reporting generates documentation for a semantic model: structure, measures, relationships, metadata, and quality signals. It helps handoff, audits, review, and onboarding teammates.

Ask:

"Would you like SemanticOps MCP to generate model documentation or a review report?"

Choices:

- Executive overview: concise report for owners and stakeholders.
- Technical handoff: detailed model structure and measures.
- Governance review: focus on descriptions, roles, relationships, and quality gaps.
- Skip reporting for now.

Recommended default:

Offer a concise overview after the model is connected.

Tool/docs surface:

- `list_model` report operation
- Pro feature: `model_reporting`
- Dynamic resources: `pbi://docs/model`, `pbi://diagram/relationships`
- Wiki: https://semanticops.dev/wiki/tools/manage_model_properties, https://semanticops.dev/wiki/workflows/explore-and-understand

Approval requirement:

Report generation requires explicit approval if it may include sensitive model details. Avoid data values unless specifically requested and safe.

## Advanced Query Diagnostics

Plain-language explanation:

Advanced diagnostics help understand why a DAX query is slow or expensive. Pro can include query plans, timings, traces, and VertiPaq/performance references.

Ask:

"Do you want advanced query diagnostics available when we troubleshoot performance?"

Choices:

- Use diagnostics when a query is slow.
- Ask before collecting query plans or timings.
- Keep diagnostics off unless I request them.

Recommended default:

Ask before collecting query plans or timings.

Tool/docs surface:

- `run_query` parameters: `include_query_plan`, `include_timings`, `trace`
- Pro feature: `advanced_tool_use`
- Related resources: `query-performance-guide.md (from the mcp-engine-dax-performance skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`, `dax-query-plan-reference.md (from the mcp-engine-dax-performance skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`, `vertipaq-optimization-guide.md (from the mcp-engine-dax-performance skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`

Approval requirement:

Running queries or collecting diagnostics requires approval for the exact query purpose and scope.

## RLS Effective User Testing

Plain-language explanation:

Effective-user testing lets Service/XMLA users test what a specific user would see under RLS. This is useful for validating roles and security behavior.

Ask:

"Do you need to test security as specific users?"

Choices:

- Yes, for Service/XMLA datasets.
- Not now, but remind me when we inspect roles.
- No, this model does not use RLS.

Recommended default:

Ask again if roles are discovered in the connected model.

Tool/docs surface:

- `manage_model_connection` effective user support
- `run_query` under impersonation
- Pro feature: `rls_effective_user`
- `security-roles-guide.md (from the mcp-engine-security-governance skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`

Approval requirement:

Do not set impersonation without explicit user identity and approval. Never invent identities.

## Pro Prompts

Plain-language explanation:

Some Pro features are helpers rather than settings, including built-in prompts. Guide documents are available to all tiers.

Ask:

"Would you like me to use Pro helper prompts when they fit your task?"

Choices:

- Use them automatically when relevant.
- Ask before using Pro-only prompts.
- Keep explanations short unless I ask for depth.

Recommended default:

Use them automatically when relevant and available.

Tool/docs surface:

- Pro feature: built-in prompts
- Prompts: `dax_best_practices`, `naming_conventions`, `security_guidelines`

Approval requirement:

No persistent setup by default. Mention license gating if unavailable.

## Enterprise Audit Logging

Plain-language explanation:

Audit logging records tool activity for compliance and investigation. It is Enterprise-only and often controlled by an admin.

Ask only when Enterprise is active or the user mentions audit/compliance:

"Do you need audit evidence or an audit posture review?"

Choices:

- Review current audit status.
- Export or summarize audit evidence.
- Ask an admin to configure audit first.
- Skip audit setup.

Recommended default:

Review status only; do not change audit posture casually.

Tool/docs surface:

- `manage_audit`
- Enterprise feature: `audit_logging`
- `audit-logging-guide.md (from the mcp-engine-security-governance skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`
- Wiki: https://semanticops.dev/wiki/tools/manage_audit

Approval requirement:

Audit operations require explicit approval and Enterprise availability. Do not expose sensitive audit details unnecessarily.

## Enterprise Admin Policy Bundle

Plain-language explanation:

Admin policy bundles are centrally managed guardrails. When active, local policy edits may be locked because the organization controls the rules.

Ask only when Enterprise is active, bundle mode is detected, or the user is an admin:

"Is this a centrally governed setup where policy should come from an admin bundle?"

Choices:

- Check whether a bundle is active.
- Explain what local policy edits are locked.
- Prepare a message for the admin.
- Skip admin-bundle discussion.

Recommended default:

Check status and explain, not change.

Tool/docs surface:

- `manage_policy` status
- Enterprise feature: `admin_policy_bundle`
- `policy-guide.md (from the mcp-engine-security-governance skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`
- Wiki: https://semanticops.dev/wiki/governance/org-deployment

Approval requirement:

Do not attempt to override centrally managed policy. Explain the approved path.

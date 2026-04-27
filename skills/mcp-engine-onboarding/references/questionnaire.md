# Onboarding Questionnaire

Use the shortest interview that can produce a useful setup plan. Start with the core questions, then ask follow-ups only when the answer changes preferences, policies, tests, or workflow guidance.

## Core Questions

Ask for:

- Primary persona: analyst, Power BI developer, consultant, admin/team lead, or mixed.
- Target connection: Power BI Desktop, Power BI Service/XMLA, or both.
- Expected mode: full, read-only, browse-only, or unknown.
- Data sensitivity: none/low, customer or people data, finance data, regulated data, or sensitive numeric values.
- Authoring risk tolerance: permissive, confirm-heavy, or deny destructive operations.
- Preferred workflows: authoring, validation/testing, governance, refresh troubleshooting, AI readiness, or general discovery.
- Model scope: one PBIX, many client models, shared workspace, or organization/team standard.

## Grounding Checks

When MCP Engine tools are available, inspect current state before final recommendations:

- Use `manage_model_connection` with `operation: "get_current"` to determine whether a model is already selected.
- If no model is connected and the user wants model-specific setup, use `manage_model_connection` with `operation: "list"` and ask the user which model to select.
- Use `list_model` only after connection to ground recommendations in model metadata.
- Use `manage_preferences` with `action: "list"`, `resource: "rendered"`, and `scope: "global"` to inspect effective global preferences when appropriate.
- Use workspace or model preference scopes only when Pro tier and the current connection make those scopes available.

## Defaults When Answers Are Missing

Use these assumptions in the setup plan when the user does not know yet:

- Persona: mixed analyst/developer.
- Connection: start with Power BI Desktop, add Service/XMLA later if needed.
- Mode: full for solo local development, read-only for shared/team review, browse-only for discovery-only support.
- Sensitivity: assume customer/finance sensitivity until the user says otherwise.
- Risk tolerance: confirm-heavy.
- Workflows: connection bootstrap, metadata discovery, validation/testing, and safe-change workflow.
- Scope: one current model unless the user mentions team standards or client work.

## Approval Prompt

Before any write or query execution, summarize exactly what will be called:

- Tool name.
- Operation or action.
- Scope.
- Pack or setting IDs.
- Whether the call is a preview/dry run or a persistent change.
- What is intentionally not being changed.

Do not proceed with `manage_preferences`, `manage_policy` writes, `manage_tests` writes, model metadata writes, or `run_query` validation until the user approves that specific batch.

# Copilot Readiness Workflow

Use this workflow when assessing a Power BI semantic model for Copilot, Fabric data-agent, or natural-language Q&A readiness.

## Microsoft Prep data for AI boundaries

- Power BI Prep data for AI features are authored in Power BI Desktop and the Power BI service.
- The main Prep data for AI authoring surfaces are AI instructions, AI data schemas, and verified answers.
- AI instructions and AI data schemas save to model metadata/LSDL. Git or deployment pipeline edits can require a model refresh in the Power BI service before Copilot reflects the changes.
- Verified answers are human-approved visual responses with trigger phrases and optional filters. They require Power BI Q&A to be enabled and are saved on the semantic model.
- AI data schemas define focused fields for Copilot data-question capabilities. They are not a universal filter for every Copilot capability.
- A semantic model can be marked Approved for Copilot in the Power BI service, but that is a service setting and not a live MCP Engine model-write operation.

Sources:
- https://learn.microsoft.com/en-us/power-bi/create-reports/copilot-prepare-data-ai
- https://learn.microsoft.com/en-us/power-bi/create-reports/copilot-prepare-data-ai-instructions
- https://learn.microsoft.com/en-us/power-bi/create-reports/copilot-prepare-data-ai-data-schema
- https://learn.microsoft.com/en-us/power-bi/create-reports/copilot-prepare-data-ai-verified-answers
- https://learn.microsoft.com/en-us/power-bi/developer/projects/projects-dataset

## Assessment sequence

1. Establish context.
   - Identify the target audience, domain, primary decisions, certified metrics, and expected end-user questions.
   - If the user has no examples, generate 8 to 12 representative business questions from model metadata and ask them to confirm before treating them as acceptance criteria.

2. Inventory the model.
   - Use `list_model` to inspect tables, fields, measures, hierarchies, relationships, roles, model properties, descriptions, display folders, and hidden state.
   - Use `manage_model_properties` with `operation="get"` for model-level description, culture, compatibility level, annotations, and implicit-measure setting.
   - Use `manage_dependencies` when duplicate or overlapping metrics need impact analysis before rename, hide, or consolidation recommendations.

3. Audit natural-language ambiguity.
   - Flag measures or fields with unclear names, abbreviations, overloaded business terms, duplicate metrics, technical prefixes, or inconsistent display folders.
   - Compare measure expressions and dependencies when similarly named metrics might answer different questions.
   - Identify missing descriptions on high-value tables, measures, and dimensions.
   - Flag visible helper keys, bridge tables, internal sort columns, technical date columns, inactive-date ambiguity, and low-value fields that could distract Copilot.

4. Audit date and grain defaults.
   - Identify the default date table, fiscal calendar behavior, business grain, and common period comparisons.
   - Flag questions where Copilot could choose order date, ship date, invoice date, fiscal date, or calendar date incorrectly.
   - Recommend explicit AI instructions for preferred date role, fiscal year start, default period, comparison period, and default grain.

5. Validate critical answer scenarios.
   - Use `run_query` for aggregate validation only when needed to verify metric definitions, default grain, or filters.
   - Avoid previewing sensitive row-level values. If examples need values, use masked, synthetic, or aggregate examples.
   - Use `manage_tests` suggestions for stable business questions that should remain correct across model changes.

6. Produce artifacts.
   - Use the scorecard rubric for severity and readiness level.
   - Draft AI instructions that are explicit, grouped by theme, and short enough to avoid conflicting guidance.
   - Draft AI data schema recommendations as focused include/exclude lists by table, field, and measure.
   - Draft verified-answer candidates for common or high-risk questions that need curated visual answers.
   - List MCP-applicable model metadata fixes separately from manual Prep data for AI artifacts.

## Tool use patterns

Use these patterns as examples, not as required exact payloads.

```json
{ "operation": "list", "type": "tables", "include_hidden": true, "include_descriptions": true }
```

```json
{ "operation": "list", "type": "measures", "include_hidden": true, "include_expression": true, "include_descriptions": true }
```

```json
{ "operation": "get" }
```

```json
{ "operation": "used_by", "spec": { "target": { "type": "measure", "table": "Sales", "name": "Total Sales" }, "depth": 2 } }
```

## MCP-applicable versus draft-only examples

Can apply through existing MCP/model metadata when user asks and gates allow:
- Add or improve object descriptions.
- Rename ambiguous measures or fields.
- Hide helper columns or technical tables.
- Set model description, culture, annotations, or discourage implicit measures where supported.
- Create tests or baselines for critical aggregate scenarios.

Draft/export for Power BI Desktop, Power BI service, PBIP, Git, or manual review:
- AI instructions.
- AI data schema selections.
- Verified answers and trigger phrases.
- Approved for Copilot service setting.
- Copilot pane validation and HCAAT/diagnostics review.

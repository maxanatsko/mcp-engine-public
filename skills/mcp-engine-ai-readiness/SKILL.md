---
name: mcp-engine-ai-readiness
description: Assess Power BI semantic models for Copilot, Fabric data agent, and natural-language Q&A readiness. Use when reviewing whether a model has clear business terminology, unambiguous metrics, usable date defaults, focused field exposure, descriptions, AI instructions, AI data schema recommendations, verified-answer candidates, or natural-language validation tests.
---

# PBI AI Readiness

Use this skill to turn an existing Power BI semantic model into a Copilot-ready assessment and artifact pack. This is an authoring and review workflow, not a runtime MCP tool.

## Start Here

1. Confirm the user wants a readiness assessment, artifact drafts, or both.
2. Prefer metadata-level inspection before querying data values.
3. Use existing MCP Engine tools when available: `list_model`, `manage_dependencies`, `run_query`, `manage_tests`, and `manage_model_properties`.
4. Keep unsupported Prep data for AI actions as drafts for Power BI Desktop, Power BI service, PBIP, Git, or manual review.
5. Separate recommendations into:
   - can apply through MCP/model metadata now
   - draft/export for Prep data for AI UI or PBIP/Git workflow
   - validate manually in Copilot

## Workflow

- Read [copilot-readiness-workflow](references/copilot-readiness-workflow.md) for the assessment sequence, tool usage, privacy guardrails, and output order.
- Read [readiness-scorecard](references/readiness-scorecard.md) when producing severity, score, business impact, and remediation priority.
- Read [ai-artifact-templates](references/ai-artifact-templates.md) when drafting AI instructions, AI data schema recommendations, verified answers, or `manage_tests` candidates.
- Read [domain-examples](references/domain-examples.md) when the model is sales, finance, support, or operational and the user wants concrete starting examples.

## Guardrails

- Do not claim MCP Engine can directly configure all Power BI Prep data for AI settings over live TOM/XMLA.
- Treat AI instructions, AI data schemas, and verified answers as draft artifacts unless the user provides an explicit supported PBIP/Git path or asks for manual-application guidance.
- Do not expose sensitive values from data previews. Prefer names, descriptions, expressions, relationships, dependencies, and aggregate-only validation queries.
- Respect MCP Engine mode, policy, confirmation, license, and audit gates for any suggested or requested model change.
- Make nondeterminism explicit: readiness work can improve Copilot behavior, but it cannot guarantee identical answers for every prompt.

## Output Standard

Return a compact readiness pack unless the user asks for raw details:

1. Executive summary with readiness level.
2. Scorecard grouped by critical, high, medium, and low findings.
3. Recommended MCP-applicable model metadata fixes.
4. Draft AI instructions.
5. Draft AI data schema recommendation.
6. Verified-answer backlog.
7. Optional natural-language test suggestions.
8. Manual validation checklist for Power BI Desktop or service.


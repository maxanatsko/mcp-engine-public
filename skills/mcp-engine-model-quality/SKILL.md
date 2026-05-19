---
name: mcp-engine-model-quality
description: Assess Power BI semantic models for bad or questionable modeling practices and produce a source-backed quality scorecard. Use when reviewing model quality, semantic model assessment, semantic model best practices, star schema fit, relationships, DAX maintainability, VertiPaq/storage risk, metadata hygiene, governance signals, validation gaps, or when the user asks for a scorecard, recommendations, model audit, model quality review, bad practices review, best practices audit, or best practices assessment.
---

# PBI Model Quality

Use this skill to assess a connected Power BI semantic model and return a source-backed quality scorecard with prioritized recommendations. This is an assess-only workflow; do not apply model changes.

## Start Here

1. Confirm the current model context with SemanticOps MCP tools when needed.
2. Gather metadata before querying data.
3. Use `list_model`, `manage_dependencies`, `run_query`, `manage_tests`, and `manage_model_connection` where available.
4. Use `run_query` only for small aggregated validation, performance analysis, VertiPaq/storage diagnostics, or access tests.
5. Do not dump raw rows or sensitive values.
6. Cite bundled Microsoft Learn and SQLBI source links for material findings.

## Workflow

- Read [model-quality-assessment-workflow](references/model-quality-assessment-workflow.md) for the inspection sequence, SemanticOps MCP tool usage, safety rules, and final output order.
- Read [model-quality-scorecard](references/model-quality-scorecard.md) when scoring the model, assigning severity, formatting findings, and building the remediation backlog.
- Read [model-quality-rulebook](references/model-quality-rulebook.md) for source-backed bad/questionable practice checks and recommended remediation language.

## Assessment Areas

- Model shape and star-schema fit.
- Relationships and filter propagation risk.
- DAX and semantic layer maintainability.
- Storage and performance risk, including high-cardinality and unnecessary imported data.
- Metadata, naming, descriptions, display folders, and field exposure.
- Governance signals, including roles, sensitive-field exposure, and perspective-vs-security separation.
- Validation and test coverage.

## Guardrails

- Do not call write operations from authoring or governance tools during the assessment.
- Treat unavailable Pro diagnostics, browse-only mode, policy denials, or missing tool capabilities as scope limitations, not model defects.
- Keep source-backed guidance nuanced; do not turn "generally recommended" practices into absolute rules when the source allows exceptions.
- Mark inferred findings with lower confidence unless tool evidence confirms them.
- End with concrete remediation steps and validation suggestions, not broad advice.

## Output Standard

Return a compact quality assessment unless the user asks for raw detail:

1. Executive score and quality band.
2. Top 3 risks.
3. Category scorecard.
4. Findings grouped by critical, high, medium, and low severity.
5. Prioritized remediation backlog.
6. Validation/test recommendations.
7. Source notes.

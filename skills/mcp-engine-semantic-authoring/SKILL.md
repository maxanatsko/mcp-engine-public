---
name: mcp-engine-semantic-authoring
description: Use when creating or updating measures, KPIs, calculation groups, DAX UDFs, named expressions, Power Query parameters, or model properties, when deciding between a measure and a calculated column, or when cleaning up naming, display folders, and semantic style. For one-off DAX queries, use mcp-engine-query; for physical tables, columns, and relationships, use mcp-engine-schema-authoring; for consolidating or renaming measures with downstream consumers, use mcp-engine-refactoring.
---

# PBI Semantic Authoring

Semantic-layer work through `manage_semantic`. Read only the references needed for the current task.

## Decide the semantic path

1. Prefer a measure for reusable business logic. Choose a calculated column only when a persisted row-level attribute is required — creating one is a `manage_schema` `create_calc_column` operation covered by `mcp-engine-schema-authoring`.
2. Ground names first: `list_model` `{ "operation": "list", "spec": { "type": "measures" } }` (repeat with `"tables"`; `spec.type` takes one value per call), and check for an existing equivalent with `{ "operation": "search", "spec": { "query": "<name>", "mode": "name" } }` before creating a near-duplicate measure.
3. Keep semantic edits separate from table or relationship edits unless the task explicitly spans both.

## Branch by object type

- Measures and KPIs (`create_measure`, `update_measure`, `create_kpi`): [measure-authoring-guide](references/measure-authoring-guide.md). When logic placement is unclear: [modeling-best-practices-guide](references/modeling-best-practices-guide.md).
- Calculation groups (`create_calc_group`): [calc-groups-guide](references/calc-groups-guide.md). DAX UDFs (`create_udf`): [dax-udf-guide](references/dax-udf-guide.md).
- Named expressions (`create_named_expression`): [named-expressions-guide](references/named-expressions-guide.md). Power Query parameters (`create_pq_parameter`, RangeStart/RangeEnd): [power-query-parameters-guide](references/power-query-parameters-guide.md).
- Model properties (descriptions, culture, annotations, implicit-measure behavior): [model-properties-guide](references/model-properties-guide.md). Naming, folders, and style cleanup across many objects: [semantic-layer-style-guide](references/semantic-layer-style-guide.md).

## Guardrails

- Follow the model's existing naming and folder conventions unless the user requests a deliberate change.
- Validate each new or changed measure with a scoped `run_query` `{ "operation": "execute", "query": "<query evaluating the measure>" }` in a realistic filter context.
- Use `dry_run: true` and per-item identifiers for bulk semantic edits.

## Report results

After semantic work, report:

1. Each object created or changed, with its DAX or M expression.
2. Whether `dry_run` or bulk mode was used and the outcome.
3. The validation query run and its result.
4. Naming or folder decisions that follow (or deliberately break) model conventions.

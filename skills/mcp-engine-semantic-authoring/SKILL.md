---
name: mcp-engine-semantic-authoring
description: Author and refine Power BI semantic layer logic. Use when creating or updating measures, calculation groups, named expressions, DAX UDFs, Power Query parameters, model properties, or semantic naming and style patterns, and when deciding between measures and calculated columns.
---

# PBI Semantic Authoring

Use this skill for semantic-layer work that sits above physical schema shape. Read only the bundled references needed for the current semantic task.

## Decide the semantic path

1. Decide whether the user needs a measure, calculated column alternative, calculation group, shared expression, model property update, or style cleanup.
2. Prefer measures for reusable business logic unless persisted row-level attributes are required.
3. Keep semantic edits separate from table or relationship edits unless the task explicitly spans both.

## Branch by object type

### Measures

- Read [measure-authoring-guide](references/measure-authoring-guide.md) when creating, updating, formatting, or organizing measures.
- Use [modeling-best-practices-guide](references/modeling-best-practices-guide.md) when deciding whether logic belongs in a measure or in model structure.

### Calculation groups and DAX UDFs

- Read [calc-groups-guide](references/calc-groups-guide.md) for reusable transformation patterns such as time intelligence and formatting changes.
- Read [dax-udf-guide](references/dax-udf-guide.md) when the user wants reusable DAX function-style logic.

### Named expressions and Power Query parameters

- Read [named-expressions-guide](references/named-expressions-guide.md) for shared M expressions.
- Read [power-query-parameters-guide](references/power-query-parameters-guide.md) when the task involves RangeStart, RangeEnd, or other Power Query parameter management.

### Model properties and style

- Read [model-properties-guide](references/model-properties-guide.md) when updating descriptions, culture, compatibility, annotations, or implicit-measure behavior.
- Read [semantic-layer-style-guide](references/semantic-layer-style-guide.md) when the task is naming, folders, style consistency, or semantic cleanup across many objects.

## Guardrails

- Prefer measure-first reasoning before introducing calculated columns.
- Keep naming and foldering consistent with existing model conventions unless the user requests a deliberate change.
- Validate major semantic edits with targeted queries after authoring.
- Use the best-practices guide when the right home for logic is unclear.

## References

- [measure-authoring-guide](references/measure-authoring-guide.md)
- [calc-groups-guide](references/calc-groups-guide.md)
- [named-expressions-guide](references/named-expressions-guide.md)
- [dax-udf-guide](references/dax-udf-guide.md)
- [power-query-parameters-guide](references/power-query-parameters-guide.md)
- [model-properties-guide](references/model-properties-guide.md)
- [semantic-layer-style-guide](references/semantic-layer-style-guide.md)
- [modeling-best-practices-guide](references/modeling-best-practices-guide.md)

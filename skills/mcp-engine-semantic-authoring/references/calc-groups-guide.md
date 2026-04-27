# Calculation Groups Guide (`manage_semantic`)

This guide covers calculation group authoring patterns, time intelligence items, and bulk workflows.

## Related Tools

- `manage_semantic`: Create/update/delete calculation groups and items (operations: `create_calc_group`, `update_calc_group`, `delete_calc_group`)
- `list_model`: Inspect calc groups (`operation: "list"`, `spec: { type: "calculation_groups", include_details: true, include_expression: true }`)
- `list_model`: Find references to calc items (`operation: "search"`, `spec: { mode: "dax", query: "SELECTEDMEASURE(" }`)
- `run_query`: Validate results for representative measures (`operation: "execute"`)

## Compatibility Level

Calculation groups require compatibility level >= 1500. `manage_semantic` with `create_calc_group` supports:

- `allow_compatibility_upgrade=true` in spec to attempt an upgrade if needed (may be blocked by host).

## Create a Calculation Group

`items` is optional. Omit it to create an empty calculation group shell and add items later with `update_calc_group`.

```json
{
  "operation": "create_calc_group",
  "target": "Time Intelligence",
  "spec": {
    "column_name": "Time Calc",
    "precedence": 10,
    "items": [
      { "name": "Current", "expression": "SELECTEDMEASURE()" },
      { "name": "YOY", "expression": "CALCULATE(SELECTEDMEASURE(), SAMEPERIODLASTYEAR('Date'[Date]))" }
    ],
    "allow_compatibility_upgrade": true
  }
}
```

## Update Items (Upsert / Rename / Delete)

```json
{
  "operation": "update_calc_group",
  "target": "Time Intelligence",
  "spec": {
    "items_upsert": [
      {
        "name": "YTD",
        "expression": "CALCULATE(SELECTEDMEASURE(), DATESYTD('Date'[Date]))",
        "format_string_expression": "SELECTEDMEASUREFORMATSTRING()"
      }
    ],
    "items_delete": ["Current"],
    "reorder_by_ordinal": true
  }
}
```

To rename an item and update its expression in one call, include `new_name` on the `items_upsert` entry:

```json
{
  "operation": "update_calc_group",
  "target": "Time Intelligence",
  "spec": {
    "items_upsert": [
      {
        "name": "YTD",
        "new_name": "Year to Date",
        "expression": "CALCULATE(SELECTEDMEASURE(), DATESYTD('Date'[Date]))"
      }
    ]
  }
}
```

## Format String Expressions

Prefer `format_string_expression` when your calc item changes the meaning/units of a measure:

- Keep the base format with `SELECTEDMEASUREFORMATSTRING()`
- Override for percent change items, etc.

## Common Time Intelligence Patterns

- `SELECTEDMEASURE()` for pass-through
- `SAMEPERIODLASTYEAR('Date'[Date])` for YoY
- `DATESYTD('Date'[Date])` for YTD
- `DATEADD('Date'[Date], -1, YEAR)` for prior year shift

Ensure:

- A proper date table exists and is marked as a date table (`manage_schema` with `operation: "update_table"` and `spec: { mark_date_table: true }`).
- Date column used is a Date/DateTime column.

## Bulk Operations: `bulk_items` (Not `items`)

Because `items` is used for calculation items, bulk uses `bulk_items`:

```json
{
  "operation": "update_calc_group",
  "transaction": true,
  "dry_run": true,
  "bulk_items": [
    {
      "target": "Time Intelligence",
      "spec": {
        "items_upsert": [{ "name": "MTD", "expression": "CALCULATE(SELECTEDMEASURE(), DATESMTD('Date'[Date]))" }]
      }
    }
  ]
}
```

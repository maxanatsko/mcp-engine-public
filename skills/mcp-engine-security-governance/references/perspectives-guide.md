# Perspectives Guide (`manage_security`)

Perspectives provide curated views of a semantic model for different audiences (e.g., executive vs analyst).

## Related Tools

- `manage_security`: Create/update/delete perspectives (operations: `create_perspective`, `update_perspective`, `delete_perspective`)
- `list_model`: Inspect perspectives (`operation: "list"`, `spec: { type: "perspectives", include_details: true }`)
- `list_model`: Choose members to include (`operation: "list"`, `spec: { type: "tables|measures|columns|hierarchies" }`)

## Create a Perspective

Members are objects with:

- `type`: `Table|Column|Measure|Hierarchy`
- `table`: table name
- `name`: object name (required for non-Table members)

```json
{
  "operation": "create_perspective",
  "target": "Executive",
  "spec": {
    "description": "Executive view",
    "members": [
      { "type": "Table", "table": "Sales" },
      { "type": "Measure", "table": "Sales", "name": "Total Sales" },
      { "type": "Measure", "table": "Sales", "name": "Total Margin" }
    ]
  }
}
```

## Update a Perspective (Add / Remove)

```json
{
  "operation": "update_perspective",
  "target": "Executive",
  "spec": {
    "add": [{ "type": "Measure", "table": "Sales", "name": "YOY Sales" }],
    "remove": [{ "type": "Table", "table": "Staging" }]
  }
}
```

## Curation Best Practices

- Include only "public" measures; keep helpers hidden or excluded.
- Prefer curated hierarchies over raw columns when possible.
- Align perspectives with roles/audiences, but note: perspectives are not security boundaries.

## Bulk Perspective Operations

```json
{
  "operation": "delete_perspective",
  "transaction": false,
  "items": [
    { "target": "Deprecated View" },
    { "target": "Old Executive" }
  ]
}
```

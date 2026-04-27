# Power Query Parameters Guide (`manage_semantic`)

Power Query parameters are stored in the semantic model as **shared M expressions** (named expressions) annotated with a `meta [...]` record.

Example of a parameter expression:

```m
(0) meta [IsParameterQuery=true, Type="Any", IsParameterQueryRequired=true]
```

This guide covers the recommended, structured API for creating and updating parameters without hand-writing `meta [...]`.

## Related Tools

- `manage_semantic`: Create/update/read/delete Power Query parameters:
  - `create_pq_parameter`
  - `update_pq_parameter`
  - `read_pq_parameter`
  - `delete_pq_parameter`
- `manage_semantic`: Underlying storage is still a named expression (`create_named_expression`, etc.). Use these only if you need raw control.
- `list_model`: Inspect existing parameters/named expressions (`operation: "list"`, `spec: { type: "named_expressions", include_expression: true }`)

## Create a Parameter

```json
{
  "operation": "create_pq_parameter",
  "target": "Country",
  "spec": {
    "type": "Text",
    "required": true,
    "current_value": "US"
  }
}
```

### Parameter Types

Supported `spec.type` values:

- `Any` (default)
- `Text`
- `Number`
- `Logical`
- `Date`
- `DateTime`
- `Time`
- `Duration`

Notes:

- `Date` expects `current_value` as `"YYYY-MM-DD"`
- `DateTime` expects `current_value` as `"YYYY-MM-DDTHH:MM:SS"`
- `Time` expects `current_value` as `"HH:MM:SS"`
- `Duration` expects `current_value` as `"d.hh:mm:ss"`

## Suggested Values

Power Query parameters can provide dropdown suggestions via `AllowedValues` metadata.

### Any value (default)

Omit `suggested_values`, or set:

```json
{ "mode": "any" }
```

### List of values

```json
{
  "operation": "create_pq_parameter",
  "target": "Country",
  "spec": {
    "type": "Text",
    "current_value": "US",
    "suggested_values": { "mode": "list", "values": ["US", "CA", "MX"] }
  }
}
```

### Query-based suggested values

This mode references another shared query that returns a **list**.

```json
{
  "operation": "create_pq_parameter",
  "target": "Country",
  "spec": {
    "type": "Text",
    "current_value": "US",
    "suggested_values": { "mode": "query", "query": "AllowedCountries" }
  }
}
```

If your source is a table query, create a small helper list query in Power Query (example):

```m
AllowedCountries = List.Distinct(DimCountry[Country])
```

Then reference that list query in the parameter.

## Update a Parameter

Update always requires `current_value`.

If you only change `current_value` (and omit `type` / `required` / `suggested_values`), the server preserves the existing `meta [...]` block (including `AllowedValues`).

If you change `type` and/or `required` (and omit `suggested_values`), the server preserves the existing `AllowedValues` and other `meta [...]` fields when possible.

If you provide `suggested_values`, the server replaces `AllowedValues` to match your request.

```json
{
  "operation": "update_pq_parameter",
  "target": "Country",
  "spec": {
    "current_value": "CA"
  }
}
```

Partial updates:

- If you supply `suggested_values`, the server **replaces** `AllowedValues` to match your request.
- If you supply `type` and/or `required` (and omit `suggested_values`), the server **updates only** `Type` / `IsParameterQueryRequired` and preserves the rest of the existing `meta [...]` record (including `AllowedValues`).
  - Note: the server does **not** validate or coerce existing `AllowedValues` when you change `type`. If you change the type, consider also updating `suggested_values` (or set `suggested_values: { mode: "any" }`) to avoid type-incompatible suggestions in the UI.

## Read / Delete

Read:

```json
{ "operation": "read_pq_parameter", "target": "Country" }
```

Delete:

```json
{ "operation": "delete_pq_parameter", "target": "Country" }
```

`delete_pq_parameter` fails if the target expression does not look like a Power Query parameter (missing `IsParameterQuery=true`).

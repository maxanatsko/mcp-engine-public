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

Power Query parameters can provide dropdown suggestions via metadata. MCP writes `AllowedValues`; native Power BI Desktop parameters may use the equivalent `List` field.

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

Update is patch-style: provide `target` plus only the fields you want to change. `current_value` is optional.

If you only change `current_value` (and omit `type` / `required` / `suggested_values`), the server preserves the existing `meta [...]` block (including `AllowedValues` or native `List`).

If you change `type` and/or `required` without `current_value` (and omit `suggested_values`), the server preserves the existing value expression, existing suggestion metadata (`AllowedValues` or native `List`), and other `meta [...]` fields when possible.

If you provide `suggested_values`, the server replaces suggestion metadata to match your request, normalizing native `List` to `AllowedValues`, and preserves the existing value expression unless you also provide `current_value`.

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

- If you supply `suggested_values`, the server **replaces** suggestion metadata to match your request. Native Power BI Desktop `List` metadata is removed and normalized to `AllowedValues` when finite or query-based suggestions remain.
- If you supply `type` and/or `required` without `current_value` (and omit `suggested_values`), the server **updates only** `Type` / `IsParameterQueryRequired` and preserves the existing value expression plus the rest of the existing `meta [...]` record (including `AllowedValues` or native `List`).
  - Note: the server does **not** validate or coerce existing `AllowedValues` / `List` values when you change `type`. If you change the type, consider also updating `suggested_values` (or set `suggested_values: { mode: "any" }`) to avoid type-incompatible suggestions in the UI.
- If you provide only `new_name`, `description`, `query_group`, or `clear_query_group`, the value expression and parameter metadata are preserved.
- A no-op update, including only `allow_compatibility_upgrade`, is rejected.

## Read / Delete

Read returns `parameter_meta.suggested_values_mode = "list"` for both MCP-authored `AllowedValues={...}` and native Power BI Desktop `List={...}` parameters. Query-backed `AllowedValues=Record.Field(#shared, "...")` returns `"query"` plus `suggested_values_query`.

Read:

```json
{ "operation": "read_pq_parameter", "target": "Country" }
```

Delete:

```json
{ "operation": "delete_pq_parameter", "target": "Country" }
```

`delete_pq_parameter` fails if the target expression does not look like a Power Query parameter (missing `IsParameterQuery=true`).

On Desktop, creating/updating/deleting Power Query parameters updates shared M expressions and returns `desktop_sync_pending=true`. On June 2026+ Power BI Desktop, M/TOM changes are reflected immediately and silently with no Desktop action needed. Older Desktop versions may still show an external-changes banner; only if prompted, click `Discard` to accept MCP's external changes and do not click `Apply`. You may continue additional M changes, but in all Desktop versions call `manage_model_connection` with `operation="reload"` before continuing non-M operations (non-M write tools are blocked until reload verification succeeds). If multiple M changes were made before reload, responses include `desktop_sync_items[]`.

# Model Properties Guide (`manage_model_properties`, `list_model`)

This guide covers common global model settings and safe update patterns.

## Related Tools

- `manage_model_properties`: Get/update global properties
- `list_model`: List properties in a discovery-friendly shape (`operation: "list"`, `spec: { type: "model_properties" }`)
- `manage_schema` / `manage_semantic`: Some features require specific compatibility levels

## Read Model Properties

Using `manage_model_properties`:

```json
{ "operation": "get" }
```

Using `list_model`:

```json
{ "operation": "list", "spec": { "type": "model_properties" } }
```

## Update Common Properties

### Description

```json
{ "operation": "update", "description": "Contoso Sales Model" }
```

Set to empty string to clear:

```json
{ "operation": "update", "description": "" }
```

### Culture

```json
{ "operation": "update", "culture": "en-US" }
```

### Discourage Implicit Measures

```json
{ "operation": "update", "discourage_implicit_measures": true }
```

## Annotations

Upsert:

```json
{
  "operation": "update",
  "annotations_upsert": [{ "name": "owner", "value": "finance" }]
}
```

Delete:

```json
{
  "operation": "update",
  "annotations_delete": ["owner"]
}
```

## Compatibility Level Upgrades (Irreversible)

You can request a compatibility level increase:

```json
{
  "operation": "update",
  "compatibility_level": 1702,
  "allow_compatibility_upgrade": true
}
```

Notes:

- Only upgrades are attempted; downgrades are not supported.
- Upgrades may trigger a confirmation prompt (elicitation) and may be blocked by the host.
- If the client cannot show confirmation prompts, send `"confirm": true` with the request or the server will reject the upgrade.
- Keep a backup of the PBIX before upgrades.

# Security Roles Guide (`manage_security`)

This guide covers row-level security (RLS) and object-level security (OLS) patterns using `manage_security`.

## Related Tools

- `manage_security`: Create/delete roles; set filters; set permissions (operations: `create_role`, `delete_role`, `set_role_filters`, `set_role_permissions`)
- `list_model`: Inspect roles (`operation: "list"`, `spec: { type: "roles", include_details: true }`)
- `list_model`: Confirm object names before applying filters/permissions (`operation: "list"`, `spec: { type: "tables|columns" }`)

## Create a Role

```json
{
  "operation": "create_role",
  "target": "Sales Team"
}
```

## Security DAX Functions

Use these functions in RLS filter expressions to create dynamic, user-aware security:

| Function | Description | Use Case |
|----------|-------------|----------|
| `USERPRINCIPALNAME()` | Returns Azure AD / Entra ID email (e.g., `user@company.com`) | **Preferred** for cloud deployments |
| `USERNAME()` | Returns domain\username format (e.g., `DOMAIN\user`) | Legacy / on-premises |
| `CUSTOMDATA()` | Returns custom string from connection string | App-specific context |
| `HASONEFILTER(column)` | True if exactly one filter on column | Conditional security logic |
| `HASONEVALUE(column)` | True if single value in filter context | Dynamic RLS patterns |
| `SELECTEDVALUE(column)` | Returns single selected value or BLANK | User-specific lookups |

**Example: Dynamic RLS with user lookup table**

```dax
// Filter expression for Sales table
VAR CurrentUser = USERPRINCIPALNAME()
VAR UserRegion = LOOKUPVALUE('SecurityTable'[Region], 'SecurityTable'[Email], CurrentUser)
RETURN 'Sales'[Region] = UserRegion
```

**Best Practice**: Prefer `USERPRINCIPALNAME()` over `USERNAME()` for Microsoft Entra ID (Azure AD) environments.

## Row-Level Security (RLS): `set_role_filters`

`filters` items in spec are:

- `table`: table name
- `expression`: optional DAX filter expression for that table

Example:

```json
{
  "operation": "set_role_filters",
  "target": "Sales Team",
  "spec": {
    "filters": [
      { "table": "Sales", "expression": "'Sales'[Region] = \"West\"" },
      { "table": "Customer", "expression": "NOT ISBLANK('Customer'[CustomerId])" }
    ]
  }
}
```

Best practices:

- Keep filters simple and sargable when possible (avoid heavy iterators).
- Prefer filtering dimensions over facts when it yields the same effect.
- Validate that relationships propagate filters as intended.
- If role updates start failing after a Power BI Desktop refresh, reload model metadata first with `manage_model_connection` `operation: "reload"`.
- When the engine returns underlying TOM or SSAS diagnostics for a role update, `manage_security` includes them in the response detail.

## Object-Level Security (OLS): `set_role_permissions`

`permissions` items in spec are:

- `object_type`: `table` or `column`
- `table`: table name
- `column`: required for `object_type="column"`
- `permission`: `read`, `none`, or `default`
  - `read` — allow read access (explicit grant)
  - `none` — explicitly block access (deny)
  - `default` — reset to inherited/unset (removes the OLS restriction)

Example: hide salary column:

```json
{
  "operation": "set_role_permissions",
  "target": "Sales Team",
  "spec": {
    "permissions": [
      { "object_type": "column", "table": "Employees", "column": "Salary", "permission": "none" }
    ]
  }
}
```

## Least Privilege Workflow

1. Create role.
2. Add minimal RLS filters.
3. Add OLS restrictions for sensitive tables/columns.
4. Re-check that required measures still compute and visuals do not error.

## Bulk Role Operations

```json
{
  "operation": "create_role",
  "transaction": true,
  "items": [
    { "target": "Sales Team" },
    { "target": "Finance" }
  ]
}
```

Bulk rules:

- Top-level `transaction`, `dry_run`, `include_items`, and `include_details` apply to the whole request.
- Put `target` and `spec` inside each `items[]` entry.
- Do not rely on top-level `target` being copied into bulk items.

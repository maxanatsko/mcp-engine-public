# Relationships Guide (`manage_schema`)

This guide covers relationship modeling best practices and how to apply them with `manage_schema`.

## Related Tools

- `manage_schema`: Create/update/delete relationships (operations: `create_relationship`, `update_relationship`, `delete_relationship`)
- `list_model`: Inspect existing relationships (`operation: "list"`, `spec: { type: "relationships", active_only: true|false }`)
- `list_model`: Confirm key column data types (`operation: "list"`, `spec: { type: "columns" }`)
- `run_query`: Validate filter propagation scenarios (`operation: "execute"`)

## Core Concepts

- The **many side** should be the fact table (foreign keys): `from_table` / `from_column` in spec.
- The **one side** should be the dimension table (primary keys): `to_table` / `to_column` in spec.
- Canonical cardinality fields are `from_cardinality` and `to_cardinality`, each with values `Many` or `One`.
- Default cardinalities are typically `Many` (from) to `One` (to).
- `cardinality` is accepted as a create-only shorthand alias for `ManyToOne`, `OneToMany`, `OneToOne`, or `ManyToMany`.

## Create a Relationship

```json
{
  "operation": "create_relationship",
  "target": "Sales_Customer",
  "spec": {
    "from_table": "Sales",
    "from_column": "CustomerId",
    "to_table": "Customer",
    "to_column": "CustomerId",
    "from_cardinality": "Many",
    "to_cardinality": "One",
    "cross_filter_direction": "OneDirection",
    "is_active": true,
    "rely_on_referential_integrity": false
  }
}
```

Shorthand alias form:

```json
{
  "operation": "create_relationship",
  "target": "Sales_Customer",
  "spec": {
    "from_table": "Sales",
    "from_column": "CustomerId",
    "to_table": "Customer",
    "to_column": "CustomerId",
    "cardinality": "ManyToOne"
  }
}
```

After creating a relationship:

- If you immediately validate a measure that uses `RELATED()`, `USERELATIONSHIP`, or new filter propagation, a `Calculate` refresh may still be required.
- When in doubt, run a `Calculate` refresh on the affected table or model before treating the validation error as a DAX bug.

## Cross-filter Direction

Common values:

- `OneDirection` (recommended default): filters flow from dimension to fact
- `BothDirections`: use sparingly; can cause ambiguous filter paths
- `Automatic`: engine decides (often avoid for determinism)

## Active vs Inactive Relationships

- Use inactive relationships for alternate date keys (e.g., OrderDate vs ShipDate).
- In DAX, activate an inactive relationship with `USERELATIONSHIP`.

If multiple relationships match the same endpoints, use:

- `match_is_active=true|false` in spec on update/delete to target the intended one.

## Weak Relationships (Limited Relationships)

A **weak relationship** occurs when the "one" side column does not contain unique values or when the relationship is created between calculated columns or columns with incompatible data types.

**Symptoms of weak relationships:**
- Unexpected totals or duplicated values in reports
- Performance degradation due to expanded table scans
- Warning icon on the relationship in Power BI Desktop

**Common causes:**
- Many-to-many relationships without a proper bridge table
- Joining on calculated columns
- Data quality issues (duplicates in dimension key)

**Fixes:**
- Ensure the dimension key column contains unique values
- Use surrogate integer keys instead of natural keys
- Create a proper bridge table for many-to-many scenarios
- Use `TREATAS` for virtual relationships when physical relationships aren't possible

**Example: Using TREATAS instead of weak relationship**

```dax
// Virtual relationship via TREATAS
CALCULATE(
    [Total Sales],
    TREATAS(VALUES('Budget'[ProductKey]), 'Sales'[ProductKey])
)
```

## Rename / Update a Relationship

```json
{
  "operation": "update_relationship",
  "target": "Sales_Customer",
  "spec": {
    "new_name": "Sales_to_Customer",
    "cross_filter_direction": "OneDirection",
    "is_active": true
  }
}
```

## Delete a Relationship

```json
{
  "operation": "delete_relationship",
  "target": "Sales_to_Customer"
}
```

## Ambiguity Checks (Recommended)

Before enabling bidirectional filtering or multiple active paths:

1. `list_model` with `operation: "list"` and `spec: { type: "relationships" }` to map paths between major tables.
2. Prefer star schemas: facts in the center, dimensions around.
3. Avoid multiple active relationships between the same two tables unless necessary.

## Recommended Ordering

For newly authored models:

1. Create or update tables.
2. Confirm user-facing columns are visible. For calculated tables, a successful save alone does not guarantee that user-facing columns materialized; `process=true` must complete successfully.
3. Create relationships.
4. Run a `Calculate` refresh if validation-sensitive measures are next.
5. Create or validate measures that rely on the new relationships.

## Bulk Relationship Changes

```json
{
  "operation": "create_relationship",
  "dry_run": true,
  "items": [
    { "target": "Sales_Product", "spec": { "from_table": "Sales", "from_column": "ProductId", "to_table": "Product", "to_column": "ProductId" } },
    { "target": "Sales_Store", "spec": { "from_table": "Sales", "from_column": "StoreId", "to_table": "Store", "to_column": "StoreId" } }
  ]
}
```

# Hierarchies Guide (`manage_schema`)

This guide covers hierarchy creation, management, and best practices for drill-down navigation in Power BI semantic models.

## Related Tools

- `manage_schema`: Create, update, or delete hierarchies (operations: `create_hierarchy`, `update_hierarchy`, `delete_hierarchy`)
- `list_model`: List existing hierarchies (`operation: "list"`, `spec: { type: "hierarchies", include_levels: true, table: "..." }`)
- `list_model`: Identify columns to include in hierarchies (`operation: "list"`, `spec: { type: "columns" }`)
- `manage_schema`: Set `sort_by` for hierarchy-related columns (`operation: "update_column_properties"`)

## What Are Hierarchies?

Hierarchies define drill-down paths through related columns, enabling users to navigate from summary to detail levels in visuals. Common examples:

- **Date hierarchy**: Year > Quarter > Month > Day
- **Geography hierarchy**: Country > State > City
- **Product hierarchy**: Category > Subcategory > Product
- **Organization hierarchy**: Division > Department > Team

## Create a Hierarchy

```json
{
  "operation": "create_hierarchy",
  "table": "DimDate",
  "target": "Date Hierarchy",
  "spec": {
    "levels": [
      { "name": "Year", "column": "Year" },
      { "name": "Quarter", "column": "Quarter" },
      { "name": "Month", "column": "MonthName" },
      { "name": "Day", "column": "Day" }
    ]
  }
}
```

**Level properties:**
- `name`: Display name for the level (can differ from column name)
- `column`: Source column in the table

## Update a Hierarchy

### Rename a hierarchy

```json
{
  "operation": "update_hierarchy",
  "table": "DimDate",
  "target": "Date Hierarchy",
  "spec": { "new_name": "Calendar Hierarchy" }
}
```

### Add levels

```json
{
  "operation": "update_hierarchy",
  "table": "DimDate",
  "target": "Date Hierarchy",
  "spec": {
    "levels_upsert": [
      { "name": "Week", "column": "WeekOfYear", "ordinal": 3 }
    ]
  }
}
```

**Note**: `ordinal` specifies position (0-based). If omitted, level is added at the end.

### Remove levels

```json
{
  "operation": "update_hierarchy",
  "table": "DimDate",
  "target": "Date Hierarchy",
  "spec": {
    "levels_delete": ["Week"]
  }
}
```

### Reorder levels

```json
{
  "operation": "update_hierarchy",
  "table": "DimDate",
  "target": "Date Hierarchy",
  "spec": {
    "levels_upsert": [
      { "name": "Year", "column": "Year", "ordinal": 0 },
      { "name": "Quarter", "column": "Quarter", "ordinal": 1 },
      { "name": "Month", "column": "MonthName", "ordinal": 2 },
      { "name": "Week", "column": "WeekOfYear", "ordinal": 3 },
      { "name": "Day", "column": "Day", "ordinal": 4 }
    ],
    "reorder_by_ordinal": true
  }
}
```

### Update hierarchy properties

```json
{
  "operation": "update_hierarchy",
  "table": "DimDate",
  "target": "Date Hierarchy",
  "spec": {
    "description": "Standard calendar drill-down path",
    "display_folder": "Time",
    "is_hidden": false
  }
}
```

## Delete a Hierarchy

```json
{
  "operation": "delete_hierarchy",
  "table": "DimDate",
  "target": "Date Hierarchy"
}
```

## Best Practices

### Naming Conventions

- Use descriptive names: `Date Hierarchy`, `Geography`, `Product Category`
- Avoid generic names like `Hierarchy1` or `H1`
- Level names should be business-friendly: `Year`, `Quarter`, `Month` (not `Col1`, `Col2`)

### Level Design

- Order levels from coarsest to finest grain (Year before Month before Day)
- Ensure each level has a many-to-one relationship with the level above
- Use columns with appropriate sort order (numeric month number sorts correctly, month name may not)

### Sort Order Considerations

If a text column needs custom sorting (e.g., month names):

1. Ensure a numeric sort column exists (e.g., `MonthNumber`)
2. Set sort-by using `manage_schema`:

```json
{
  "operation": "update_column_properties",
  "table": "DimDate",
  "target": "MonthName",
  "spec": { "sort_by": "MonthNumber" }
}
```

### When to Use Hierarchies vs Flat Columns

**Use hierarchies when:**
- Natural drill-down path exists (time, geography, organization)
- Users need to navigate from summary to detail
- Multiple levels share a logical grouping

**Use flat columns when:**
- Columns are independent attributes (not hierarchical)
- Users typically filter rather than drill
- No natural parent-child relationship exists

### Performance Considerations

- Hierarchies themselves don't impact query performance significantly
- Underlying column cardinality matters (high-cardinality columns are slower)
- Drill-down increases query complexity with each level

## Common Hierarchy Patterns

### Date Hierarchy (Calendar)

```json
{
  "operation": "create_hierarchy",
  "table": "DimDate",
  "target": "Calendar",
  "spec": {
    "levels": [
      { "name": "Year", "column": "Year" },
      { "name": "Quarter", "column": "QuarterLabel" },
      { "name": "Month", "column": "MonthName" },
      { "name": "Date", "column": "Date" }
    ]
  }
}
```

### Date Hierarchy (Fiscal)

```json
{
  "operation": "create_hierarchy",
  "table": "DimDate",
  "target": "Fiscal Calendar",
  "spec": {
    "levels": [
      { "name": "Fiscal Year", "column": "FiscalYear" },
      { "name": "Fiscal Quarter", "column": "FiscalQuarter" },
      { "name": "Fiscal Month", "column": "FiscalMonth" }
    ]
  }
}
```

### Geography Hierarchy

```json
{
  "operation": "create_hierarchy",
  "table": "DimGeography",
  "target": "Geography",
  "spec": {
    "levels": [
      { "name": "Country", "column": "Country" },
      { "name": "State/Province", "column": "StateProvince" },
      { "name": "City", "column": "City" },
      { "name": "Postal Code", "column": "PostalCode" }
    ]
  }
}
```

### Product Hierarchy

```json
{
  "operation": "create_hierarchy",
  "table": "DimProduct",
  "target": "Product Categories",
  "spec": {
    "levels": [
      { "name": "Category", "column": "Category" },
      { "name": "Subcategory", "column": "Subcategory" },
      { "name": "Product", "column": "ProductName" }
    ]
  }
}
```

### Organization Hierarchy

```json
{
  "operation": "create_hierarchy",
  "table": "DimEmployee",
  "target": "Organization",
  "spec": {
    "levels": [
      { "name": "Division", "column": "Division" },
      { "name": "Department", "column": "Department" },
      { "name": "Team", "column": "Team" },
      { "name": "Employee", "column": "EmployeeName" }
    ]
  }
}
```

## Bulk Operations

### Create multiple hierarchies

```json
{
  "operation": "create_hierarchy",
  "transaction": true,
  "items": [
    {
      "table": "DimDate",
      "target": "Calendar",
      "spec": {
        "levels": [
          { "name": "Year", "column": "Year" },
          { "name": "Month", "column": "MonthName" }
        ]
      }
    },
    {
      "table": "DimProduct",
      "target": "Products",
      "spec": {
        "levels": [
          { "name": "Category", "column": "Category" },
          { "name": "Product", "column": "ProductName" }
        ]
      }
    }
  ]
}
```

### Delete multiple hierarchies

```json
{
  "operation": "delete_hierarchy",
  "transaction": true,
  "items": [
    { "table": "DimDate", "target": "Old Hierarchy" },
    { "table": "DimProduct", "target": "Deprecated Hierarchy" }
  ]
}
```

## Parent-Child Hierarchies

Power BI supports parent-child hierarchies (e.g., employee reporting structures) through DAX functions rather than standard hierarchies.

**Key DAX functions:**
- `PATH()`: Returns delimited path from child to root
- `PATHITEM()`: Extracts item at specific position in path
- `PATHLENGTH()`: Returns number of levels in path
- `PATHCONTAINS()`: Checks if value exists in path

**Typical pattern:**
1. Create a calculated column using `PATH(ChildKey, ParentKey)`
2. Create level columns using `PATHITEM(PathColumn, N)`
3. Create a standard hierarchy from the level columns

**Example:**

```dax
// Path column
Path = PATH('Employee'[EmployeeKey], 'Employee'[ManagerKey])

// Level columns
Level1 = PATHITEM('Employee'[Path], 1, INTEGER)
Level2 = PATHITEM('Employee'[Path], 2, INTEGER)
Level3 = PATHITEM('Employee'[Path], 3, INTEGER)
```

Then create a hierarchy from Level1, Level2, Level3 columns.

## Troubleshooting

### Hierarchy not appearing in visuals

- Check `is_hidden` is not `true`
- Verify table is not hidden
- Confirm hierarchy was saved successfully

### Drill-down not working as expected

- Verify level order (coarsest to finest)
- Check that columns have appropriate data
- Ensure sort order is correct for text columns

### Performance issues during drill-down

- Check cardinality of columns at each level
- Consider aggregations for deep drill paths
- Optimize underlying column data types

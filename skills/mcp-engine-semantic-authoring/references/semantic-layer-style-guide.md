# Semantic Layer Style Guide (Pro)

This guide defines conventions for naming, organization, and "public surface area" of a Power BI semantic model. It's intended to keep models consistent, discoverable, and safe to evolve.

## Related Tools

- Discovery: `list_model` (`operation: "list"`, `operation: "search"`)
- Authoring: `manage_schema` (tables, columns, relationships, hierarchies), `manage_semantic` (measures, calc groups), `manage_model_properties`
- Security: `manage_security` (roles, perspectives)

## 1) Model Surface Area: Public vs Internal

Establish a clear contract for consumers:

- **Public** objects: intended for report authors and end users.
- **Internal** objects: helpers used by measures/calculation items; hidden from clients.

Practical rules:

- Public measures: visible, described, placed in a curated folder.
- Internal measures: hidden (`is_hidden=true`) and stored in an `Internal\...` folder (or prefixed).
- Technical columns/keys: hidden unless they are user-facing slicers or used in visuals.

## 2) Naming Conventions

### Tables

- Use singular dimension names: `Customer`, `Product`, `Date`.
- For fact tables, choose one convention and apply consistently:
  - Prefixed: `FactSales`, `FactInventory` (traditional data warehouse style)
  - Business-friendly: `Sales`, `Orders`, `Inventory` (modern semantic layer style)
- Avoid spaces in technical/staging tables. Prefer user-facing, readable names for curated tables.

### Columns

- Use PascalCase or Title Case consistently (pick one across the model).
- Prefer descriptive names: `OrderDate`, `ShipDate`, `CustomerId`.
- Avoid ambiguous names like `Name` in multiple tables unless qualified by table context.

### Measures

Use noun/metric style names:

- `Total Sales`
- `Total Sales YTD`
- `Margin %`
- `Avg Order Value`

Avoid:

- `Measure1`, `Test`, `Calc`
- embedding table names unless needed for disambiguation

### Internal helpers

Choose one consistent pattern:

- Prefix: `_Total Sales (Internal)` or `_TotalSales_Internal`
  - and set `is_hidden=true`, or
- Folder: `Internal\...` (recommended for cleanliness)

## 3) Display Folders

Use folders to make the field list navigable.

Recommended folder structure (example):

- `Sales\Revenue`
- `Sales\Cost`
- `Sales\Margins`
- `Time Intelligence`
- `Internal\Time`
- `Internal\Base Measures`

Tool argument names:

- Measures: `display_folder`
- Columns/Hierarchies: `display_folder`

All tools use consistent `snake_case` naming. Folder path format is the same (e.g., `Sales\Revenue`).

## 4) Descriptions and Metadata

Public objects should have descriptions:

- What it represents
- Units / currency
- Grain / assumptions
- Any filters/logic caveats

Examples:

- Measure description: “Total sales amount in USD. Excludes returns.”
- Column description: “Customer segment label used for slicing.”

## 5) Format Strings

Use formats to prevent misinterpretation:

- Currency: `$#,0.00` or locale-appropriate (e.g., `€#,0.00`, `£#,0.00`)
- Percent: `0.0%` or `0.00%` for precision
- Integers/Counts: `#,0`
- Decimals: `#,0.00`
- Large numbers: `#,0,,"M"` (millions) or `#,0,"K"` (thousands)

Prefer dynamic `format_string_expression` only when needed (e.g., mixed units). Example:

```dax
IF([Value] >= 1000000, FORMAT([Value]/1000000, "#,0.0") & "M", FORMAT([Value], "#,0"))
```

Keep format expressions simple to avoid performance overhead.

## 6) Hiding Strategy

Hide:

- surrogate keys used only for relationships (`CustomerId`, `ProductId`)
- technical columns imported for refresh logic
- helper measures used only inside other measures

Keep visible:

- meaningful slicers and dimensions used by report authors
- business measures and KPIs

## 7) Perspectives (Curation)

Perspectives are not security boundaries; they're a curated view.

Recommended:

- Create at least one "Executive" or "Business" perspective containing:
  - key measures
  - key dimension tables (but not staging/technical tables)

Use `manage_security` with `operation: "create_perspective|update_perspective"` to align the field list with the intended audience.

## 8) Hierarchies

Hierarchies improve navigation and enable drill-down in visuals.

Naming conventions:

- Use descriptive names: `Date Hierarchy`, `Geography`, `Product Category`
- Avoid generic names like `Hierarchy1`

Level naming:

- Use business terms: `Year → Quarter → Month → Day`
- Be consistent across similar hierarchies
- Match column names when appropriate

When to use hierarchies vs flat columns:

- Use hierarchies for natural drill paths (time, geography, org structure)
- Keep flat columns for independent attributes that don't form a path

Use `manage_schema` with `operation: "create_hierarchy|update_hierarchy"` to create and maintain hierarchies.

## 9) Calculation Groups

Calculation groups modify measure behavior dynamically (e.g., time intelligence, currency conversion).

Naming conventions:

- Group names: descriptive of the transformation type (`Time Calculations`, `Currency Conversion`, `Scenario Analysis`)
- Item names: clear action descriptors (`YTD`, `Prior Year`, `USD`, `EUR`, `Budget`, `Forecast`)

Organization:

- Limit calculation groups to logically related items
- Consider precedence when multiple groups interact
- Document precedence order in group description

Best practices:

- Test with representative measures before deploying
- Hide the calculation group column if users shouldn't interact directly
- Use `SELECTEDMEASURE()` and `SELECTEDMEASURENAME()` for flexible expressions

Use `manage_semantic` with `operation: "create_calc_group|update_calc_group"` to create and maintain calculation groups.

## 10) Relationships

### Role-Playing Dimensions

When a dimension relates to a fact table multiple times (e.g., Date table for OrderDate and ShipDate):

- Name relationships descriptively using the role: `Order Date`, `Ship Date`, `Due Date`
- Only one relationship can be active; others must be inactive
- Document inactive relationships clearly
- Use `USERELATIONSHIP()` in measures that need inactive relationships

### Relationship Conventions

- Prefer single-direction cross-filtering unless bidirectional is required
- Document bidirectional relationships and their purpose
- Use consistent cardinality patterns (many-to-one from fact to dimension)

Use `manage_schema` with `operation: "create_relationship|update_relationship"` to configure relationships.

## 11) Security Roles

Row-Level Security (RLS) and Object-Level Security (OLS) control data access.

Naming conventions:

- Use descriptive role names: `Regional Sales Manager`, `Finance Team`, `External Partner`
- Avoid technical names like `Role1` or `TestRole`

Filter expression patterns:

- Keep filters simple and on dimension tables when possible
- Prefer `USERPRINCIPALNAME()` over `USERNAME()` for Azure AD environments
- Test with `run_query` using `CALCULATE(..., USERELATIONSHIP(...))` patterns

Documentation:

- Add descriptions explaining what each role restricts
- Document any OLS column/table restrictions

Testing:

- Validate role filters return expected results
- Test edge cases (users with no matching data, multiple role membership)

Use `manage_security` with `operation: "create_role|set_role_filters"` to create and maintain security roles.

## 12) Annotations

Use annotations to add custom metadata for governance and tooling:

- `Certified`: mark validated/approved objects
- `Owner`: identify responsible team or person
- `Department`: categorize by business area
- `LastReviewed`: track review dates

Annotations are key-value pairs that don't affect model behavior but aid discoverability and governance.

## 13) Date Tables

For time intelligence to work correctly, mark your date table:

Required setup:

- Set `isDateTable=true` on the table
- Designate the date column via `dateColumn` property
- Ensure the date column has no gaps and covers the full date range needed

Recommended columns:

- `Date` (the key column, date type)
- `Year`, `Quarter`, `Month`, `MonthName`, `Day`
- `WeekOfYear`, `DayOfWeek` (optional)
- `IsCurrentMonth`, `IsCurrentYear` (optional flags)

Naming:

- Use `Date` or `Calendar` as the table name
- Avoid `DimDate` unless following strict data warehouse conventions

## 14) Safe Refactors (Renames and Deletions)

Renames do not automatically rewrite dependent DAX/M expressions.

Workflow:

1. Locate dependents:
   - `list_model` with `operation: "search"` and `spec: { mode: "dax", query: "[MeasureName]" }` for measure references
   - `list_model` with `operation: "search"` and `spec: { mode: "m", query: "RangeStart" }` for M references (named expressions + M partitions)
   - `list_model` with `operation: "search"` and `spec: { mode: "partition", query: "SELECT " }` for non-M partition expressions (SQL / Calculated / Entity)
2. Rename the object.
3. Update dependents in the same session to maintain consistency.
4. Validate:
   - `run_query` for correctness queries
   - `run_query` with `operation: "analyze"` for performance regressions on representative queries

## 15) Field Parameters

Field parameters allow users to dynamically switch measures or dimensions in visuals via slicers.

### Naming Conventions

- Parameter table names: descriptive of the choice type (`Metric Selector`, `Time Period`, `Comparison Axis`)
- Avoid generic names like `Parameter1` or `Fields`

### When to Use Field Parameters

- **Measure selection**: Let users choose between Revenue, Profit, Units, etc.
- **Dimension selection**: Dynamic axis/legend switching (by Category, by Region, by Product)
- **Time comparisons**: YTD vs MTD vs Full Year
- **Scenario analysis**: Budget vs Actual vs Forecast

### Implementation Notes

Field parameters are calculated tables with special annotations. They contain:
- A text column for display names
- A column containing the actual field references
- An ordinal column for sort order

```dax
// Example field parameter (created via Power BI Desktop UI)
Metric Selector = {
    ("Total Sales", NAMEOF('Sales'[Total Sales]), 0),
    ("Total Profit", NAMEOF('Sales'[Total Profit]), 1),
    ("Total Units", NAMEOF('Sales'[Total Units]), 2)
}
```

### Best Practices

- Use clear display names that match measure/column names
- Keep the number of options manageable (3-7 items typically)
- Consider creating separate field parameters for different use cases
- Document the parameter's purpose in the table description
- Hide the underlying measures if users should only access them via the parameter

### Limitations

- Field parameters work only in Power BI Service and Desktop (not in Excel)
- Cannot combine measures and columns in the same field parameter
- Performance impact is minimal but test with representative queries

## 16) Release Checklist (Model Hygiene)

Before shipping a model update:

- [ ] `list_model` with `operation: "list"`, `spec: { type: "measures" }`: no "test" measures left visible
- [ ] `list_model` with `operation: "search"`, `spec: { query: "temp", mode: "name" }`: no temporary objects
- [ ] `list_model` with `operation: "list"`, `spec: { type: "relationships" }`: no unintended bidirectional relationships
- [ ] Date table marked with `isDateTable=true` (if using time intelligence)
- [ ] Top KPIs validated with `run_query`
- [ ] All public objects have descriptions
- [ ] Format strings applied to all numeric measures
- [ ] Internal/helper objects are hidden
- [ ] Security roles tested (if applicable)
- [ ] Field parameters documented and tested (if applicable)
- [ ] Model changes reviewed via `../../mcp-engine-testing-changes/references/model-changes-guide.md` (Pro)

# DAX Query Writing Guide

This guide provides comprehensive instructions for writing DAX queries using the `run_query` tool.

## Related Tools

- `run_query`: Execute DAX queries and return results (`operation: "execute"`)
- `list_model`: Discover model schema (`operation: "list"`, `spec: { type: "tables|columns|measures|relationships" }`)
- `list_model`: Understand table data before writing queries (`operation: "analyze"`, `spec: { mode: "describe|preview" }`)
  - `describe` filters Power BI system artifacts by default; set `include_system_artifacts: true` only when you need raw internal metadata
- `manage_semantic`: Create DAX measures for reusable calculations (`operation: "create_measure"`)

## Tool Invocation

The `run_query` tool requires `operation: "execute"` and a `query` parameter containing the DAX query:

```json
{
  "operation": "execute",
  "query": "EVALUATE TOPN(10, 'Sales') ORDER BY 'Sales'[Amount] DESC"
}
```

**Simple query example:**

```json
{
  "operation": "execute",
  "query": "EVALUATE ROW(\"Total\", SUM('Sales'[Amount]))"
}
```

**Query with DEFINE block:**

```json
{
  "operation": "execute",
  "query": "DEFINE MEASURE 'Sales'[AvgAmount] = AVERAGE('Sales'[Amount]) EVALUATE SUMMARIZECOLUMNS('Product'[Category], \"Avg\", [AvgAmount]) ORDER BY 'Product'[Category]"
}
```

## Response Format

Query results are returned in a compact **tabular format** optimized for LLM consumption:

```json
{
  "columns": ["Category", "TotalSales"],
  "display_columns": ["Category", "TotalSales*"],
  "masking_indicator": {
    "marker": "*",
    "meaning": "value masked (PII or numeric)"
  },
  "rows": [
    ["Electronics", 15000.50],
    ["Clothing", 8500.25]
  ],
  "row_count": 2,
  "truncated": false
}
```

**Key response behaviors:**

- **`columns`**: Array of column names (strings)
- **`display_columns`**: Optional array of display-only column names for model-visible responses. Present when PII or numeric masking was applied and a column header should show a marker such as `*`. Raw `columns` always stay stable.
- **`masking_indicator`**: Optional machine-readable explanation for display-only masking markers, such as `{"marker":"*","meaning":"value masked (PII or numeric)"}`.
- **`rows`**: Array of row arrays (each row is `object?[]`, values in column order)
- **`query`**: Omitted by default. Use `spec: { include_query: true }` to echo the query back
- **`truncated`**: `true` if results were capped by row limits

### Masking Metadata

- `run_query` `operation: "execute"` may include `display_columns` for model-visible calls when PII or numeric masking was applied.
- `run_query` `operation: "test_access"` may include `results.<role>.numeric_masked` for model-visible calls when the returned scalar value came from a numerically masked column.
- `list_model` `operation: "analyze"` may include `display_columns` for `preview` mode and root-level `display_column` for `column_stats` mode when PII or numeric masking was applied.
- When marker metadata is present, `masking_indicator` explains the marker so clients and LLMs can render a legend without guessing.
- App-internal tool surfaces keep raw payloads presentation-free and do not emit these display-only masking markers.

**Row limits:**

- Default: **1,000 rows** (LLM-friendly to avoid context overflow)
- Use `spec: { max_rows: N }` to request fewer rows (capped by server config)
- Server config `MCP_ENGINE_MAX_QUERY_ROWS` sets the default cap
- Runtime override: `manage_preferences` `{ action: "set", resource: "setting", id: "max_query_rows", value: "500" }` (takes effect immediately)

**Example with options:**

```json
{
  "operation": "execute",
  "query": "EVALUATE 'LargeTable'",
  "spec": {
    "max_rows": 50,
    "include_query": true
  }
}
```

## DAX Query Best Practices

When writing DAX queries, follow these recommendations:

- Include comments for clarity (DAX supports `//` for single-line and `/* */` for block comments; do not use `--`)
- Always include an ORDER BY clause when returning multiple rows
- Use meaningful variable names to improve readability
- Define measures with fully qualified names in DEFINE blocks
- Use `list_model` first to discover available tables, columns, and measures

## Quoting Rules

- **Table names**: Use single quotes: `'Sales'`, `'Product Category'`
- **Column names**: Use brackets: `[Amount]`, `[Product Name]`
- **Fully qualified**: `'Sales'[Amount]`, `'Product Category'[Name]`
- **Measures**: Use brackets only: `[Total Sales]` (or fully qualified: `'Sales'[Total Sales]`)

## DAX Query Syntax Rules

### Query Structure

#### DEFINE Block

- Use DEFINE at the beginning if the query includes VAR, MEASURE, COLUMN, or TABLE definitions
- Only use a single DEFINE block per query
- Separate definitions with new lines (no commas or semicolons)

#### Measure Definitions

- **When defining**: ALWAYS fully qualify the measure name including its host table
  - Example: `DEFINE MEASURE 'TableName'[MeasureName] = ...`
  - The host table must exist in the model
- **When using**: Refer to the measure by name only, without the table qualifier
  - Example: Use `[MeasureName]` in expressions like `CALCULATE([MeasureName], ...)`

#### Ordering Results

- ALWAYS include an ORDER BY clause when EVALUATE returns multiple rows
- Do not use the ORDERBY function to sort the final query result

### CALCULATE and CALCULATETABLE Filter Rules

Boolean filters in CALCULATE or CALCULATETABLE have important restrictions:

- Cannot directly use a measure or another CALCULATE function
  - **Solution**: Use a variable to store the result, then reference the variable
- Cannot reference columns from two different tables
- When using the IN operator, the table operand must be a table variable, not a table expression
- Do not assign a boolean filter to a VAR definition

### SUMMARIZECOLUMNS Function

**Purpose**: Build summary tables with groupby columns and measure-like extension columns

**Parameter Order** (all optional, but must follow this order if used):

1. Groupby columns (can be from one or multiple tables)
2. Filters
3. Measures or measure-like calculations

**Key Rules**:

- Use SUMMARIZECOLUMNS as the default for building summary tables with measures
- Do not use SUMMARIZECOLUMNS without measure-like extension columns
- Returns only rows where at least one measure value is not BLANK
- Allows ANY number of measure-like calculations of arbitrary complexity
- DO NOT use boolean filters with SUMMARIZECOLUMNS

**When to Use Alternatives**:

- If there are no measures or calculations, use SUMMARIZE instead

### SUMMARIZE Function

**Allowed Pattern**:

```dax
SUMMARIZE(<table expression>, <column1>, ..., <columnN>)
```

**Critical Restrictions**:

- NEVER use SUMMARIZE with measure-like expressions
  - Incorrect: `SUMMARIZE(<table>, <column>, "expr1", <expr1>, ...)`
  - Correct: Use SUMMARIZECOLUMNS for measure calculations
- Use for extracting distinct combinations of columns only
- `VALUES('Table'[Column])` is a shortcut for `SUMMARIZE('Table', 'Table'[Column])`
- When extracting a column from a table variable: `SUMMARIZE(_TableVar, [Column])`
  - Note: `_TableVar[Column]` is not valid syntax

**When to Use Alternatives**:

- For measure calculations: Use SUMMARIZECOLUMNS
- For aggregations on table variables: Use GROUPBY

### GROUPBY Function

**Purpose**: Perform simple aggregations on table-valued variables at a grouped level

**Key Rules**:

- Only use GROUPBY with a table-valued variable as the first argument
- The CURRENTGROUP function is valid ONLY within GROUPBY
- CURRENTGROUP must not be used elsewhere

### SELECTCOLUMNS Function

**Purpose**: Project columns while preserving duplicates or renaming columns

**Key Rules**:

- Use to preserve duplicate rows (unlike SUMMARIZE which removes them)
- Use to rename columns for clarity
- When renaming columns, subsequent expressions (TOPN, ORDER BY) must use the NEW column names

**Important**: Include all columns needed for later operations (ORDER BY, FILTER, etc.)

### Table Expressions and Filters

- When using table expressions (SELECTCOLUMNS, CALCULATETABLE), include any columns needed later
- Filters applied to one table can propagate across relationships based on filter direction

### Set Functions

When using INTERSECT, UNION, or EXCEPT:

- Both input tables must produce an identical number of columns

### Time Intelligence Functions

**DATESINPERIOD Rolling Windows**:

- The negative period offset must precisely match the number of periods required
- Examples:
  - 12-month window: Use -12 (not -11)
  - 3-month window: Use -3 (not -2)
- This prevents off-by-one errors

**Maintaining Clear Date Context**:

- Always establish a valid date context for time intelligence calculations
- Methods:
  - Include groupby columns from the date table, OR
  - Apply filters on date columns
- Without date context, time intelligence functions cannot determine a "current date" reference
- When using ROW function with time intelligence, supply external filters through CALCULATETABLE

## DAX Query Examples

The following examples demonstrate proper DAX query syntax and best practices.

### Example 1: Time Intelligence with Rolling Averages

**Scenario**: Calculate year-to-date total sales and 14-day moving average for red products.

```dax
// Year-to-date total sales and 14-day moving average of sales for red products.
EVALUATE
  CALCULATETABLE(
    ROW(
      "Total Sales Amount YTD", TOTALYTD([Total Amount], 'Calendar'[Date]),
      "Total Sales Amount 14-Day MA", AVERAGEX(DATESINPERIOD('Calendar'[Date], MAX('Calendar'[Date]), -14, DAY), [Total Amount])
    ),
    'Product'[Color] == "Red",
    TREATAS({ MAX('Sales'[OrderDate]) }, 'Calendar'[Date])
  )
```

**Key Concepts**:

- Uses CALCULATETABLE with ROW to establish date context for time intelligence functions
- TREATAS establishes a clear "current date" reference for time intelligence
- DATESINPERIOD uses -14 (not -13) for a proper 14-day window

### Example 2: Multi-Level Aggregation with GROUPBY

**Scenario**: Calculate average, minimum, and maximum monthly sales quantity by year for Consumer Electronics before 2023.

```dax
DEFINE
  VAR _Filter1 = TREATAS(
    { "Consumer Electronics" },
    'Product'[Category]
  )
  VAR _Filter2 = FILTER(
    ALL('Calendar'[Year]),
    'Calendar'[Year] < 2023
  )
  VAR _SummaryTable = SUMMARIZECOLUMNS(
    'Calendar'[Year],
    'Calendar'[Month],
    'Calendar'[MonthNumberOfYear],
    _Filter1,
    _Filter2,
    "Monthly Quantity", [Total Quantity]
  )

EVALUATE
  GROUPBY(
    _SummaryTable,
    'Calendar'[Year],
    'Calendar'[Month],
    'Calendar'[MonthNumberOfYear],
    "Avg Monthly Quantity", AVERAGEX(CURRENTGROUP(), [Monthly Quantity]),
    "Min Monthly Quantity", MINX(CURRENTGROUP(), [Monthly Quantity]),
    "Max Monthly Quantity", MAXX(CURRENTGROUP(), [Monthly Quantity])
  )
  ORDER BY
    'Calendar'[Year] ASC,
    'Calendar'[MonthNumberOfYear] ASC
```

**Key Concepts**:

- SUMMARIZECOLUMNS creates initial summary with measures
- GROUPBY performs secondary aggregation on the summary table
- CURRENTGROUP is used exclusively within GROUPBY
- Includes MonthNumberOfYear for proper sorting

### Example 3: Filtering with Measures Using Variables

**Scenario**: Find products with total sales over $1 million that are red or black.

```dax
DEFINE
  VAR _Filter = TREATAS(
    { "Red", "Black" },
    'Product'[Color]
  )
  VAR _SummaryTable = SUMMARIZECOLUMNS(
    'Product'[Name],
    _Filter,
    "Total Sales", [Total Amount]
  )

EVALUATE
  SELECTCOLUMNS(
    FILTER(_SummaryTable, [Total Sales] > 1000000),
    'Product'[Name]
  )
  ORDER BY
    'Product'[Name] ASC
```

**Key Concepts**:

- TREATAS creates a filter from a list of values
- SUMMARIZECOLUMNS builds summary with measure
- FILTER applied to summary table variable
- SELECTCOLUMNS projects only needed columns

### Example 4: SUMMARIZE for Distinct Values

**Scenario**: Get unique combinations of color, category, and product key for products sold in 2022.

```dax
EVALUATE
  CALCULATETABLE(
    SUMMARIZE(
      'Sales',
      'Product'[Color],
      'Product'[Category],
      'Sales'[ProductKey]
    ),
    'Calendar'[Year] == 2022
  )
  ORDER BY
    'Product'[Color] ASC,
    'Product'[Category] ASC,
    'Sales'[ProductKey] ASC
```

**Key Concepts**:

- SUMMARIZE removes duplicate rows
- CALCULATETABLE applies year filter
- Related table columns accessed via relationships

### Example 5: SELECTCOLUMNS for Preserving Duplicates

**Scenario**: Get color, category, and product key for products sold in 2022, keeping all duplicate rows.

```dax
EVALUATE
  CALCULATETABLE(
    SELECTCOLUMNS(
      'Sales',
      "Color", RELATED('Product'[Color]),
      "Category", RELATED('Product'[Category]),
      'Sales'[ProductKey]
    ),
    'Calendar'[Year] == 2022
  )
  ORDER BY
    [Color] ASC,
    [Category] ASC,
    'Sales'[ProductKey] ASC
```

**Key Concepts**:

- SELECTCOLUMNS preserves duplicate rows (unlike SUMMARIZE)
- Column renaming requires using new names in ORDER BY
- RELATED accesses columns from related tables

### Example 6: Finding Products with No Sales

**Scenario**: Identify products that have never been sold.

```dax
DEFINE
  MEASURE 'Sales'[Row Count] = COUNTROWS('Sales')

EVALUATE
  FILTER(
    'Product',
    ISBLANK([Row Count])
  )
  ORDER BY
    'Product'[Name] ASC,
    'Product'[ProductKey] ASC
```

**Key Concepts**:

- Defines a measure for row counting
- FILTER with ISBLANK identifies products with no related sales
- Filter context propagates from Product to Sales table

### Example 7: Using Variables to Store Measure Results

**Scenario**: Find products with list prices above the median.

```dax
DEFINE
  VAR _MedianListPrice = [Median List Price]

EVALUATE
  CALCULATETABLE(
    VALUES('Product'[Name]),
    'Product'[List Price] > _MedianListPrice
  )
  ORDER BY
    'Product'[Name] ASC
```

**Key Concepts**:

- Variables store measure results for use in boolean filters
- Cannot use measures directly in CALCULATE boolean filters
- Variable evaluation happens before the filter context

### Example 8: Using Table Variables as Filters

**Scenario**: Find the product with highest demand since 2020 and get its sale dates.

```dax
DEFINE
  VAR _Filter = FILTER(
    ALL('Calendar'[Year]),
    'Calendar'[Year] >= 2020
  )
  VAR _TopProduct = TOPN(
    1,
    SUMMARIZECOLUMNS(
      'Product'[ProductKey],
      _Filter,
      "Total Quantity", [Total Quantity]
    ),
    [Total Quantity], DESC
  )

EVALUATE
  SELECTCOLUMNS(
    CALCULATETABLE('Sales', _TopProduct),
    "Product Name", RELATED('Product'[Name]),
    'Sales'[OrderDate]
  )
  ORDER BY
    [Product Name] ASC,
    'Sales'[OrderDate] ASC
```

**Key Concepts**:

- TOPN identifies the product with highest total quantity
- Table variables can be used directly as filters in CALCULATETABLE
- Calculated columns in table variables don't affect filter context

### Example 9: Calculating Averages on Filtered Subsets

**Scenario**: Calculate the average list price of products sold in 2022.

```dax
DEFINE
  VAR _ProductsSold2022 = CALCULATETABLE(
    SUMMARIZE('Sales', 'Product'[ProductKey]),
    'Calendar'[Year] == 2022
  )

EVALUATE
  ROW(
    "Average List Price of Products Sold in 2022",
    CALCULATE(AVERAGE('Product'[List Price]), _ProductsSold2022)
  )
```

**Key Concepts**:

- SUMMARIZE extracts distinct products sold in 2022
- ROW returns a single-row result
- Table variable applied as filter in CALCULATE

### Example 10: Column Renaming with SELECTCOLUMNS

**Scenario**: Get products sold and customers, sorted by renamed column names.

```dax
DEFINE
  VAR _UniqueProductCustomerPairs = SUMMARIZE(
    'Sales',
    'Product'[Name],
    'Customer'[Name]
  )

EVALUATE
  SELECTCOLUMNS(
    _UniqueProductCustomerPairs,
    "Product Name", 'Product'[Name],
    "Customer Name", 'Customer'[Name]
  )
  ORDER BY
    [Product Name] ASC,
    [Customer Name] ASC
```

**Key Concepts**:

- SELECTCOLUMNS renames columns for clarity
- ORDER BY must reference the NEW column names, not original names
- SUMMARIZE removes duplicate product-customer pairs

### Example 11: Multiple Filters with TREATAS

**Scenario**: For the three oldest customers, show total discounts by year for the last three years.

```dax
DEFINE
  VAR _OldestThreeCustomers = TOPN(3, 'Customer', 'Customer'[Age], DESC)
  VAR _LastYear = YEAR(MAX('Sales'[OrderDate]))

EVALUATE
  SUMMARIZECOLUMNS(
    'Customer'[Name],
    'Calendar'[Year],
    TREATAS({_LastYear, _LastYear - 1, _LastYear - 2}, 'Calendar'[Year]),
    _OldestThreeCustomers,
    "Total Discount", [Total Discount]
  )
  ORDER BY
    'Customer'[Name] ASC,
    'Calendar'[Year] ASC
```

**Key Concepts**:

- TOPN selects top 3 customers by age
- TREATAS creates filter from calculated year values
- Determines last year from actual sales data, not calendar table

### Example 12: Filtering Aggregated Results

**Scenario**: For each customer, show products purchased at least three times with purchase count.

```dax
DEFINE
  MEASURE 'Sales'[Purchase Count] = COUNTROWS('Sales')
  VAR _SummaryTable = SUMMARIZECOLUMNS(
    'Customer'[Name],
    'Product'[Name],
    "Purchase Count", [Purchase Count]
  )

EVALUATE
  FILTER(_SummaryTable, [Purchase Count] >= 3)
  ORDER BY
    'Customer'[Name] ASC,
    'Product'[Name] ASC
```

**Key Concepts**:

- Defines measure for counting purchases
- SUMMARIZECOLUMNS creates summary with measure
- FILTER applied to summary table variable
- Simple and efficient two-step pattern

### Example 13: Calculated Columns in DEFINE

**Scenario**: Categorize products by price (above/below median) and show max/min quantities per category.

```dax
DEFINE
  COLUMN 'Product'[Price Group] =
    VAR _MedianListPrice = [Median List Price]
    RETURN
    IF('Product'[List Price] > _MedianListPrice, "High Priced", "Low Priced")
  MEASURE 'Sales'[Max Quantity] = MAX('Sales'[Order Quantity])
  MEASURE 'Sales'[Min Quantity] = MIN('Sales'[Order Quantity])

EVALUATE
  SUMMARIZECOLUMNS(
    'Product'[Price Group],
    "Max Quantity", [Max Quantity],
    "Min Quantity", [Min Quantity]
  )
  ORDER BY 'Product'[Price Group] ASC
```

**Key Concepts**:

- COLUMN defines a calculated column in DEFINE block
- Calculated columns can be used as groupby columns in SUMMARIZECOLUMNS
- Multiple measures defined and used in same query
- IF expression for conditional logic

## Summary

Key takeaways for writing valid DAX queries:

1. **Always use ORDER BY** when returning multiple rows
2. **Store measure results in variables** before using in boolean filters
3. **Choose the right function**: SUMMARIZECOLUMNS for measures, SUMMARIZE for distinct values, GROUPBY for table variables
4. **Establish date context** for time intelligence functions
5. **Use renamed columns** in ORDER BY after SELECTCOLUMNS
6. **Leverage table variables** as filters for cleaner, more maintainable code
7. **Use `list_model`** to discover the model schema before writing queries

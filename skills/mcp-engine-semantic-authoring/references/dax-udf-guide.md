# DAX User-Defined Functions Guide

This guide explains how to create and manage User-Defined Functions (UDFs) in Power BI using `manage_semantic`.

## Related Tools

- `manage_semantic`: Create, update, or delete UDFs (operations: `create_udf`, `update_udf`, `delete_udf`)
- `list_model`: List existing UDFs in the model (`operation: "list"`, `spec: { type: "udfs" }`)

## Overview

DAX User-Defined Functions (UDFs) allow you to create reusable function definitions in Power BI semantic models. UDFs require model compatibility level 1702 or higher.

## Basic Syntax

A UDF definition consists of:

```
(param1 [: Type [Subtype] [Mode]], param2 [: Type [Subtype] [Mode]], ...) [: ReturnType [ReturnSubtype]] => <Function body>
```

### Return Type Hints (Optional)

You can optionally annotate the return type for clarity and documentation:

```dax
(radius : SCALAR NUMERIC) : SCALAR NUMERIC => PI() * radius * radius
```

Return type annotations:
- `SCALAR [Subtype]`: Returns a single value (with optional subtype like NUMERIC, STRING, BOOLEAN, DATETIME, VARIANT)
- `TABLE`: Returns a table expression

Return type hints are optional but improve readability and can help catch type mismatches during authoring.

## Type System

### Parameter Types

DAX UDFs support these parameter types:

| Type | Description | Use Case |
|------|-------------|----------|
| ANYVAL | No explicit type hint (engine infers). This is the default when you omit `type` and `subtype`. | Flexible inputs when you don't want to constrain the caller |
| SCALAR | A single value (number, text, date/time, boolean) | Most calculations |
| TABLE | A DAX table expression | Working with datasets |
| ANYREF | A direct reference to a model object without pre-evaluation | Passing columns/measures to functions like CALCULATE |

### Scalar Subtypes

When using `SCALAR` type, you can optionally specify a subtype:

| Subtype | Description |
|---------|-------------|
| INT64 | Integer values |
| DECIMAL | Decimal numbers |
| DOUBLE | Double-precision floating-point |
| STRING | Text values |
| DATETIME | Date and time values |
| BOOLEAN | True/false values |
| NUMERIC | Any numeric type (INT64, DECIMAL, or DOUBLE) |
| VARIANT | Any scalar type (use when the expression may yield different types) |

**Note**: `BLANK()` is valid for any subtype.

### AnyRef Type

Use `ANYREF` when you need a direct reference to a model object rather than its evaluated value. This is useful for functions that need to pass references to functions like CALCULATE, TREATAS, or SAMEPERIODLASTYEAR.

Allowed reference forms:
- Column reference: `'Table'[Column]`
- Table reference: `'Table'`
- Measure reference: `[Measure]`

### Parameter Modes

Parameters can be evaluated in two modes:

| Mode | Description | Use When |
|------|-------------|----------|
| VAL | Default. Argument is evaluated at call site before entering function. | Simple calculations where you want the value |
| EXPR | Raw argument expression is substituted into function body and evaluated in inner context. | Context control with CALCULATE, FILTER, or iteration functions |

## Tool Usage

### Creating a UDF

```json
{
  "operation": "create_udf",
  "target": "CircleArea",
  "spec": {
    "body": "PI() * radius * radius",
    "parameters": [
      {"name": "radius", "subtype": "NUMERIC"}
    ],
    "description": "Calculate the area of a circle from its radius"
  }
}
```

### Creating with Multiple Parameters

```json
{
  "operation": "create_udf",
  "target": "DoubleValue",
  "spec": {
    "body": "inputValue * 2",
    "parameters": [
      {"name": "inputValue", "type": "SCALAR", "subtype": "NUMERIC", "mode": "VAL"}
    ]
  }
}
```

### Creating with AnyRef Parameter

```json
{
  "operation": "create_udf",
  "target": "Mode",
  "spec": {
    "body": "MINX(TOPN(1, ADDCOLUMNS(VALUES(col), \"Freq\", CALCULATE(COUNTROWS(tab))), [Freq], DESC), col)",
    "parameters": [
      {"name": "tab", "type": "ANYREF"},
      {"name": "col", "type": "ANYREF"}
    ],
    "description": "Returns the most frequently occurring value in a column"
  }
}
```

### Creating with EXPR Mode for Context Control

```json
{
  "operation": "create_udf",
  "target": "PriorYearValue",
  "spec": {
    "body": "CALCULATE(expression, SAMEPERIODLASTYEAR(dateColumn))",
    "parameters": [
      {"name": "expression", "type": "SCALAR", "subtype": "VARIANT", "mode": "EXPR"},
      {"name": "dateColumn", "type": "ANYREF"}
    ],
    "description": "Evaluate any scalar expression in the prior year"
  }
}
```

### Updating a UDF

```json
{
  "operation": "update_udf",
  "target": "CircleArea",
  "spec": {
    "body": "PI() * POWER(radius, 2)",
    "description": "Calculate the area of a circle (updated formula)"
  }
}
```

### Renaming a UDF

```json
{
  "operation": "update_udf",
  "target": "CircleArea",
  "spec": { "new_name": "CalculateCircleArea" }
}
```

### Deleting a UDF

```json
{
  "operation": "delete_udf",
  "target": "CircleArea"
}
```

### Hiding a UDF from IntelliSense

```json
{
  "operation": "update_udf",
  "target": "InternalHelper",
  "spec": { "is_hidden": true }
}
```

## Examples

### Example 1: Simple Numeric Calculation

Calculate the area of a circle from its radius.

```json
{
  "operation": "create_udf",
  "target": "CircleArea",
  "spec": {
    "body": "PI() * radius * radius",
    "parameters": [{"name": "radius", "subtype": "NUMERIC"}]
  }
}
```

**Usage**: `CircleArea(5)` returns approximately 78.54

### Example 2: Statistical Function with AnyRef

Returns the most frequently occurring value in a column.

```json
{
  "operation": "create_udf",
  "target": "Mode",
  "spec": {
    "body": "MINX(TOPN(1, ADDCOLUMNS(VALUES(col), \"Freq\", CALCULATE(COUNTROWS(tab))), [Freq], DESC), col)",
    "parameters": [
      {"name": "tab", "type": "ANYREF"},
      {"name": "col", "type": "ANYREF"}
    ]
  }
}
```

**Explanation**: Uses `ANYREF` for both parameters because it needs to pass the table and column references to DAX functions like VALUES and CALCULATE.

**Usage**: `Mode('Sales', 'Sales'[ProductKey])`

### Example 3: Time Intelligence with EXPR Mode

Evaluate any scalar expression in the prior year.

```json
{
  "operation": "create_udf",
  "target": "PriorYearValue",
  "spec": {
    "body": "CALCULATE(expression, SAMEPERIODLASTYEAR(dateColumn))",
    "parameters": [
      {"name": "expression", "type": "SCALAR", "subtype": "VARIANT", "mode": "EXPR"},
      {"name": "dateColumn", "type": "ANYREF"}
    ]
  }
}
```

**Explanation**: The `expression` parameter uses `EXPR` mode so it's evaluated within the CALCULATE context with the prior year filter applied. The `dateColumn` uses `ANYREF` to pass the column reference to SAMEPERIODLASTYEAR.

**Usage**: `PriorYearValue([Total Amount], 'Calendar'[Date])`

### Example 4: Returning a Table Filter

Return today's date as a one-row table for filtering.

```json
{
  "operation": "create_udf",
  "target": "TodayAsDate",
  "spec": {
    "body": "TREATAS({ TODAY() }, 'Date'[Date])",
    "parameters": []
  }
}
```

**Explanation**: This function returns a table. It uses TREATAS to convert the single-value table into a filter compatible with the Date table.

**Usage**: `CALCULATE([Total Amount], TodayAsDate())`

### Example 5: Table-Returning Function

Return a table of the top 3 Products by sales.

```json
{
  "operation": "create_udf",
  "target": "Top3ProductsBySales",
  "spec": {
    "body": "TOPN(3, VALUES('Product'[ProductKey]), [Sales], DESC)",
    "parameters": []
  }
}
```

**Usage**: `CALCULATE([Total Amount], Top3ProductsBySales())`

### Example 6: String Manipulation with Table Return

Split a text by a delimiter and return a single-column table.

```json
{
  "operation": "create_udf",
  "target": "SplitString",
  "spec": {
    "body": "VAR str = SUBSTITUTE(s, delimiter, \"|\") VAR len = PATHLENGTH(str) RETURN SELECTCOLUMNS(GENERATESERIES(1, len), \"Value\", PATHITEM(str, [Value], TEXT))",
    "parameters": [
      {"name": "s", "subtype": "STRING"},
      {"name": "delimiter", "subtype": "STRING"}
    ]
  }
}
```

**Explanation**: Uses variables (VAR) to build complex logic. Converts the delimiter to a path separator, counts the parts, and returns a table with each part as a row.

**Usage**: `SplitString("apple,banana,cherry", ",")`

### Example 7: Conditional Logic

Categorize a value based on thresholds.

```json
{
  "operation": "create_udf",
  "target": "GetPriceCategory",
  "spec": {
    "body": "SWITCH(TRUE(), price <= 10, \"Low\", price <= 50, \"Medium\", price <= 100, \"High\", \"Premium\")",
    "parameters": [
      {"name": "price", "subtype": "NUMERIC"}
    ],
    "description": "Categorize price into Low/Medium/High/Premium"
  }
}
```

**Usage**: `GetPriceCategory(75)` returns "High"

## Best Practices

1. **Use appropriate type hints**: Specify types and subtypes to make your functions more robust and self-documenting.

2. **Choose the right parameter mode**:
   - Use `VAL` (default) when you want the argument evaluated once at call time
   - Use `EXPR` when you need the expression to be evaluated in the function's context (e.g., inside CALCULATE)

3. **Use ANYREF for references**: When passing columns, tables, or measures to DAX functions that expect references, use `ANYREF`.

4. **Document your functions**: Include clear descriptions via the `description` parameter in spec.

5. **Test with edge cases**: Consider BLANK values and empty tables in your function logic.

6. **Keep functions focused**: Each function should have a single, well-defined purpose.

7. **Use variables**: For complex functions, use VAR to break down logic and improve readability.

8. **Hide internal helpers**: Use `is_hidden: true` in spec for utility functions not intended for end users.

## Parameter Format Reference

```json
{
  "name": "parameterName",
  "type": "ANYVAL | SCALAR | TABLE | ANYREF",
  "subtype": "NUMERIC | STRING | BOOLEAN | DATETIME | INT64 | DECIMAL | DOUBLE | VARIANT",
  "mode": "VAL | EXPR"
}
```

- `name`: Required. The parameter name used in the function body.
- `type`: Optional. If omitted, the parameter is untyped (`ANYVAL`) and the engine infers it. Use TABLE for table inputs, ANYREF for column/measure/table references.
- `subtype`: Optional. Scalar subtype hint (applies when type is SCALAR, or when type is omitted and you want to hint a scalar subtype).
- `mode`: Optional. Defaults to VAL. Use EXPR for lazy evaluation with context control.

# DAX User-Defined Functions Guide

This guide explains how to create and manage User-Defined Functions (UDFs) in Power BI using `manage_semantic`.

## Related Tools

- `manage_semantic`: Create, update, or delete UDFs (operations: `create_udf`, `update_udf`, `delete_udf`)
- `list_model`: List existing UDFs in the model (`operation: "list"`, `spec: { type: "udfs" }`)

## Overview

DAX User-Defined Functions (UDFs) allow you to create reusable function definitions in Power BI semantic models. UDFs require model compatibility level 1702 or higher.

Microsoft documents DAX UDFs as generally available in Power BI Desktop and Power BI Service as of the June 2026 release, including optional parameter default expressions. See Microsoft's [DAX user-defined functions](https://learn.microsoft.com/en-us/dax/best-practices/dax-user-defined-functions) documentation and SQLBI's [optional parameters in DAX user-defined functions](https://www.sqlbi.com/articles/optional-parameters-in-dax-user-defined-functions/) article for syntax and call-site examples.

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
| ANYREF | Any unevaluated reference/expression; always expression-passed | Flexible reference/expression parameters when stricter reference types do not fit |
| MEASUREREF | A measure reference from the semantic model | Parameters that must be measures |
| COLUMNREF | A model column reference | Model-independent functions that operate on caller-supplied columns |
| TABLEREF | A model table reference | Functions that require an actual model table, not a derived table expression |
| CALENDARREF | A model calendar reference | Calendar-based time intelligence functions |

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

### Reference Types

Use reference types when you need an unevaluated model reference or expression rather than an already evaluated value. `ANYREF` is the most permissive option. Prefer the more specific `MEASUREREF`, `COLUMNREF`, `TABLEREF`, or `CALENDARREF` type when the function requires that kind of model object; the engine and authoring tools can validate the call site more clearly.

Allowed reference forms:
- Column reference: `'Table'[Column]`
- Table reference: `'Table'`
- Measure reference: `[Measure]`
- Calendar reference: `'Calendar'`

### Parameter Modes

Parameters can be evaluated in two modes:

| Mode | Description | Use When |
|------|-------------|----------|
| VAL | Default. Argument is evaluated at call site before entering function. | Simple calculations where you want the value |
| EXPR | Raw argument expression is substituted into function body and evaluated in inner context. | Context control with CALCULATE, FILTER, or iteration functions |

### Optional Parameters

You can make a parameter optional by setting `default_expression` on that parameter. SemanticOps renders it in the UDF signature as `= <DefaultExpression>`:

```json
{
  "operation": "create_udf",
  "target": "IncrementLimit",
  "spec": {
    "body": "MIN(x + y, limit)",
    "parameters": [
      {"name": "x", "subtype": "NUMERIC"},
      {"name": "y", "subtype": "NUMERIC", "default_expression": "1"},
      {"name": "limit", "subtype": "NUMERIC", "default_expression": "10"}
    ]
  }
}
```

This creates a signature equivalent to:

```dax
(x : NUMERIC, y : NUMERIC = 1, limit : NUMERIC = 10) => MIN(x + y, limit)
```

Callers can omit trailing optional arguments, for example `IncrementLimit(5)`, or skip an optional argument in the middle with an empty argument position, for example `IncrementLimit(5, , 20)`.

Default expressions can be constants or DAX expressions such as `BLANK()` or `DATE(2026, 1, 1)`. The Power BI engine validates whether the default expression is legal for the parameter type and scope.

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

### Creating with a Full Signature Body

The recommended tool shape is to put parameters in `spec.parameters` and the DAX expression in `spec.body`. If you already have a complete UDF signature from DAX Query View or TMDL, you can provide the signature in `spec.body` without `spec.parameters`:

```json
{
  "operation": "create_udf",
  "target": "AddTax",
  "spec": {
    "body": "(amount : NUMERIC, taxRate : NUMERIC = 0.1) => amount * (1 + taxRate)",
    "description": "Add tax to an amount"
  }
}
```

`FUNCTION AddTax = (...) => ...` and TMDL-style `function AddTax = (...) => ...` inputs are also accepted and normalized to the stored `(...) => ...` expression. Do not also supply `spec.parameters` when `spec.body` already contains a full signature.

### Validation Results

Before creating or renaming a UDF, SemanticOps checks the terminal function-name segment against the connected provider's current reserved keywords and DAX built-in functions. Explicit parameter names are checked against provider keywords. Existing functions whose names now collide remain updateable and deletable. If provider metadata cannot be read, the write continues with the existing syntax checks and the response warns that collision validation was skipped.

`create_udf` and `update_udf` return a top-level `validation` object with `ok`, `state`, and `error_message` from TOM after the write. If Power BI saves the UDF but reports a non-ready state or an error message, the tool returns an error result and includes `saved_udf`. The mutation outcome remains `applied`: inspect the saved object and correct it with a new update; do not repeat the original create or update.

Each UDF mutation makes one commit attempt. SemanticOps never refreshes metadata and replays the write:

- `not_applied` means the commit was not applied and the outcome explicitly says whether retrying is safe.
- `applied` means the write committed, even when validation or read-back reports a problem. Do not repeat the original request.
- `unknown` means the commit response was ambiguous. Inspect the selected model, UDF list, and operation history before deciding on a new action; never retry the original request automatically.

Transactional bulk requests plan, approve, and commit all UDF changes once. Non-transactional batches are approved once, commit each item once, and stop after the first `unknown` outcome.

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

3. **Prefer specific reference types when possible**: Use `MEASUREREF`, `COLUMNREF`, `TABLEREF`, or `CALENDARREF` when the function requires that specific kind of model object. Reserve `ANYREF` for parameters that genuinely accept multiple reference/expression forms.

4. **Document your functions**: Include clear descriptions via the `description` parameter in spec.

5. **Test with edge cases**: Consider BLANK values and empty tables in your function logic.

6. **Keep functions focused**: Each function should have a single, well-defined purpose.

7. **Use variables**: For complex functions, use VAR to break down logic and improve readability.

8. **Hide internal helpers**: Use `is_hidden: true` in spec for utility functions not intended for end users.

9. **Place optional parameters last**: DAX allows required parameters after optional parameters, but that forces callers to use empty argument positions. Prefer making every parameter after the first optional parameter optional as well.

## Parameter Format Reference

```json
{
  "name": "parameterName",
  "type": "ANYVAL | SCALAR | TABLE | ANYREF | MEASUREREF | COLUMNREF | TABLEREF | CALENDARREF",
  "subtype": "NUMERIC | STRING | BOOLEAN | DATETIME | INT64 | DECIMAL | DOUBLE | VARIANT",
  "mode": "VAL | EXPR",
  "default_expression": "DAX expression used when the caller omits this argument"
}
```

- `name`: Required. The parameter name used in the function body.
- `type`: Optional. If omitted, the parameter is untyped (`ANYVAL`) and the engine infers it. Use TABLE for table inputs, ANYREF for flexible expression/reference inputs, and MEASUREREF/COLUMNREF/TABLEREF/CALENDARREF for stricter model-reference inputs.
- `subtype`: Optional. Scalar subtype hint (applies when type is SCALAR, or when type is omitted and you want to hint a scalar subtype).
- `mode`: Optional. Defaults to VAL. Use EXPR for lazy evaluation with context control.
- `default_expression`: Optional. DAX expression used when the caller omits the argument; setting this makes the parameter optional.

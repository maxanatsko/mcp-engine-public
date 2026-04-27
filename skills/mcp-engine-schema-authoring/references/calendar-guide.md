# Calendar Configuration Guide

This guide explains how to configure calendar objects in Power BI using `manage_schema` for time intelligence support.

## Related Tools

- `manage_schema`: Create, update, and delete calendars with time unit mappings (operations: `create_calendar`, `delete_calendar`, `rename_calendar`, `add_time_unit`, `remove_time_unit`, `add_related_group`, `remove_related_group`)
- `list_model`: List existing calendars in the model (`operation: "list"`, `spec: { type: "calendars" }`)

## Concepts

### Calendar Column Groups

Calendar column groups map date table columns to standardized time units (Year, Quarter, Month, Week, Date). This enables time intelligence functions to work correctly and consistently across models.

### Primary vs Associated Columns

- **Primary Column**: The column that uniquely identifies periods for a time unit. When a column maps to a specific unit, make it the `primary_column`.
- **Associated Columns**: Additional columns that have a 1-to-1 relationship with the primary (e.g., month name text sorted by month number). These are optional labels.

**Sorting Rule**: If column A is sorted by column B (Power BI SortByColumn), then B should be the `primary_column` and A should be an `associated_column`.

### Time-Related Groups

Use `add_related_group` for relative columns that are time-aware but don't represent a standard time unit (e.g., RelativeMonth with values like "Current"/"Previous", or Season). These can slice/label time-aware analyses but don't define a standard unit.

## Time Units

### Complete Units (Include Calendar Context)

Complete units uniquely identify a single period and must include the year:

| Time Unit | Example Values | Notes |
|-----------|---------------|-------|
| Year | 2024 | Full year |
| Quarter | "Q3 2024" | Year + quarter |
| Month | "January 2024", "2024-01" | Year + month |
| Week | "2024-W49" | ISO Year-Week format |
| Date | "2024-01-15" | Full date |

### Partial Units (Position Within Period)

Partial units represent positions within a larger period and are not unique by themselves:

| Time Unit | Range | Example |
|-----------|-------|---------|
| QuarterOfYear | 1-4 | 4 (4th quarter) |
| MonthOfYear | 1-12 or names | "January" |
| MonthOfQuarter | 1-3 | 2 (2nd month of quarter) |
| WeekOfYear | 1-52/53 | 49 |
| WeekOfQuarter | varies | 11 |
| WeekOfMonth | 1-5 | 3 |
| DayOfYear | 1-366 | 241 |
| DayOfQuarter | 1-92 | 71 |
| DayOfMonth | 1-31 | 23 |
| DayOfWeek | 1-7 | 4 (Thursday if 1=Monday) |

**Important**: Do not map MonthOfYear to Month, WeekOfYear to Week, or QuarterOfYear to Quarter - these are different concepts.

### ISO 8601 Week Numbering

For ISO 8601 compliance (common in Europe and international contexts):

- Weeks start on **Monday**
- The first week of the year is the week containing **January 4th** (equivalently, the first week with at least 4 days in the new year)
- Week numbers range from 1 to 52 or 53
- Use ISO year-week format: `2025-W01`, `2025-W52`

**Note**: ISO week year may differ from calendar year at year boundaries. For example, December 31, 2024 might be in ISO week 2025-W01.

When implementing ISO weeks:
- Create separate `ISOYear` and `ISOWeekOfYear` columns
- Use `ISOYearWeekNumber` (integer like 202501) as the primary column for the `Week` time unit
- Associate `ISOYearWeek` (text like "2025-W01") as a display column

## Tool Usage

### Creating a Calendar

```json
{
  "operation": "create_calendar",
  "table": "DimDate",
  "target": "Gregorian Calendar"
}
```

Creates an empty calendar. Add time units afterward.

### Creating with Initial Time Unit

```json
{
  "operation": "create_calendar",
  "table": "DimDate",
  "target": "Gregorian Calendar",
  "spec": {
    "time_unit": "Year",
    "primary_column": "Year"
  }
}
```

### Adding Time Units

```json
{
  "operation": "add_time_unit",
  "table": "DimDate",
  "target": "Gregorian Calendar",
  "spec": {
    "time_unit": "Month",
    "primary_column": "YearMonthNumber",
    "associated_columns": ["YearMonth", "YearMonthShort"]
  }
}
```

### Adding Time-Related Groups

For relative or seasonal columns:

```json
{
  "operation": "add_related_group",
  "table": "DimDate",
  "target": "Gregorian Calendar",
  "spec": {
    "columns": ["RelativeMonth", "Season"]
  }
}
```

### Removing Time Units

```json
{
  "operation": "remove_time_unit",
  "table": "DimDate",
  "target": "Gregorian Calendar",
  "spec": {
    "time_unit": "WeekOfYear"
  }
}
```

### Renaming a Calendar

```json
{
  "operation": "rename_calendar",
  "table": "DimDate",
  "target": "Gregorian Calendar",
  "spec": { "new_name": "Standard Calendar" }
}
```

### Deleting a Calendar

```json
{
  "operation": "delete_calendar",
  "table": "DimDate",
  "target": "Gregorian Calendar"
}
```

## Best Practices

1. **Use complete units for hierarchies**: Year -> Quarter -> Month -> Date should all be complete units.

2. **Associate partial units with complete primaries**: You may associate a partial label (MonthOfYear name) with a complete primary (Year Month Number) if it's 1-to-1.

3. **Prefer ISO Year-Week for Week units**: Include year context in week identifiers.

4. **Don't repeat time units**: Each calendar should have at most one mapping per time unit.

5. **Keep columns on the same table**: All columns in a calendar must belong to the host table.

6. **Use strict=false for lenient operations**: When unsure about column validity, use `"strict": false` in spec to skip invalid columns and get feedback on what was accepted/rejected.

## Example: Complete Date Table Calendar

```json
{
  "operation": "create_calendar",
  "table": "DimDate",
  "target": "Gregorian Calendar",
  "spec": {
    "time_unit": "Date",
    "primary_column": "Date"
  }
}
```

Then add additional time units:

```json
{"operation": "add_time_unit", "table": "DimDate", "target": "Gregorian Calendar", "spec": {"time_unit": "Year", "primary_column": "Year"}}
```

```json
{"operation": "add_time_unit", "table": "DimDate", "target": "Gregorian Calendar", "spec": {"time_unit": "Quarter", "primary_column": "YearQuarterNumber", "associated_columns": ["YearQuarter"]}}
```

```json
{"operation": "add_time_unit", "table": "DimDate", "target": "Gregorian Calendar", "spec": {"time_unit": "Month", "primary_column": "YearMonthNumber", "associated_columns": ["YearMonth", "YearMonthShort"]}}
```

```json
{"operation": "add_time_unit", "table": "DimDate", "target": "Gregorian Calendar", "spec": {"time_unit": "MonthOfYear", "primary_column": "MonthNumber", "associated_columns": ["Month", "MonthShort"]}}
```

```json
{"operation": "add_time_unit", "table": "DimDate", "target": "Gregorian Calendar", "spec": {"time_unit": "Week", "primary_column": "ISOYearWeekNumber", "associated_columns": ["ISOYearWeek"]}}
```

## Fiscal Calendar Example

For fiscal calendars with different year boundaries:

```json
{
  "operation": "create_calendar",
  "table": "DimDate",
  "target": "Fiscal Calendar"
}
```

```json
{"operation": "add_time_unit", "table": "DimDate", "target": "Fiscal Calendar", "spec": {"time_unit": "Year", "primary_column": "FiscalYearNumber", "associated_columns": ["FiscalYearName"]}}
```

```json
{"operation": "add_time_unit", "table": "DimDate", "target": "Fiscal Calendar", "spec": {"time_unit": "Month", "primary_column": "FiscalYearMonthNumber", "associated_columns": ["FiscalYearMonth"]}}
```

```json
{"operation": "add_time_unit", "table": "DimDate", "target": "Fiscal Calendar", "spec": {"time_unit": "MonthOfYear", "primary_column": "FiscalMonthNumberOfYear", "associated_columns": ["FiscalMonthName"]}}
```

Then add time-related groups for relative analysis:

```json
{"operation": "add_related_group", "table": "DimDate", "target": "Fiscal Calendar", "spec": {"columns": ["RelativeMonth", "Season"]}}
```

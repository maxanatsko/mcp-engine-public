# AI Artifact Templates

Use these templates for the readiness pack. Keep drafts concrete and model-specific. Replace placeholder names with actual tables, fields, measures, and business terms from the semantic model.

## Readiness pack

```markdown
# AI Readiness Pack: <Model Name>

## Executive Summary
- Readiness level: Ready | Mostly ready | At risk | Not ready
- Score: <0-100>
- Primary risk: <one sentence>
- Recommended next step: <one sentence>

## Scorecard
| Area | Score | Key risks |
| --- | ---: | --- |
| Business terminology and descriptions | <n>/20 | <summary> |
| Metric clarity | <n>/20 | <summary> |
| Date, grain, and filter defaults | <n>/15 | <summary> |
| Field exposure and schema focus | <n>/15 | <summary> |
| Relationships and dependencies | <n>/10 | <summary> |
| AI instruction/data schema drafts | <n>/10 | <summary> |
| Verified answers and tests | <n>/10 | <summary> |

## Findings
<severity-grouped findings>

## MCP-Applicable Model Fixes
<descriptions, names, visibility, display folders, model properties, tests>

## Prep Data for AI Drafts
<AI instructions, AI data schema, verified answers>

## Manual Validation
<Power BI Desktop/service Copilot tests, HCAAT/diagnostics, publish/refresh notes>
```

## AI instructions draft

```markdown
# AI Instructions Draft

## Business Context
- This model is used for <audience> to analyze <domain/process>.
- Prefer business terminology from <source of truth>.
- When users say "<term>", interpret it as <table/field/measure>.

## Preferred Metrics
- For sales/revenue questions, use <measure> unless the user explicitly asks for <alternate measure>.
- For customer count questions, use <measure> and do not count <ambiguous table/field>.
- For margin/profitability, use <measure> and explain whether it is gross, net, or contribution margin.

## Date and Period Defaults
- Use <date role/table> as the default date for trend and period questions.
- Use fiscal calendar with fiscal year starting <month> when users ask about year, quarter, or YTD.
- Use <grain> as the default grain for trend answers unless the user asks otherwise.

## Filters and Segments
- For geography, prefer <field> over <ambiguous alternate>.
- For product/category, prefer <field> and avoid <technical field>.
- Ask a clarifying question when <specific ambiguity> is present.

## Terms to Avoid or Clarify
- Avoid <field/measure> for user-facing answers because <reason>.
- If the user asks for <ambiguous phrase>, ask whether they mean <option A> or <option B>.
```

## AI data schema recommendation

```markdown
# AI Data Schema Recommendation

## Include
| Table | Fields or measures | Reason |
| --- | --- | --- |
| <table> | <field/measure list> | <business purpose> |

## Exclude or Deprioritize
| Table | Fields or measures | Reason |
| --- | --- | --- |
| <table> | <field/measure list> | <ambiguity, technical field, duplicate, helper> |

## Review Notes
- Hidden model fields may appear in setup panes, but the initial schema should prioritize clean, visible, business-facing fields.
- Treat the AI data schema as Copilot guidance, not a security boundary or universal schema filter.
- The schema only applies to Copilot capabilities that use it. Copilot paths such as report summaries, report-page creation, search, and DAX generation can still consider the broader semantic model.
- Validate by asking one question that should be answerable from included fields and one question that should not rely on excluded fields.
```

## Verified-answer candidate

```markdown
## Verified Answer Candidate: <Question Family>

- Business question: <canonical question>
- Trigger phrases:
  - <natural phrase 1>
  - <natural phrase 2>
  - <natural phrase 3>
- Expected visual: <visual type and grain>
- Required measures: <measure list>
- Required dimensions: <dimension list>
- Optional user filters: <up to three report/page/visual filters that already exist without fixed values>
- Persistent filters: <filters baked into visual, if any>
- Validation notes: <expected behavior, caveats, owner approval>
- Apply path: Power BI Desktop or Power BI service verified-answer setup
```

Verified-answer constraints to preserve in candidates:
- Use five to seven representative trigger phrases when possible.
- Available user filters must already exist on the report, page, or visual and cannot have a fixed selected value.
- Slicers are not carried into verified answers as filter options.
- Relative date filters are not supported; use supported date ranges instead.
- Keep within the 10 filter-permutation limit.
- Avoid unsupported visuals, report measures, and hidden fields in verified-answer visuals.
- Do not treat verified answers as a security boundary for RLS or OLS behavior.

## manage_tests suggestion

Use `manage_tests` suggestions only for deterministic model checks. Do not pretend these are Copilot tests; they validate the underlying semantic answer scenario.

```markdown
## Natural-Language Scenario Test Suggestion

- Scenario: <business question>
- Expected interpretation: <measure, date role, filters, grain>
- Suggested DAX validation: <aggregate query or summary of query>
- Tolerance: <exact, range, non-empty, row count, relative comparison>
- Why it matters: <business risk>
- Tool path: draft a `manage_tests` definition after the model author confirms the expected result
```

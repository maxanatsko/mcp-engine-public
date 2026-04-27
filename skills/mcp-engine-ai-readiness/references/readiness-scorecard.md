# Readiness Scorecard

Use this rubric to make findings comparable across semantic models.

## Readiness levels

- 85-100: Ready. The model has clear business language, focused exposure, documented metrics, and validation coverage for common natural-language questions.
- 70-84: Mostly ready. Core questions should work, but several ambiguity or documentation gaps need cleanup before broad rollout.
- 50-69: At risk. Copilot can answer some questions, but users are likely to see wrong metric, date, filter, or grain choices.
- 0-49: Not ready. The model is too ambiguous, technical, undocumented, or under-tested for reliable natural-language use.

## Category weights

- Business terminology and descriptions: 20 points
- Metric clarity and duplicate-measure risk: 20 points
- Date, grain, and filter defaults: 15 points
- Field exposure and schema focus: 15 points
- Relationships, dependencies, and model structure: 10 points
- AI instructions and data schema draft quality: 10 points
- Verified answers and natural-language tests: 10 points

## Severity definitions

- Critical: Likely to produce materially wrong answers for common executive or operational questions.
- High: Likely to confuse Copilot or users for important questions, but the failure is scoped to a domain, metric, or date role.
- Medium: Creates friction, follow-up clarification, or inconsistent wording, but does not usually invert a core answer.
- Low: Polish, maintainability, or adoption issue that improves trust but is not blocking.

## Finding format

Use this shape for every finding:

```markdown
### [Severity] Short finding title

- Area: metric clarity | terminology | date defaults | field exposure | schema focus | verified answers | tests
- Evidence: exact table, field, measure, expression pattern, dependency, or metadata gap
- Impact: how a natural-language user or Copilot answer can fail
- Recommendation: direct action
- Apply path: MCP-applicable | Prep data for AI draft | manual validation | mixed
- Suggested owner: model author | business owner | report author | platform admin
```

## Audit checklist

Business terminology:
- High-value tables and measures have descriptions.
- Names use business language, not source-system abbreviations.
- Synonyms and acronyms are defined in AI instructions.
- Model description explains audience, domain, and primary analytical use.

Metric clarity:
- Certified or preferred measures are obvious.
- Similar metrics explain their differences.
- Deprecated or helper measures are hidden or marked clearly.
- Measures have stable display folders by business process.

Date and grain:
- Preferred date role is explicit.
- Fiscal and calendar behavior is documented.
- Default grain is clear for trend questions.
- Relative periods have consistent definitions.

Schema focus:
- Visible columns are useful for natural-language questions.
- Keys, bridge columns, sort columns, and technical flags are hidden unless user-facing.
- Dimensions expose clean attributes rather than many near-duplicates.
- AI data schema draft prioritizes fields end users should ask about.

Validation:
- Critical questions have expected measure, dimension, filter, and grain.
- Verified-answer candidates cover common curated answers.
- `manage_tests` suggestions cover the questions most likely to regress.

## Scoring method

Start from 100 and subtract:
- Critical finding: 12 to 20 points each
- High finding: 6 to 10 points each
- Medium finding: 2 to 5 points each
- Low finding: 1 point each

Do not let one category consume the entire score. Cap deductions per category at its category weight unless a critical finding makes the model unsafe for broad Copilot use.


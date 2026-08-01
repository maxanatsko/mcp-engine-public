# Semantic Model Quality Scorecard

Use this rubric to make semantic model quality reviews comparable across models. Start from 100 and subtract points for confirmed or strongly evidenced findings.

## Quality Bands

- 90-100: Strong. The model follows strong dimensional modeling, semantic, performance, and governance practices. Remaining work is mostly polish.
- 75-89: Good with risks. The model is usable, but several practices could create maintenance, performance, or user-trust issues.
- 60-74: Needs improvement. Users or authors are likely to encounter ambiguity, slow queries, brittle measures, or confusing field exposure.
- 40-59: High risk. Structural or semantic issues can cause wrong answers, poor performance, or difficult maintenance.
- 0-39: Critical. The model is not safe to broaden without focused redesign or governance remediation.

## Category Weights

- Model shape and star-schema fit: 20 points
- Relationships and filter propagation risk: 15 points
- DAX and semantic layer maintainability: 15 points
- Storage and performance risk: 15 points
- Metadata, naming, descriptions, folders, and field exposure: 15 points
- Governance and security signals: 10 points
- Validation and test coverage: 10 points

## Deduction Guidance

- Critical finding: subtract 12-20 points
- High finding: subtract 6-10 points
- Medium finding: subtract 2-5 points
- Low finding: subtract 1 point

Use the midpoint of the severity range by default (critical 16, high 8, medium 3). Move toward the low or high end only for specific stated evidence, and name that reason in the finding, so repeated assessments of the same model land on the same score.

Do not let one category consume the entire score. Cap ordinary deductions per category at the category weight. You may exceed the category cap only when a critical defect makes broad model use unsafe, and you must explain why.

## Severity Definitions

- Critical: Likely to produce materially wrong answers, bypass expected security, or make the model unsuitable for broad use.
- High: Likely to cause important ambiguity, slow common queries, brittle maintenance, or misleading report-author behavior.
- Medium: Creates avoidable friction, inconsistent interpretation, or future maintenance risk.
- Low: Polish, documentation, naming, or hygiene issue that improves trust but is not blocking.

## Finding Format

Use this shape for every finding:

```markdown
### [Severity] Short finding title

- Area: model shape | relationships | DAX | performance | metadata | governance | validation
- Evidence: exact table, field, measure, relationship, expression pattern, role, perspective, or diagnostic result
- Impact: how the issue can affect users, maintainers, performance, or governance
- Recommendation: direct remediation
- Source: Microsoft Learn or SQLBI link from the rulebook
- Confidence: high | medium | low
```

## Scorecard Output

Use this compact scorecard table:

```markdown
| Category | Score | Key deductions |
| --- | ---: | --- |
| Model shape and star-schema fit | 0-20 | ... |
| Relationships and filter propagation risk | 0-15 | ... |
| DAX and semantic layer maintainability | 0-15 | ... |
| Storage and performance risk | 0-15 | ... |
| Metadata and field exposure | 0-15 | ... |
| Governance and security signals | 0-10 | ... |
| Validation and test coverage | 0-10 | ... |
```

## Remediation Backlog

End with a prioritized backlog:

```markdown
| Priority | Work item | Objects | Expected benefit | Validation |
| --- | --- | --- | --- | --- |
| P0 | ... | ... | ... | ... |
```

Priority rules:

- P0: correctness, security, or severe performance risk
- P1: common author/user confusion or high-maintenance defects
- P2: performance, metadata, and usability improvements with clear value
- P3: polish and cleanup

## Source Citation Rules

- Cite Microsoft Learn for Power BI product guidance and relationship/modeling recommendations.
- Cite SQLBI for expert modeling, VertiPaq, and DAX optimization guidance.
- Do not overstate a source. If a source says "generally" or describes exceptions, preserve that nuance.
- Use citations as support for recommendations, not as a substitute for model-specific evidence.
- If a finding is based on local convention rather than an external source, label it as "local convention" and keep the severity modest unless the impact is proven.

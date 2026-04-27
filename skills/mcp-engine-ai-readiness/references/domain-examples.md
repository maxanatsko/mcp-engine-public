# Domain Examples

Use these as starting patterns. Replace every example with model-specific names before returning final artifacts.

## Sales

Common readiness risks:
- Multiple revenue measures with unclear preference.
- Order date, ship date, invoice date, and close date all visible.
- Product hierarchy has both source codes and business categories.
- Customer, account, partner, and reseller terms overlap.

AI instruction seeds:
- For revenue questions, use the certified net sales measure unless the user asks for gross sales.
- Use order date for sales performance trends and invoice date for billing questions.
- Interpret "top customers" by net sales unless the user asks for order count or margin.

Verified-answer candidates:
- Sales trend by month.
- Top customers by net sales.
- Product category performance by quarter.
- Pipeline conversion by stage, if the model includes sales pipeline facts.

## Finance

Common readiness risks:
- Actuals, budget, forecast, and variance measures are not clearly separated.
- Fiscal calendar behavior is undocumented.
- Account hierarchy exposes technical ledger levels.
- Sign conventions for expenses, revenue, and variance are unclear.

AI instruction seeds:
- Use fiscal calendar for year, quarter, YTD, and prior-year comparisons.
- When users ask for variance, return actual minus budget unless the model owner defines another convention.
- Clarify whether the user wants actuals, budget, forecast, or variance when the prompt is ambiguous.

Verified-answer candidates:
- P&L summary by fiscal period.
- Budget versus actual variance by cost center.
- Forecast accuracy by month.
- Expense trend by department.

## Support and Customer Success

Common readiness risks:
- Ticket, case, incident, and request terminology is inconsistent.
- Resolution time and response time measures have unclear business definitions.
- Customer health fields are visible without scoring context.
- Status fields include too many operational states.

AI instruction seeds:
- Use resolved date for resolution trend questions and created date for intake trend questions.
- Treat "backlog" as open tickets excluding closed and canceled statuses.
- Clarify whether "response time" means first response time or total resolution time.

Verified-answer candidates:
- Open backlog by priority.
- Average resolution time by product.
- SLA attainment by customer segment.
- Ticket intake trend by week.

## Operations

Common readiness risks:
- Facility, region, site, and location fields overlap.
- Throughput, capacity, utilization, and productivity measures are not clearly distinguished.
- Date roles for planned, scheduled, completed, and shipped events are ambiguous.
- Technical equipment or batch identifiers are exposed as common fields.

AI instruction seeds:
- Use completed date for throughput and shipped date for fulfillment questions.
- Interpret utilization as actual output divided by available capacity unless the user asks for labor utilization.
- Use site as the default location grain unless the user asks for facility or region.

Verified-answer candidates:
- Throughput trend by site.
- Capacity utilization by week.
- On-time completion rate by work center.
- Backlog by priority and due date.


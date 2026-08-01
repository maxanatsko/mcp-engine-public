# Semantic Model Quality Assessment Workflow

Use this workflow to assess a connected Power BI semantic model for bad or questionable modeling practices. The workflow is assess-only: gather evidence, score risks, and recommend remediation. Do not apply model changes from this workflow.

## Related Tools and Resources

- Connection context: `manage_model_connection`
- Model metadata: `list_model`
- Dependencies and impact: `manage_dependencies`
- Validation and diagnostics: `run_query` (`execute`, `analyze`, `vertipaq`, `test_access` when available)
- Test recommendations: `manage_tests`
- Related resources:
  - `model-quality-scorecard.md`
  - `model-quality-rulebook.md`
  - `modeling-best-practices-guide.md (from the mcp-engine-semantic-authoring skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`
  - `relationships-guide.md (from the mcp-engine-schema-authoring skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`
  - `measure-authoring-guide.md (from the mcp-engine-semantic-authoring skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`
  - `semantic-layer-style-guide.md (from the mcp-engine-semantic-authoring skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`
  - `security-roles-guide.md (from the mcp-engine-security-governance skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`
  - `perspectives-guide.md (from the mcp-engine-security-governance skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`
  - `vertipaq-optimization-guide.md (from the mcp-engine-dax-performance skill; if not installed, rely on the tool's inputSchema or ask the user to add that skill)`

## Operating Rules

- Stay read-only. Do not call authoring operations from `manage_schema`, `manage_semantic`, `manage_model_properties`, `manage_security`, `manage_policy`, or other write tools.
- Prefer metadata evidence before querying data.
- Use `run_query` only for small aggregated validation, DAX performance analysis, VertiPaq/storage diagnostics, or access tests.
- Never dump full tables or raw sensitive rows. Keep results aggregated, masked, and capped.
- Treat unsupported tool operations, browse-only mode, policy denials, missing licenses, and unavailable model connections as scope limitations, not as model defects.
- Cite the bundled source rule behind each material finding.

## Assessment Sequence

### 1) Establish Scope and Connection

1. If model context is unclear, use `manage_model_connection` to get the current connection or list/select the intended model.
2. Record model name, source type when available, mode limitations, and whether query/performance tools are available.
3. Ask for business context only when the model metadata cannot establish the intended domain or audience.

### 2) Inventory the Model

Gather a compact inventory with `list_model`:

- model properties
- tables
- columns
- measures
- relationships
- hierarchies
- calculation groups and calculation items
- partitions and named expressions
- roles and object-level security
- perspectives
- cultures/translations

If responses are large, summarize first and drill into suspicious tables, relationships, or expressions.

### 3) Assess Model Shape

Use tables, columns, relationships, and names to infer facts, dimensions, bridges, date tables, and technical helper objects.

Look for:

- flat or single-table structures where facts and dimensions appear mixed
- snowflaked dimensions that increase relationship chains without clear benefit
- facts loaded at inconsistent grain
- missing date table or unclear role-playing dates
- visible technical keys, sort columns, bridge keys, or helper fields

### 4) Assess Relationships

Use `list_model` relationship metadata and targeted table expansion.

Look for:

- bi-directional filtering outside known special cases
- many-to-many relationships without clear bridge design
- inactive relationships that may indicate role-playing dimension ambiguity
- ambiguous or cyclic filter paths
- missing, disabled, or suspicious relationships between likely facts and dimensions

When a relationship risk affects DAX, use expression search for patterns such as `USERELATIONSHIP`, `TREATAS`, `CROSSFILTER`, and `RELATED`.

### 5) Assess DAX and Semantic Layer

Use measure and calculation item metadata, descriptions, folders, dependencies, and expression search.

Look for:

- duplicate or overlapping measures with unclear business definitions
- helper measures visible to report authors
- complex repeated DAX patterns that should be centralized
- calculated columns used where a measure or source transformation would be more appropriate
- missing display folders or descriptions for business-facing measures
- inconsistent naming, abbreviations, and metric suffixes

Do not rewrite DAX during assessment. Recommend a remediation pattern and identify the objects to review.

### 6) Assess Storage and Performance Risk

Use `run_query` with `operation: "vertipaq"` when available. If unavailable, infer risk from metadata only.

Look for:

- high-cardinality text, GUID, transaction ID, URL, or timestamp columns on large tables
- unnecessary imported columns
- visible or imported fields that are not used for grouping, filtering, summarization, security, sorting, or relationships
- calculated columns on large tables
- query patterns that may be slow because model shape forces complex DAX

Use `run_query` with `operation: "analyze"` only for representative aggregate queries. Run small tests and summarize timing/plan evidence rather than returning raw trace dumps.

### 7) Assess Metadata and Curation

Use model properties, table/column/measure metadata, perspectives, cultures, and descriptions.

Look for:

- missing model/table/measure descriptions on business-facing objects
- source-system abbreviations without descriptions
- inconsistent display folders
- visible technical fields
- missing or confusing perspectives for user-facing subsets
- translations/cultures present but incomplete or inconsistent

### 8) Assess Governance Signals

Use roles, OLS metadata, perspectives, masking annotations if visible, and optional access tests.

Look for:

- reliance on hidden fields or perspectives as if they were security
- roles with broad or unclear filters
- sensitive columns that are visible and not governed by RLS/OLS or documented masking intent
- missing validation suggestions for important roles

If the user requests role validation and tools allow it, use `run_query` with small aggregate access tests only.

### 9) Score and Recommend

Use `model-quality-scorecard.md` to calculate the score and produce the final scorecard. Use `model-quality-rulebook.md` for source-backed finding language.

Final output order:

1. Executive score and quality band.
2. Top 3 risks.
3. Scorecard by category.
4. Findings grouped by severity.
5. Prioritized remediation backlog.
6. Validation/test recommendations.
7. Source notes.

## Evidence Standards

Each material finding must include:

- exact model object evidence, such as table, column, relationship, measure, role, perspective, expression pattern, or VertiPaq result
- why it matters
- recommended remediation
- source links from Microsoft Learn or SQLBI where applicable
- confidence level when evidence is inferential rather than directly proven

Use cautious wording for inferred issues. For example, "likely fact table" or "appears to be a technical key" is acceptable when based on naming, relationships, and data type alone.

# Copilot Investigation Prompt: QueryMate DAB Architecture, MCP Contract, Security, and Scale Readiness

You are acting as a senior Microsoft Data API Builder (DAB), SQL Server, Model Context Protocol (MCP), and AI-platform architect. The `querymate-dab` application is open in this workspace.

Perform a complete, evidence-first investigation of the DAB application. This is an **investigation and planning task**, not an implementation task. Inspect the repository, run only safe read-only checks, and produce the requested sanitized investigation documents. Do not change application behavior, DAB configuration, database scripts, security rules, or runtime code.

## Why this investigation is needed

QueryMate has two separately deployable applications:

- `querymate-api`: the Spring Boot/Spring AI application that owns the user conversation, system prompt, LLM orchestration, tool selection policy, response formatting, token diagnostics, and any future schema-RAG orchestration.
- `querymate-dab`: the independently runnable DAB application that exposes approved database capabilities through MCP and possibly other explicitly enabled protocols.

The DAB proof of concept currently appears to expose approximately three entities. It may later expose 40 or more entities. QueryMate currently expects a small generic MCP surface such as `describe_entities`, `read_records`, and `aggregate_records`, but these names, descriptions, schemas, and behavior are hypotheses until verified from this repository and a real `tools/list` response.

The investigation must determine:

1. What DAB actually exposes and enforces today.
2. What information and payload size reach the LLM through the MCP contract and tool results.
3. Which current QueryMate system-prompt rules duplicate DAB's contract or runtime validation.
4. Which query-planning responsibilities belong in the prompt, in QueryMate Java orchestration, in DAB configuration/MCP, or in SQL Server.
5. Whether future schema RAG is justified at 40+ entities, what authoritative metadata it would need, and what must remain outside DAB.
6. What should be optimized in each application without breaking current functionality.

## Non-negotiable safety and scope rules

- Start by reading any `AGENTS.md`, `README`, repository instructions, and existing architecture or analysis documents.
- Inspect before concluding. Do not infer DAB features, defaults, or version-specific syntax from memory.
- Determine and record the exact DAB version, packaging method, startup method, SQL Server version if discoverable safely, and MCP transport.
- Do not modify `dab-config.json`, environment files, SQL files, scripts, Docker files, pipeline files, or application source.
- The only permitted file changes are the investigation documents requested under **Deliverables**. If repository instructions prohibit even documentation changes, stop and report that constraint.
- Do not commit, push, create a branch, or open a pull request.
- Do not install packages or global tools. Prefer existing scripts and already-installed tooling. If MCP Inspector or another client is already present, it may be used; otherwise record the exact missing prerequisite instead of installing it.
- Do not start services unless the repository already documents a safe local startup command and the required local/test configuration is available. Never connect to an unknown, shared, staging, or production database merely because a connection string is present.
- Do not execute create, update, delete, DDL, stored-procedure, arbitrary-SQL, or other mutating operations through SQL, MCP, REST, or GraphQL.
- Do not run a negative write test against any database unless the repository already contains an explicitly documented disposable local test environment and an existing security-check script designed for that purpose. Static inspection is sufficient otherwise.
- Never print or copy connection strings, passwords, tokens, API keys, usernames, hostnames that are classified as sensitive, or real database records into reports. Preserve environment-variable names while replacing values with `<redacted>`.
- Sanitize captured requests and responses. Use field names and synthetic identifiers where useful; omit or mask actual business values.
- Do not log or report hidden model reasoning. This task should not require an LLM call.
- If a step needs credentials, elevated access, a network connection, a mutating action, or an ambiguous environment choice, skip that step, record the blocker and exact safe next action, and continue all other stages.
- Do not propose moving a responsibility merely to reduce prompt length. Correctness, security, maintainability, and evidence come first.

## Evidence standard

Label every material finding with one of these statuses:

- **Confirmed — configuration:** directly supported by a repository file, with file path and relevant section/property.
- **Confirmed — runtime:** observed in a safe command or MCP exchange, with the sanitized command and result.
- **Inferred:** a reasoned conclusion from confirmed evidence; state the reasoning and uncertainty.
- **Unknown:** not available or not safely testable; state what evidence would resolve it.

For every command executed, record:

- purpose;
- sanitized command;
- working directory;
- exit code;
- concise sanitized outcome;
- whether it changed any state.

Do not treat a successful configuration parse as proof of runtime authorization. Do not treat documentation as proof that this repository uses a feature. Do not treat a failed request as "no matching data" when it is actually a validation, authorization, connectivity, or server error.

## Operating method

Work through Stages 0–9 in order. Do not pause between stages unless a safety rule requires user input. If runtime testing is unavailable, finish the static stages and clearly mark runtime-dependent conclusions as unknown.

Maintain a small evidence ledger while working. Do not create empty placeholder deliverables. At the end, verify with `git status` and `git diff --stat` that only permitted investigation documents changed.

---

## Stage 0 — Establish the repository and runtime baseline

### Inspect

- Repository root and complete relevant file tree.
- `README.md` and all DAB-specific documentation.
- `dab-config.json` and any environment-specific or child configuration files.
- `.env.example` or equivalent placeholder configuration without reading secret values from real environment files.
- Dockerfile, Compose files, launch profiles, package manifests, and CI workflows if present.
- Existing MCP smoke-test scripts and security-check scripts.
- Database schema, seed, read-only security, and verification scripts.
- Any schema-analysis, metadata-generation, or QueryMate integration documents.
- Current Git status, branch, and pre-existing uncommitted changes.

### Determine

- Whether this is standard Microsoft DAB, a wrapper around it, a fork, or a mixed application.
- Exact pinned or resolved DAB version. If only `latest` or an unpinned reference is used, report that as a risk.
- Supported startup paths: CLI, container, Compose, or another host.
- The intended local/test environment and whether it is safe to run.
- Configured database provider and protocol surface: MCP, REST, GraphQL, health, telemetry.
- MCP transport and endpoint/process model.

### Stage output

Add a baseline table with `Item`, `Observed value`, `Evidence`, `Status`, and `Risk/unknown` columns.

Do not run the application yet.

---

## Stage 1 — Inventory the DAB configuration and approved exposure

Inspect the effective DAB configuration without changing it. If an existing safe command can show or validate effective configuration or permissions, use it only after confirming the exact installed-version syntax.

### Capture

- Top-level data sources and environment-variable references.
- Runtime protocol enablement and authentication provider.
- Global pagination/result limits, timeouts, health, logging, telemetry, and error-detail settings.
- Every configured entity and its physical source object type: table, view, or stored procedure.
- Entity description and per-field name, alias, description, primary-key marker, inclusion/exclusion, and protocol exposure.
- Entity permissions by role and action, including inherited or effective permissions when supported by the actual version.
- Relationships, source/target fields, direction, cardinality, and linking object where applicable.
- Entity-level MCP configuration and custom tools if present.
- Any automatically exposed objects or broad patterns that could unintentionally expand the surface.

### Produce these matrices

1. **Entity exposure matrix**
   - DAB entity name
   - physical source
   - primary/canonical key
   - enabled protocols
   - permitted roles/actions
   - exposed and excluded fields
   - relationship count
   - description quality
   - evidence

2. **Relationship matrix**
   - relationship name
   - source entity/fields
   - target entity/fields
   - cardinality/direction
   - database FK evidence
   - DAB configuration evidence
   - ambiguity or missing metadata

3. **Runtime control matrix**
   - control
   - configured value
   - version-specific default only when confirmed from matching official documentation
   - observed runtime behavior if tested
   - gap or risk

Run the repository's existing validation command if it is safe and needs no secrets. If it requires a database connection or unavailable tooling, record the blocker; do not alter configuration to make it pass.

---

## Stage 2 — Capture the actual MCP tool contract

The raw MCP contract is essential because tool definitions are repeatedly supplied to the model by QueryMate. Do not rely solely on README examples or remembered DAB behavior.

### Preferred evidence order

1. Existing repository MCP smoke script.
2. Existing checked-in MCP client configuration or captured sanitized contract.
3. An already-installed MCP Inspector/client using the repository's documented local/test startup.
4. Static DAB version documentation, clearly labeled as documentation rather than observed runtime behavior.

### Capture the raw `tools/list` response when safely possible

For every exposed tool, record exactly:

- tool name;
- tool description;
- input JSON Schema;
- required and optional arguments;
- types, enums, defaults, limits, and examples;
- output schema when supplied;
- annotations or capability metadata;
- character count and UTF-8 byte count of the complete serialized tool definition.

Calculate totals for the entire `tools/list` response. Preserve a sanitized, valid JSON copy in the requested contract deliverable. Do not manually reformat the JSON before measuring; state whether counts use compact or original serialization.

Specifically verify whether tools equivalent to these exist:

- `describe_entities`
- `read_records`
- `aggregate_records`

These names are expected candidates, not guaranteed facts. Report additional tools, missing tools, and any create/update/delete/execute/arbitrary-query capability as high-visibility findings.

### Duplication analysis

For each rule encoded by the tool description or JSON Schema, identify whether the QueryMate system prompt may be repeating it, including:

- allowed filter operators and syntax;
- quoting/escaping rules;
- supported aggregation functions;
- field selection behavior;
- pagination and maximum page size;
- sorting/grouping syntax;
- required entity identifiers;
- invalid-field/error behavior;
- relationship or join limitations.

Do not recommend deleting a prompt rule until Stage 3 proves how DAB behaves when the model violates it.

### Token measurement rule

Report exact characters and UTF-8 bytes. Report token counts only if an already-available tokenizer exactly matches QueryMate's deployed model tokenizer. Otherwise label any token figure as an approximation and keep it separate from exact provider-reported usage. Never present characters divided by four as an exact token count.

---

## Stage 3 — Run safe MCP behavioral and payload tests

Run this stage only against a confirmed disposable local/test database using already-documented startup and smoke-test commands. If that cannot be established, mark the stage blocked and do not improvise a connection.

Prefer existing scripts. Do not rewrite them during this investigation.

### Successful read-only cases

Execute the smallest safe representative calls supported by the discovered contract:

1. List or describe all available entities.
2. Describe one selected entity.
3. Read one entity with explicit field projection and the smallest permitted page size.
4. Read using one simple valid filter.
5. Aggregate with a basic count.
6. If safely supported, group by one low-sensitivity synthetic/reference field.

### Controlled invalid read-only cases

Use obviously nonexistent synthetic names and never include injection payloads against an unknown environment:

1. Unknown entity.
2. Unknown field.
3. Unsupported filter operator or function.
4. Malformed filter syntax.
5. Page size one above the configured maximum.
6. Invalid aggregation field or function.
7. Unauthorized or excluded field only if the local test identity and synthetic dataset make this safe.

### Capture for every exchange

- sanitized request;
- sanitized response or error;
- MCP/HTTP status and error category where available;
- latency if the existing client reports it;
- character and UTF-8 byte counts for request and response;
- row count and field count, without preserving real values;
- pagination envelope and continuation metadata;
- whether errors expose permitted entity/field names, internals, stack traces, SQL, or sensitive details;
- whether the error is sufficiently precise for QueryMate to correct and retry safely;
- whether behavior matches the tool description.

### Payload-shape review

Identify avoidable payload overhead such as repeated field names, duplicated metadata, verbose wrappers, null-heavy records, full entity catalogs returned for a narrow request, or excessively verbose errors. Separate protocol-required structure from optional verbosity.

Do not optimize or alter payloads in this task.

---

## Stage 4 — Assess schema, relationships, and query-planning support

Inspect the database schema scripts and DAB relationships together. The goal is to learn what a planner can know, not to generate or execute SQL.

### Build an entity knowledge inventory

For every entity, document:

- canonical key and any user-facing business identifier;
- display/name/description fields;
- lookup or reference-table role;
- status/code fields and where their meanings come from;
- date/time fields and their business meaning when documented;
- foreign keys and configured DAB relationships;
- relationship direction and cardinality;
- fields usable for filters, grouping, aggregation, or sorting under current exposure rules;
- sensitive or internal-only fields that must never enter schema RAG;
- aliases, descriptions, synonyms, and example business phrases when already documented;
- unresolved semantic gaps.

### Evaluate representative planning questions

Using metadata only, not real row data, assess whether the current DAB contract gives QueryMate enough information to plan:

- direct record lookup by canonical ID;
- lookup by a descriptive label that first requires resolving an ID;
- count of business objects versus count of relationship rows;
- aggregation by status or reference entity;
- a request spanning two related entities;
- a request needing a relationship that is present in SQL but absent from DAB;
- a request with ambiguous business terminology.

For each scenario, state:

- metadata required;
- metadata currently available;
- which tool calls would be needed;
- failure or ambiguity risk;
- correct owning layer for any fix.

Do not claim that DAB performs joins for MCP unless the observed version and contract explicitly prove it. Distinguish GraphQL relationship behavior from MCP tool behavior.

---

## Stage 5 — Verify the read-only security model

Inspect DAB authorization and SQL Server security as defense in depth.

### Verify statically

- DAB uses environment references or an approved secret provider, not committed credentials.
- The intended DAB database identity is dedicated and least privilege.
- SQL grants allow only the required reads and do not grant broad writer/owner/DDL roles.
- DAB entity permissions expose only approved `read` behavior to the intended roles.
- Create, update, delete, execute, and arbitrary SQL capabilities are absent from the exposed MCP surface.
- Excluded entities and fields cannot be selected, filtered, sorted, aggregated, or traversed through relationships under the intended role.
- Authentication mode is appropriate for local versus shared environments.
- Error, health, log, and telemetry settings avoid exposing secrets and records.
- Scripts meant to validate denied writes are clearly scoped to a disposable environment.

### Runtime evidence

Use existing safe security verification scripts only when their target is confirmed disposable and their behavior is already documented. Never conduct ad hoc destructive security testing.

Create a security table with `Control`, `Configuration evidence`, `Runtime evidence`, `Result`, `Risk`, and `Recommended owner`.

Classify a control as **not runtime verified** rather than passing it from static evidence alone.

---

## Stage 6 — Model growth from approximately 3 entities to 40+

Do not assume every cost grows linearly. Measure and separate these components:

1. MCP `tools/list` contract size.
2. `describe_entities` result size for all entities.
3. `describe_entities` result size for one or a selected subset, if supported.
4. Read-result size per row and per projected field.
5. Aggregate-result size.
6. Relationship metadata size and graph density.
7. Business glossary content that DAB does not contain.

### Scaling analysis

- Determine whether generic tool schemas remain nearly constant as entity count grows or embed the entity catalog directly.
- Determine which payloads actually grow with entity and field count.
- From the current measured data, calculate transparent lower/base/upper projections for 40 entities. Show formulas and assumptions.
- Do not fabricate future entities or claim a precise token total. Use ranges and sensitivity analysis.
- Model at least sparse, moderate, and relationship-dense schema cases.
- Identify the point at which asking DAB for the full entity catalog on every conversation becomes wasteful.
- Identify whether selective entity description is supported today. If not, record it as a potential DAB/API capability gap, not an instruction to implement custom behavior.
- Separate input-token cost, tool-result cost, database execution cost, latency, and operational complexity.

### Schema-RAG readiness

Assess whether QueryMate should later build a reviewed schema/business-metadata index. Do not implement RAG.

Define candidate document types:

- compact entity cards;
- field cards only where field-level retrieval is justified;
- relationship cards or paths;
- lookup/reference semantics;
- business glossary/synonym cards;
- status/code meaning cards;
- security/domain tags that prevent retrieval of excluded metadata.

For each candidate type, identify:

- authoritative source;
- owner and review process;
- stable identifier;
- version/schema fingerprint;
- security classification;
- refresh trigger;
- fields safe to embed;
- fields that must stay out of the vector store;
- how retrieved metadata would be verified against current DAB configuration before tool execution.

Explicitly answer:

- Is RAG unnecessary at the current 3-entity scale?
- What measured threshold or symptom should trigger a RAG proof of concept?
- Should DAB generate candidate technical metadata while humans approve business meaning?
- How can QueryMate avoid retrieving irrelevant tables while retaining relationship paths needed for a query?

---

## Stage 7 — Assign query-planning and enforcement responsibilities

Create a responsibility matrix with these possible owners:

- QueryMate system prompt
- QueryMate Java/Spring AI orchestration
- DAB configuration and built-in MCP contract
- SQL Server schema/security
- human-reviewed business metadata or future schema-RAG index

For every rule or behavior, choose one **primary owner** and list secondary enforcement only when it provides useful defense in depth. Avoid maintaining the same fast-changing detail independently in multiple layers.

At minimum classify:

- user intent interpretation;
- ambiguity and clarification decisions;
- tool choice: describe, read, or aggregate;
- entity candidate retrieval;
- relationship-path selection;
- lookup-label to canonical-ID resolution;
- distinguishing business-object counts from relationship-row counts;
- exact entity/field/operator validation;
- filter grammar and escaping;
- pagination and result caps;
- retry budgets and repeated-invalid-call prevention;
- read-only enforcement;
- row/field authorization;
- response-category JSON and presentation formatting;
- provider token diagnostics;
- schema metadata retrieval and caching;
- schema-RAG indexing, versioning, and authorization.

### Query-planning recommendation

Recommend a future high-level planning flow without implementing it. A suitable shape may be:

1. Interpret the business request and determine whether clarification is required.
2. Retrieve only candidate entities and relationship metadata.
3. Build a compact structured plan in QueryMate memory, not user-visible output.
4. Validate entity, field, operator, page, and aggregate choices against authoritative metadata.
5. Resolve descriptive labels to canonical identifiers when needed.
6. Execute the minimum read/aggregate calls with bounded results.
7. Correct only recoverable validation errors within a strict retry budget.
8. Ground the final answer only in successful tool evidence.

Verify each step against actual DAB capabilities. Mark steps owned by QueryMate rather than pretending DAB supplies a full semantic planner.

### Prompt-rule disposition

For each major rule in the current QueryMate prompt, assign one disposition:

- **Keep in prompt:** semantic judgment or user-facing behavior the model must follow.
- **Shorten in prompt:** necessary model instruction already partly explained by tool schema.
- **Move to QueryMate Java:** deterministic orchestration, validation, caching, retry, or formatting that should not depend on model compliance.
- **Enforce in DAB/SQL:** authorization, valid surface, read-only access, limits, or schema truth.
- **Defer pending evidence:** behavior not yet proven.

Do not edit the QueryMate prompt or either application in this task.

---

## Stage 8 — Produce a phased recommendation backlog

Recommendations must be evidence-backed, reversible where possible, and assigned to the correct repository.

Group recommendations into:

### A. No-change findings

Behaviors already correct and worth preserving.

### B. QueryMate API changes for later consideration

Examples may include dynamic metadata retrieval, schema-RAG orchestration, compact plan state, deterministic prevalidation, retry budgets, caching, or prompt reduction. Recommend only what the evidence supports.

### C. DAB changes for later consideration

Examples may include better reviewed descriptions, tighter permissions, explicit limits, safer errors, selective metadata capability, or smaller result envelopes. Do not recommend custom DAB code when built-in configuration is sufficient.

### D. SQL Server changes for later consideration

Examples may include missing foreign keys, read-only views, indexes justified by measured queries, or narrower grants. Do not recommend schema changes based only on speculation.

### E. Human metadata/governance work

Examples may include business synonyms, status meanings, table ownership, sensitivity classification, and entity approval.

### Prioritization

For every recommendation include:

- problem and evidence;
- target owner/repository;
- expected correctness, security, token, latency, or maintainability impact;
- dependency;
- risk and rollback approach;
- effort: small, medium, or large;
- recommended phase: now with 3 entities, before 10 entities, before 40 entities, or production hardening;
- validation/acceptance criterion.

Do not describe token savings as guaranteed until measured with QueryMate's provider-reported per-round usage.

---

## Stage 9 — Create deliverables and perform final verification

Create only these documents, adapting the path only if repository instructions define an existing documentation convention:

1. `docs/dab-investigation/querymate-dab-investigation-report.md`
2. `docs/dab-investigation/mcp-tools-list.sanitized.json` — only when a real `tools/list` response was safely captured
3. `docs/dab-investigation/mcp-test-evidence.sanitized.md` — only when runtime tests were safely executed

Do not create empty files for blocked runtime evidence.

### Required investigation report structure

1. Executive summary
2. Scope, environment, and safety boundaries
3. Evidence-status legend
4. Repository/runtime baseline
5. DAB configuration and entity exposure inventory
6. Relationship inventory
7. Actual MCP tool contract and exact size measurements
8. Safe behavioral-test results and payload analysis
9. Read-only security assessment
10. Query-planning capability and gaps
11. 3-to-40+ scaling model with formulas and assumptions
12. Schema-RAG readiness and trigger criteria
13. Responsibility/ownership matrix
14. QueryMate prompt-rule disposition table
15. Phased recommendation backlog
16. Blockers, unknowns, and evidence needed
17. Commands executed and sanitized results
18. Files changed

### Final checks

- Re-run safe configuration validation if it was available initially.
- Validate that the sanitized JSON is syntactically valid.
- Search the new documents for secret-like values, connection strings, tokens, passwords, and real row data.
- Review `git status`, `git diff --stat`, and the full documentation diff.
- Confirm that no application/configuration/database/security file changed.
- Do not commit or push.

## Acceptance criteria

The investigation is complete when:

- actual repository evidence is clearly separated from assumptions and official documentation;
- the exact DAB version and effective protocol/configuration surface are identified or explicitly marked unknown;
- every configured entity, relationship, permission, and relevant runtime limit is inventoried;
- the real MCP contract is captured and measured when safely possible;
- successful and invalid read-only behaviors are tested when a confirmed safe local environment exists;
- no secret, credential, raw production record, or sensitive endpoint appears in the deliverables;
- growth from 3 to 40+ entities is modeled by component with formulas and ranges, not one unsupported linear estimate;
- schema RAG is evaluated as a future QueryMate capability with authoritative-source, review, security, versioning, and refresh requirements;
- query-planning responsibilities are assigned deliberately across prompt, QueryMate Java, DAB, SQL Server, and reviewed business metadata;
- recommendations identify target repository, phase, dependency, risk, and validation method;
- no runtime code, DAB config, SQL, security, scripts, or deployment files are modified;
- Git diff proves that only the permitted investigation documents changed.

## Required final response in Copilot chat

After finishing, reply with:

1. A concise executive summary of what was confirmed.
2. The investigation files created.
3. Commands/tests run and whether each passed, failed, or was skipped.
4. The most important DAB finding for current QueryMate behavior.
5. The most important scaling finding for 40+ entities.
6. Whether schema RAG is recommended now, later, or not yet, and the evidence-based trigger.
7. The recommended ownership split for query planning.
8. All blockers and unknowns.
9. A Git diff summary proving no application behavior was changed.

Do not implement any recommendation until the investigation is reviewed and the user explicitly approves a separate implementation phase.

## Authoritative references to verify against the discovered DAB version

- Data API Builder documentation: <https://learn.microsoft.com/en-us/azure/data-api-builder/>
- DAB configuration schema: <https://learn.microsoft.com/en-us/azure/data-api-builder/configuration/>
- DAB entity configuration: <https://learn.microsoft.com/en-us/azure/data-api-builder/configuration/entities>
- SQL MCP server overview: <https://learn.microsoft.com/en-us/azure/data-api-builder/mcp/overview>
- SQL MCP built-in data tools: <https://learn.microsoft.com/en-us/azure/data-api-builder/mcp/data-manipulation-language-tools>
- DAB security and authentication: <https://learn.microsoft.com/en-us/azure/data-api-builder/concept/security/>
- MCP Inspector: <https://modelcontextprotocol.io/docs/tools/inspector>

Use official documentation matching the repository's exact DAB version. When current documentation differs from the installed version, the installed version and observed behavior control the investigation, and the version gap must be reported.

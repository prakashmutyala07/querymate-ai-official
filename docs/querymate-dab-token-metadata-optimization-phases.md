# QueryMate DAB Token and Metadata Optimization Plan

Status: Proposed for phased review and execution

Application: `querymate-dab`

Scope: Microsoft Data API Builder configuration, MCP contract, approved entity metadata, payload behavior, read-only permissions, scale readiness, and metadata handoff to QueryMate. This document does not implement changes.

Companion document: `querymate-api-token-optimization-phases.md`

## 1. Objective

Keep DAB an independently runnable, read-only and authoritative data-access layer while reducing avoidable metadata/tool-result verbosity and preparing approved metadata for QueryMate's future 40+ entity scale.

DAB is not responsible for the user conversation, LLM orchestration, final response formatting, or a full semantic query planner. DAB must continue to enforce the real exposed surface regardless of what the model requests.

## 2. Confirmed current configuration

The available configuration and runtime evidence establish:

- Database provider: SQL Server (`mssql`).
- Connection string comes from an environment variable.
- REST is disabled.
- GraphQL is disabled.
- MCP is enabled at `/mcp`.
- Enabled generic tools:
  - `describe_entities`
  - `read_records`
  - `aggregate_records`
- Disabled tools:
  - create
  - update
  - delete
  - execute
- Maximum and default page size: 10.
- Current entities:
  - `BookingLocation`
  - `Counterparty`
  - `Relation`
- Current POC authentication provider: Simulator.
- Current permissions include read access for anonymous and authenticated roles.
- Entity health checks are enabled.
- Entity and field descriptions contain schema facts, business meaning, logical relationships, lifecycle defaults, and query-planning guidance.

Observed MCP payloads:

| Operation | Response bytes |
|---|---:|
| `tools/list` | 5,049 |
| Full entity description | 9,778 |
| Name-only entity description | 1,977 |
| Selected subset description | 2,971 |
| One projected record | 576 |
| One unprojected record | 1,010 |
| Ten unprojected records | 8,245 |
| Count aggregate | 369 |
| Grouped aggregate | 816 |
| Distinct count aggregate | 373 |

Invalid and disabled operations returned HTTP 200 with tool-level `isError=true` and structured error information.

## 3. DAB ownership boundaries

### DAB owns

- Entity and field exposure.
- Source-object mappings.
- Tool enablement.
- Role/action permissions.
- Filter, projection, sort, pagination, and aggregation validation/execution.
- Response limits and timeouts.
- Technical schema metadata exposed through MCP.
- Safe errors and health behavior.

### DAB does not own

- User intent interpretation.
- Conversation history.
- Final answer generation.
- Response JSON category/Markdown formatting.
- LLM retry orchestration.
- Selection of schema context for each question.
- Business glossary retrieval across 40+ entities.
- A general cross-entity semantic planner.

### SQL Server owns

- Physical keys and constraints.
- Read-only database authorization.
- Views/indexes and database performance.

### Human-reviewed metadata owns

- Business synonyms.
- Status/code meaning.
- Canonical business identifiers.
- Logical relationships not enforced by physical foreign keys.
- Counting semantics and special lifecycle/date rules.

---

## Phase DAB-0 — Freeze the contract and version baseline

### Work

- Record the exact DAB runtime/CLI/container version.
- Pin the runtime artifact; do not use `latest` for executable deployment artifacts.
- Preserve a sanitized `tools/list` response and representative tool exchanges.
- Record the exact configuration-schema reference used for validation.
- Record current entity, field, permission, and runtime-control matrices.
- Keep the existing 24-call payload/behavior ledger as baseline evidence.
- Confirm whether MCP server description text is transmitted to the LLM by QueryMate or only exposed during MCP initialization.

### Acceptance criteria

- The deployed version is reproducible.
- Tool names, descriptions, input schemas, defaults, and errors are captured.
- Configuration evidence and observed runtime behavior are clearly separated.
- No configuration behavior is changed.

---

## Phase DAB-1 — Preserve and verify hard read-only enforcement

### Work

- Keep create, update, delete, and execute tools disabled.
- Keep REST and GraphQL disabled until a concrete approved consumer requires them.
- Verify that only the three approved generic MCP tools are advertised.
- Verify every entity has explicit read-only action permissions.
- Verify the SQL Server identity has only approved `SELECT` access and cannot write, execute procedures, or alter schema.
- Confirm that hidden entities and fields cannot be selected, filtered, sorted, grouped, aggregated, or traversed.
- Keep connection strings and credentials outside source control.
- Document that HTTP 200 does not mean tool success; `isError` and structured result status must be inspected.

### Acceptance criteria

- DAB and SQL Server independently enforce read-only access.
- Disabled tools consistently return `ToolDisabled` or the matching version-specific error.
- No arbitrary SQL capability is exposed.
- Static security evidence is not presented as runtime verification.

---

## Phase DAB-2 — Classify and normalize metadata content

### Problem

Current descriptions mix:

1. Stable technical schema facts.
2. Stable business meaning.
3. Environment-specific POC/DEV caveats.
4. Query-planning instructions.
5. Repeated audit/lifecycle descriptions.

This works for three entities but will produce large and difficult-to-govern metadata at 40+ entities.

### Work

- Classify every entity and field description into the five categories above.
- Preserve essential facts such as:
  - canonical/business keys;
  - preservation of leading zeroes;
  - logical relationship targets;
  - absence of automatic joins/foreign keys;
  - distinct business-object counting rules;
  - lifecycle/status meaning;
  - special date/sentinel meaning.
- Identify repeated text that can be governed once instead of copied across entities.
- Identify environment-specific statements that should not appear as permanent production semantics.
- Shorten obvious audit-field descriptions only after confirming the model does not need them for representative questions.
- Keep descriptions understandable without database source-code knowledge.

### Acceptance criteria

- No canonical-key or relationship meaning is lost.
- POC/DEV caveats are clearly separated from stable production metadata.
- Repeated descriptions decrease in measured bytes.
- Golden QueryMate questions retain correct entity/key/status/count behavior.

---

## Phase DAB-3 — Make relationships explicit and reviewable

### Confirmed current semantics

- `Relation.LINK_SYS_ID` logically refers to `Counterparty.CPY_ID` when `LINK_LEVEL = CPY`.
- `Relation.CODE_UE` logically refers to `BookingLocation.CODE`.
- Table-local surrogate `ID` values must not be joined across entities.
- Multiple `Relation` rows may exist for the same counterparty/location pair.
- Counting counterparties may require distinct `LINK_SYS_ID`, not row count.
- Current business links are not necessarily backed by physical foreign keys or automatic DAB joins.

### Work

- Create a reviewed relationship inventory with source, target, fields, condition, cardinality, uniqueness, and counting semantics.
- Configure native DAB relationships only when supported by the pinned version, accurate for the schema, useful to an enabled protocol, and safe under permissions.
- Do not add a misleading relationship merely to make metadata look complete.
- When native relationships cannot represent conditional links such as `LINK_LEVEL = CPY`, preserve them as reviewed metadata for QueryMate rather than inventing a database constraint.
- Determine exactly which relationship information appears in `describe_entities` and which requires a separate reviewed metadata handoff.

### Acceptance criteria

- Every cross-entity path is backed by physical or reviewed logical evidence.
- Conditional and polymorphic links remain explicit.
- QueryMate can distinguish relationship-row counts from business-object counts.
- No claim is made that MCP automatically joins entities unless observed behavior proves it.

---

## Phase DAB-4 — Optimize metadata and result payload behavior

### Work

- Preserve and test full, name-only, and selected-entity discovery behavior.
- Confirm whether name-only results include entity descriptions and measure that contribution.
- Confirm selected-entity filtering returns only requested approved entities.
- Keep explicit field projection available for reads.
- Keep aggregate count, grouped count, and distinct count behavior.
- Preserve cursor/pagination semantics and the hard maximum page size of 10.
- Review response envelopes for optional duplicated messages, metadata, null-heavy content, or verbose wrappers.
- Change payload shape only through supported DAB configuration/version features and only with backward-compatibility review.
- Keep errors precise enough for QueryMate to classify without returning SQL, stack traces, secrets, or records.

### Acceptance criteria

- Payload reductions are measured in characters/bytes and later validated through QueryMate provider-token usage.
- Projection and aggregate cases remain functionally correct.
- Selected-entity discovery is reliable.
- Error classification remains deterministic.
- No custom DAB fork is created for cosmetic payload savings.

---

## Phase DAB-5 — Evaluate MCP description and tool-schema duplication

### Work

- Compare the runtime MCP description, built-in tool descriptions, input JSON Schemas, QueryMate system prompt, and entity descriptions.
- Mark each rule with one primary owner.
- Retain tool-contract details required for correct direct MCP clients.
- Remove duplicate wording from configurable DAB descriptions only after proving the same fact is available authoritatively elsewhere in the contract.
- Determine which built-in tool descriptions are version-owned and cannot safely be edited.
- Prefer official configuration extension points over wrappers or forks.
- Re-capture `tools/list` and invalid-call behavior after every accepted description change.

### Rules that should remain hard DAB concerns

- Enabled and disabled operations.
- Supported argument schema.
- Maximum page size and pagination rules.
- Valid filter/aggregate behavior.
- Field and entity validation.
- Read permissions.

### Acceptance criteria

- `tools/list` size decreases only when supported and safe.
- Invalid tool calls do not increase in the QueryMate golden set.
- DAB remains usable by a standards-compliant MCP client without QueryMate-specific hidden assumptions.

---

## Phase DAB-6 — Scale approved configuration toward 40+ entities

### Work

- Maintain explicit allow-list exposure and deny-by-default review.
- Keep one configuration file while it remains reviewable.
- If splitting becomes necessary, use DAB's native multi-file support and verify pinned-version limitations before changing layout.
- Keep related entities together where cross-file relationship limitations apply.
- Require globally unique entity names.
- Use consistent ordering for source, description, fields, relationships, health, protocol enablement, and permissions.
- Add entity/domain ownership and sensitivity classification outside secrets.
- Prevent candidate/generated metadata from automatically becoming runtime exposure.

### Acceptance criteria

- All exposed entities and fields have human approval.
- Configuration validation passes on the pinned DAB version.
- Relationships resolve correctly within the chosen layout.
- Excluded entities cannot enter runtime config through generation or file splitting.
- Reviewers can understand a change without reading unrelated domains.

---

## Phase DAB-7 — Provide a governed metadata handoff for QueryMate

### Goal

Allow QueryMate to cache or index approved schema/business metadata without making DAB responsible for vector search or LLM orchestration.

### Metadata package

For each approved entity, provide or derive:

- Stable entity identifier and name.
- Concise purpose.
- Physical source and approved exposed fields.
- Canonical/business key.
- Display/lookup fields.
- Filter, sort, group, and aggregate capabilities where discoverable.
- Reviewed relationship edges.
- Lifecycle/status rules.
- Business-object counting semantics.
- Domain/security classification.
- DAB config/schema fingerprint and generation time.

### Rules

- Technical metadata may be generated as a candidate.
- Business meaning and logical relationships require review.
- The handoff must contain no credentials or row data.
- QueryMate must validate retrieved entity/field names against current DAB exposure before execution.
- DAB configuration remains the runtime authorization source; a RAG index never grants access.

### Acceptance criteria

- Unchanged DAB configuration produces stable metadata identifiers/fingerprints.
- Removed entities or fields are invalidated from QueryMate's active metadata.
- Security classification travels with metadata.
- Candidate metadata cannot overwrite runtime DAB configuration automatically.

---

## Phase DAB-8 — Harden shared and production environments

### Work

- Replace Simulator/anonymous POC behavior with the approved enterprise authentication and role model.
- Remove anonymous read access unless explicitly approved for the target environment.
- Verify role inheritance/effective permissions against the pinned version.
- Restrict detailed health information.
- Set production-appropriate logging and telemetry levels.
- Disable query-executor debug logging outside a controlled diagnostic window if it can expose query details or parameters.
- Apply TLS/private networking/secret management through the selected platform.
- Define request, timeout, payload, and rate controls appropriate to production.

### Acceptance criteria

- Shared environments cannot start with local Simulator/anonymous settings by accident.
- Logs and traces contain no credentials or business records.
- Authentication failures and authorization failures are distinguishable.
- Least-privilege SQL and DAB permissions are runtime-tested.

---

## Phase DAB-9 — Add contract and regression gates

### Automated or repeatable checks

- Pinned-version `dab validate`.
- Health check against a disposable/test database.
- Sanitized `tools/list` contract snapshot comparison.
- Full, name-only, and subset description tests.
- Projected read and maximum-page tests.
- Count, grouped count, and distinct count tests.
- Unknown entity/field/operator tests.
- Malformed filter and quoting tests.
- Page size above 10.
- Disabled create/update/delete/execute tests.
- SQL read-only negative tests in a disposable environment.
- Secret/config scan.

### Acceptance criteria

- Tool or error contract drift is visible before QueryMate integration breaks.
- Writes remain disabled at MCP and SQL layers.
- Payload-size regressions are reported.
- Production credentials and row data are never used in ordinary CI.

## 4. Recommended execution order

| Order | Phase | Primary outcome | QueryMate dependency |
|---:|---|---|---|
| 1 | DAB-0 | Reproducible contract baseline | None |
| 2 | DAB-1 | Confirmed read-only enforcement | QueryMate error classification benefits |
| 3 | DAB-2 | Cleaner governed metadata | Prompt/schema-context review |
| 4 | DAB-3 | Reliable relationship semantics | Query-plan design |
| 5 | DAB-4 | Smaller, bounded results | QueryMate projection/aggregation policy |
| 6 | DAB-5 | Reduced description duplication | QueryMate prompt disposition |
| 7 | DAB-6 | Maintainable 40+ entity config | Schema-selection strategy |
| 8 | DAB-7 | Safe metadata handoff | QueryMate cache/RAG phases |
| 9 | DAB-8 | Production security | Authentication propagation |
| 10 | DAB-9 | Contract stability | Golden integration tests |

## 5. Cross-application dependency gates

| Before QueryMate does this | DAB must provide/confirm |
|---|---|
| Remove detailed filter syntax from the system prompt | Stable tool JSON Schema and tested errors |
| Prevalidate entities and fields | Versioned approved metadata |
| Stop model-driven schema discovery | Reliable metadata refresh/fingerprint |
| Build relationship-aware plans | Reviewed relationship inventory |
| Add schema RAG | Governed metadata package and security tags |
| Shorten tool descriptions through an adapter | Pinned contract and regression snapshots |
| Treat a tool failure deterministically | Stable `isError` and structured error categories |

## 6. Decisions explicitly deferred

- Whether native DAB relationships can correctly represent all logical links.
- Whether configurable MCP descriptions can materially reduce `tools/list` size.
- Exact native multi-config layout for 40+ entities.
- Production identity provider, claims, and role mapping.
- Exact metadata-export mechanism.
- Views or physical schema changes for relationships not enforced today.

These decisions require the pinned DAB version, security review, and measured QueryMate behavior.

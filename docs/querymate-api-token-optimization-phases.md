# QueryMate API Token-Optimization Plan

Status: Proposed for phased review and execution

Application: `querymate-api`

Scope: Spring Boot, Spring AI, LLM orchestration, system prompt, query planning, MCP client behavior, schema-context selection, response formatting, and token diagnostics. This document does not implement changes.

Companion document: `querymate-dab-token-metadata-optimization-phases.md`

## 1. Objective

Reduce provider-reported tokens per QueryMate question without weakening correctness, security, read-only behavior, business semantics, or response quality.

The priority order is:

1. Reduce unnecessary LLM rounds.
2. Reduce repeated input sent on every round.
3. Reduce schema and tool-result payloads.
4. Prevent invalid retries and oversized reads.
5. Add schema RAG only when measured scale justifies it.

Token reduction is not accepted if it increases hallucination, selects the wrong entity, changes business-count semantics, bypasses DAB validation, or returns incomplete answers without disclosure.

## 2. Confirmed baseline

The current evidence establishes:

| Component | Observed value |
|---|---:|
| Earlier exchange total | Approximately 25,231 provider-reported tokens |
| Exchange after reducing maximum rows to 10 | Approximately 16,520 provider-reported tokens |
| Current system prompt | Approximately 10,620 characters |
| DAB `tools/list` response | 5,049 bytes |
| `describe_entities` full response | 9,778 bytes |
| `describe_entities` name-only response | 1,977 bytes |
| `describe_entities` subset response | 2,971 bytes |
| One projected record response | 576 bytes |
| One unprojected record response | 1,010 bytes |
| Ten unprojected records response | 8,245 bytes |
| Aggregate responses | Approximately 369–816 bytes |
| DAB maximum/default page size | 10 |

The deployed model is `GLM-5.3-Flash-NVFP4` through the company-provided OpenAI-compatible service. Exact token counts must come from provider-reported usage. Character or byte measurements are diagnostic indicators, not exact token counts.

## 3. Architecture boundaries

### QueryMate API owns

- User intent interpretation and conversational context.
- The system prompt and user-facing behavior.
- LLM and MCP orchestration.
- Selection of relevant schema/business metadata.
- Query-plan state, validation, retry limits, and loop termination.
- Differentiating MCP success, errors, and valid empty results.
- Provider token diagnostics.
- Final response category, JSON contract, and presentation.
- Future schema-RAG retrieval and metadata-cache lifecycle.

### QueryMate DAB owns

- Approved entities and fields.
- Entity/field descriptions exposed by DAB.
- Tool availability.
- Valid filter, sort, pagination, projection, and aggregation execution.
- DAB roles and read permissions.
- Hard response and page limits.
- Safe MCP errors.

### SQL Server owns

- Physical schema truth.
- Keys and constraints.
- Least-privilege read-only enforcement.
- Views/indexes justified by actual query needs.

QueryMate must not reproduce DAB's query engine or treat prompt compliance as a security boundary.

## 4. Measurement rules for every phase

Before and after each phase, run the same golden question set and record:

- LLM round count.
- Provider-reported input, output, and total tokens per round.
- Exchange totals.
- System-prompt characters and UTF-8 bytes.
- Exposed tool-definition characters and bytes.
- Tool-result characters and bytes by call.
- Tool names and call count, without arguments or business data in logs.
- End-to-end latency.
- Response category: `ANSWER`, `CLARIFICATION`, `EMPTY`, or `PARTIAL`.
- Accuracy, grounding, and business-semantic result.
- Invalid-call and retry count.

Do not claim savings from prompt length alone. A smaller payload that causes an additional LLM round may increase total tokens.

## 5. Golden regression set

Maintain representative sanitized questions covering:

1. Direct lookup by canonical identifier.
2. Lookup by descriptive label requiring identifier resolution.
3. Filtered record retrieval with explicit projection.
4. Count of rows.
5. Count of distinct business objects.
6. Grouped aggregation.
7. Question involving `BookingLocation`, `Counterparty`, and `Relation` semantics.
8. Active-only request using `SYS_STATUS = AC`.
9. Explicit request for historical/superseded records.
10. Ambiguous business term requiring clarification.
11. Valid empty result.
12. Unknown entity or field.
13. Invalid filter syntax.
14. Page size above the DAB maximum.
15. DAB unavailable/timeout.

Use synthetic or approved test data. The golden set must verify meaning, not only JSON shape.

---

## Phase API-0 — Freeze the baseline

### Work

- Preserve the current system prompt and resolved tool definitions as sanitized diagnostic artifacts.
- Record Level 2 per-round provider usage for the golden set.
- Record Level 3 component sizes where available: system prompt, history, tool definitions, tool results, and final response.
- Record the current round sequence for every question.
- Identify questions that unnecessarily call `describe_entities`, retrieve excess rows, or retry invalid calls.

### Acceptance criteria

- Every golden question has a reproducible baseline.
- Provider usage is available per model round.
- Round sequences and tool-result sizes are visible without logging sensitive content.
- No behavior changes are made.

---

## Phase API-1 — Refactor the system prompt without changing behavior

### Goal

Remove duplicated mechanical instructions while retaining semantic judgment, grounding, security, and response-quality rules.

### Keep in the prompt

- Use only approved tools and retrieved evidence for database facts.
- Never invent entities, fields, records, relationships, status meanings, or causes.
- Clarify only materially ambiguous requests.
- Resolve descriptive labels to canonical identifiers when required.
- Distinguish business-object counts from relationship-row counts.
- Treat successful empty results differently from tool failures.
- Do not expose SQL, credentials, hidden reasoning, raw payloads, or internal implementation details.
- Return only supported information and clearly label partial answers.
- Preserve concise user-facing presentation rules that cannot yet be enforced deterministically.

### Shorten or remove from the prompt after verification

- Complete OData grammar already present in the MCP tool schema.
- Repeated operator lists.
- Repeated single-quote/escaping examples.
- Exact maximum page-size text enforced by DAB.
- Lists of disabled write tools enforced by DAB.
- Repeated field-selection syntax already defined in JSON Schema.
- Detailed invalid-argument recovery already available in DAB errors.
- Repeated entity and field details supplied through selected schema context.

### Method

- Produce a rule-disposition table: `Keep`, `Shorten`, `Move to Java`, `Enforce in DAB/SQL`, or `Retrieve as metadata`.
- Change one logical prompt section at a time.
- Run the full golden set after every material reduction.
- Restore any rule whose removal causes measurable semantic or safety regression.

### Acceptance criteria

- Prompt character and provider-token contribution decrease.
- No increase in wrong tool, entity, field, operator, or response-category decisions.
- No security or grounding regression.
- Average LLM rounds do not increase.

---

## Phase API-2 — Classify MCP outcomes and control retries in Java

### Problem

DAB returns HTTP 200 for successful calls and for tool-level failures. Invalid calls carry `isError=true` and a structured error body. HTTP status alone is not a success signal.

### Work

- Classify MCP results into:
  - success with records/aggregate;
  - successful empty result;
  - recoverable validation error;
  - nonrecoverable validation error;
  - unauthorized/tool-disabled error;
  - timeout/unavailable transport failure.
- Treat DAB `isError` and structured status/error fields as authoritative.
- Set a strict per-request correction budget, normally no more than one safe correction for a recoverable validation failure.
- Never retry unchanged invalid arguments.
- Never retry disabled write/execute tools.
- Prevent the model from interpreting an error response as matching data.
- Record structured diagnostic events without logging tool arguments or returned business data.

### Acceptance criteria

- No infinite or repeated invalid-call loop.
- Tool-disabled and unauthorized results stop immediately.
- Empty success is not reported as failure.
- Correctable validation errors receive at most the configured safe retry.
- Golden error cases retain accurate response categories.

---

## Phase API-3 — Apply a bounded query-result policy

### Work

- Prefer `aggregate_records` for counts, grouped counts, and distinct counts.
- Require explicit field projection for record retrieval unless the user genuinely requests all exposed fields.
- Default to the smallest result size that can answer the question.
- Keep DAB's maximum page size of 10 as hard enforcement.
- Allow pagination only when the question requires additional rows and a continuation cursor is available.
- Stop pagination as soon as the answer is established.
- Do not retrieve records merely to calculate an aggregation supported by DAB.
- Preserve identifiers and fields needed to interpret or verify the result.

### Acceptance criteria

- Straightforward count questions use aggregation instead of row retrieval.
- Unprojected result payloads decrease.
- Pagination cannot run without a bounded reason and termination rule.
- Answers remain complete for the user's requested scope.

---

## Phase API-4 — Move schema discovery outside the LLM loop

### Goal

Avoid asking the model to rediscover stable schema metadata during every question.

### Work

- Add a QueryMate-owned schema metadata service that obtains approved metadata through DAB using a controlled non-LLM path.
- Cache compact metadata separately from chat memory.
- Preserve:
  - entity name and concise purpose;
  - canonical/business key;
  - relevant display/lookup fields;
  - filter/aggregate fields;
  - reviewed relationship edges;
  - business-count semantics;
  - lifecycle/status rules;
  - security/domain tags.
- Track the DAB version/configuration fingerprint and metadata refresh time.
- Refresh on controlled startup, TTL, explicit administration event, or detected schema/config version change.
- Fail safely when cached metadata is stale or unavailable.
- Do not copy database credentials or bypass DAB permissions.

### Important design rule

Do not automatically replace one full describe call with both name-only and subset describe calls inside the model loop. Although the payloads are smaller, the additional LLM round may cost more because the prompt and tool definitions repeat.

### Acceptance criteria

- Stable schema discovery no longer requires an LLM round for ordinary questions.
- Cached metadata is versioned, bounded, and derived only from approved exposure.
- Straightforward questions reach the data tool without first calling `describe_entities`.
- Stale metadata cannot silently authorize an entity or field absent from current DAB configuration.

---

## Phase API-5 — Introduce a compact query-planning boundary

### Target flow

For a straightforward question:

1. Interpret the request and candidate entities.
2. Supply compact relevant metadata.
3. Produce a small internal plan.
4. Validate deterministic plan fields.
5. Execute one read or aggregate call.
6. Generate the grounded final response.

Aim for two LLM rounds for ordinary single-operation questions: one tool-requesting round and one final-answer round.

### Compact plan fields

- Intent: lookup, list, count, grouped aggregate, comparison, or clarification.
- Candidate entity/entities.
- Required fields.
- Filter conditions.
- Aggregation/grouping/distinct semantics.
- Result limit.
- Lookup-resolution requirement.
- Evidence still required.

### Java validation

- Entity and field exist in current approved metadata.
- Operation is read-only and supported.
- Page size does not exceed the DAB maximum.
- Aggregation function is compatible with the selected field.
- Required projection fields are present.
- Retry budget has not been exhausted.

Java validation must not invent business intent or silently choose among materially different interpretations.

### Acceptance criteria

- Ordinary questions avoid exploratory tool loops.
- Lookup-resolution questions use the minimum necessary additional call.
- Invalid plans fail before a wasteful DAB/LLM cycle when deterministic validation is possible.
- User-visible behavior remains grounded and understandable.

---

## Phase API-6 — Make response formatting more deterministic

### Work

- Keep the public response contract with `answer` and `responseType`.
- Use supported Spring AI structured-output capabilities or application validation where they compile against the resolved Spring AI 2.0.1 version.
- Validate allowed response-type values in Java.
- Repair or reject malformed envelopes without asking the model for another full reasoning round where a deterministic correction is safe.
- Keep content decisions with the model; keep envelope correctness with Java.
- Preserve Markdown inside `answer` only where needed.

### Acceptance criteria

- Malformed JSON/envelope retries decrease.
- Response categories remain semantically correct.
- User-visible answers do not gain unnecessary formatting verbosity.

---

## Phase API-7 — Add schema retrieval for 40+ entities

### Trigger

Do not start this phase merely because a future table count is known. Start when measurements show one or more of:

- compact metadata no longer fits the allocated context budget;
- entity-selection accuracy declines;
- full/name-only discovery is repeatedly required;
- relationship-path selection becomes unreliable;
- multiple business domains require access-scoped metadata retrieval.

### Candidate retrieval documents

- Entity cards.
- Relationship cards/paths.
- Selective field cards.
- Business glossary and synonym cards.
- Status/code meaning cards.
- Counting and identifier-resolution rules.

### Requirements

- Use human-reviewed metadata derived from approved DAB exposure.
- Store stable identifiers, schema/config fingerprint, domain, and security classification.
- Exclude credentials, row data, hidden/internal entities, and unapproved fields.
- Retrieve candidate entities plus relationship neighbors needed for the question.
- Verify retrieved entity and field names against current DAB metadata before execution.
- Measure retrieval precision/recall using the golden set expanded for 40+ entities.

### Acceptance criteria

- Relevant schema context stays bounded as entity count grows.
- Retrieval does not expose metadata the user is not authorized to use.
- Entity and relationship selection meets the agreed accuracy target.
- Total tokens are lower than full-catalog injection or model-driven full discovery.

---

## Phase API-8 — Evaluate tool-definition optimization

### Work

- Measure the exact serialized definitions exposed to the LLM on every round.
- Identify instructions duplicated between tool descriptions and the compact system prompt.
- First prefer supported DAB configuration changes documented in the companion plan.
- If DAB's built-in descriptions cannot be adjusted, evaluate a thin QueryMate tool-metadata adapter only if it preserves argument schemas, behavior, validation, and names.
- Do not create custom database tools or bypass DAB merely to shorten descriptions.
- Do not hide information the model requires for correct filter, aggregation, projection, or pagination behavior.

### Acceptance criteria

- Tool-definition input decreases without increasing invalid calls.
- QueryMate remains compatible with the approved DAB MCP contract.
- No custom DAB fork or duplicated query engine is introduced.

---

## Phase API-9 — Roll out with measured gates

### Rollout sequence

1. Development diagnostic profile.
2. Automated golden-set comparison.
3. Controlled POC/user validation.
4. Small shared environment with production-like authentication and telemetry controls.
5. Broader adoption only after correctness and security gates pass.

### Required comparison report

For every phase, report:

- baseline versus new median and high-percentile token totals;
- round-count distribution;
- tool-call distribution;
- latency;
- invalid calls/retries;
- correctness regressions;
- prompt, tool-definition, and tool-result size changes;
- known limitations.

### Completion criteria

- Token reduction is demonstrated with provider-reported usage.
- Correctness and security remain at or above baseline.
- Straightforward questions usually complete in the target number of rounds.
- The implementation does not rely on hidden mutable global request state.
- DAB and SQL Server remain the hard enforcement layers for data access.

## 6. Recommended execution order

| Order | Phase | Expected primary benefit | DAB dependency |
|---:|---|---|---|
| 1 | API-0 | Trustworthy baseline | None |
| 2 | API-1 | Smaller repeated prompt | Stable current contract |
| 3 | API-2 | Fewer wasted error rounds | Stable error shapes |
| 4 | API-3 | Smaller tool results | Existing aggregate/projection support |
| 5 | API-4 | Remove repeated discovery rounds | Approved metadata access |
| 6 | API-5 | Fewer exploratory rounds | Reviewed relationship semantics |
| 7 | API-6 | Fewer formatting retries | None |
| 8 | API-7 | Scale schema selection to 40+ | Metadata/export governance |
| 9 | API-8 | Reduce repeated tool-schema input | DAB tool-description findings |
| 10 | API-9 | Safe rollout | All accepted dependencies |

## 7. Decisions explicitly deferred

- Exact rewritten system-prompt text until the rule-disposition review is approved.
- Exact schema-cache implementation and persistence technology.
- Vector store and embedding model for future schema RAG.
- Whether hybrid lexical/vector retrieval is required.
- Whether DAB tool descriptions can be configured sufficiently or require a QueryMate metadata adapter.
- Exact production authentication propagation and user-specific metadata filtering.

These decisions should be made from measurements and the actual resolved framework/DAB versions, not assumptions.

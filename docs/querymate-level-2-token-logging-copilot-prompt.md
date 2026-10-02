# Copilot Implementation Prompt: QueryMate Level 2 Token Diagnostics

You are acting as a Senior Java Technical Lead working on the QueryMate AI application.

Implement **Level 2 token diagnostics**: log the exact provider-reported token usage for **every individual LLM call/round** inside Spring AI's framework-managed tool-calling loop.

This is an implementation task. Inspect the repository, make the code changes, run the appropriate tests, and report the completed result. Do not stop after proposing a design.

## Project context

The expected application baseline is:

- Backend module: `querymate-api`
- Base package: `com.querymate.api`
- Spring Boot 4.0.6
- Spring AI 2.0.1
- Maven
- `ChatClient`
- Spring AI MCP client and `ToolCallbackProvider`
- Spring AI framework-managed tool execution through `ToolCallingAdvisor`
- Approved MCP tools filtered through the existing QueryMate tool policy
- An OpenAI-compatible company LLM endpoint

The repository documentation is under `docs/`. Treat the latest QueryMate API implementation document and the actual source code as authoritative. Some older documents may describe an earlier module name or architecture; do not recreate an old structure when the current code differs.

## Required first step: inspect the code

Before editing, inspect at least:

- root `pom.xml`
- `querymate-api/pom.xml`
- all existing classes that build or configure `ChatClient`
- `ChatService` and the active chat request flow
- current Advisor registration and ordering
- existing MCP/tool configuration and `ToolCallbackProvider` use
- `application.yaml` and any profile-specific configuration
- existing logging, correlation ID, observability, and configuration-property conventions
- existing tests and package conventions
- any `AGENTS.md` or repository-specific instructions

Confirm the resolved Spring AI version from Maven. Use only APIs that compile against the project's resolved version. Do not invent APIs from memory.

## Goal

For one user question, QueryMate may perform several LLM calls:

1. The first LLM call receives the system prompt, user/history messages, and exposed tool definitions.
2. The model requests an MCP tool.
3. Spring AI executes the tool.
4. Another LLM call receives the prior messages and tool result.
5. The loop continues until the model returns the final answer.

Add diagnostics that make these individual LLM rounds visible. A typical log sequence should be equivalent to:

```text
event=querymate_llm_round requestId=... conversationId=... round=1 inputTokens=8200 outputTokens=700 totalTokens=8900 requestedTools=[describe_entities] terminal=false durationMs=...
event=querymate_llm_round requestId=... conversationId=... round=2 inputTokens=9600 outputTokens=900 totalTokens=10500 requestedTools=[aggregate_records] terminal=false durationMs=...
event=querymate_llm_round requestId=... conversationId=... round=3 inputTokens=12300 outputTokens=2800 totalTokens=15100 requestedTools=[] terminal=true durationMs=...
event=querymate_llm_exchange_summary requestId=... conversationId=... rounds=3 inputTokens=30100 outputTokens=4400 totalTokens=34500
```

Use the project's established logging style. The exact text layout may change, but every field and behavior described below must remain available.

## Core implementation requirement

Implement a small custom Spring AI Advisor, with a clear name such as `TokenUsageLoggingAdvisor`, that observes every blocking LLM invocation in the tool loop.

The Advisor must:

- implement the Spring AI 2.0.1 `CallAdvisor` contract, unless the inspected project already uses an equivalent supported base type;
- be registered on the same `ChatClient` used by QueryMate;
- run **inside** the recursive `ToolCallingAdvisor` loop;
- use an order greater than `ToolCallingAdvisor.DEFAULT_ORDER` (Spring AI documents the default tool advisor at `Ordered.HIGHEST_PRECEDENCE + 300`; an observing advisor at approximately `+400` is inside the loop);
- call `callAdvisorChain.nextCall(request)` exactly once;
- time that individual model invocation;
- obtain the individual response from `ChatClientResponse.chatResponse()`;
- read `Usage` from `chatResponse.getMetadata().getUsage()`;
- log input/prompt, output/completion, and total tokens for that individual round;
- detect tool calls in that round and log only their tool names;
- identify whether the round is terminal (`chatResponse.hasToolCalls()` is false);
- gracefully handle a null response, absent metadata, unavailable usage, or unavailable token fields without breaking the user's request;
- return the original response unchanged.

Use the exact getter names available in the resolved Spring AI version. The standard Spring AI `Usage` contract is expected to expose:

```java
usage.getPromptTokens();
usage.getCompletionTokens();
usage.getTotalTokens();
```

Do not calculate tokens from character counts or use an unrelated tokenizer. The provider-reported `Usage` values are the source of truth for Level 2.

## Round and total tracking

Each logical QueryMate request must have an isolated round counter and isolated cumulative totals.

Required behavior:

- The first LLM invocation for a user question is round 1.
- Every further invocation caused by tool calling increments the round.
- Accumulate prompt/input, completion/output, and total tokens across the rounds.
- Emit one exchange summary after the terminal model round.
- The summary must equal the sum of the per-round provider-reported values.
- If a round has no usable `Usage`, log that usage is unavailable and do not invent a number. The summary should indicate incomplete measurement if one or more rounds could not be measured.

Do **not** put an `AtomicInteger`, mutable token totals, or any other request state directly on a singleton Advisor bean. That would mix simultaneous users and conversations.

Prefer a request-local tracker propagated through `ChatClientRequest.context()` if the actual Spring AI tool-loop implementation preserves that context across iterations. Verify this in the resolved 2.0.1 implementation or with a focused test. Use constants for context keys and a focused internal tracker type. If the current application already has a safe request context/correlation abstraction, reuse it.

Do not use a static mutable map. Do not leave request entries behind after success or failure. Avoid `ThreadLocal`, especially if the current or planned path is asynchronous or streaming.

If the actual framework behavior prevents a single inside-loop Advisor from emitting a reliable final summary, keep the inside-loop Advisor responsible for exact per-round logs and add the smallest supported outer boundary needed for one cumulative summary. Do not create a custom tool-calling loop.

## Request and conversation correlation

Include the following identifiers when they are already available safely:

- request/correlation ID;
- conversation ID;
- LLM round number.

Reuse existing MDC or request-context conventions. Do not generate a second competing correlation ID when the application already has one. Do not fail the request when an identifier is absent; use a safe placeholder or omit the field according to the existing logging convention.

The implementation must remain correct when several requests run concurrently.

## Tool-call reporting

For a response that requests tools, obtain the tool calls using supported Spring AI response APIs. Log:

- the requested tool name or names;
- the number of tool calls;
- whether another model round is expected.

Do not log tool arguments. Do not log tool results. Do not log the raw MCP payload.

The log/documentation should make the following distinction clear:

- Executing the Java/MCP tool itself does not consume LLM tokens.
- Tokens are consumed when tool definitions are included in an LLM request and when tool-result messages are sent in a later LLM request.
- Level 2 reports exact usage per LLM round; it does not attribute the input tokens to system prompt versus history versus tool schemas versus tool results.

## Configuration

Make token diagnostics configurable and disabled or appropriately restricted outside diagnostic environments.

Follow existing QueryMate configuration conventions. Prefer an application-specific property such as:

```yaml
querymate:
  observability:
    token-logging-enabled: true
```

If an equivalent property already exists, reuse it. Do not create duplicate configuration concepts.

Register the Advisor conditionally using supported Spring Boot configuration. Use a sensible default based on the project's current development posture. Document how to enable and disable it.

Do not add a new third-party dependency just for this feature. SLF4J and the existing Spring/Spring AI APIs are sufficient.

## Security and privacy constraints

Token diagnostics must never log:

- system-prompt text;
- user questions or conversation messages;
- model completion text;
- hidden reasoning or `reasoning_content`;
- tool arguments;
- tool-result data;
- SQL;
- database records;
- API keys, bearer tokens, credentials, or endpoint secrets;
- full request or response objects whose `toString()` may expose content.

Logging tool names and numeric usage metadata is allowed. Keep this feature safe for enterprise data.

Do not enable globally verbose `DEBUG` or `TRACE` logging.

## Scope boundaries

This task is **Level 2 only**.

Do not implement:

- component-level token estimation for system prompt, history, tool definitions, or tool results;
- calls to vLLM `/tokenize`;
- prompt compression;
- dynamic tool selection;
- changes to the QueryMate system prompt;
- changes to chat-memory behavior;
- changes to MCP/DAB behavior or schemas;
- a custom OpenAI client;
- a custom recursive tool-calling engine;
- UI changes;
- database persistence;
- a new backend module;
- broad observability infrastructure unrelated to this diagnostic.

Preserve the existing API response and user-visible behavior.

## Streaming

Implement the active production path found in the repository.

If QueryMate currently uses blocking `.call()`, implement and validate `CallAdvisor` support now. Do not add reactive infrastructure merely for this task.

If the active chat flow already uses `.stream()`, implement the equivalent supported `StreamAdvisor` behavior without logging one token record per chunk and without double counting. Aggregate the Spring AI stream in the supported manner and emit one usage record per model round. Do not buffer or change the user-visible stream unless the current Spring AI API requires and safely supports it.

State clearly in the implementation summary whether blocking, streaming, or both paths are covered.

## Tests

Add focused tests that provide real value and do not contact the company LLM or MCP server.

At minimum, verify:

1. The Advisor order places it inside `ToolCallingAdvisor`.
2. A normal single-round response records one round with the exact mocked provider `Usage`.
3. A tool-calling response records the requested tool name and is marked non-terminal.
4. Multiple rounds accumulate into the correct summary.
5. Missing/null usage does not fail the chat request and is reported as unavailable/incomplete.
6. Two concurrent logical requests do not share counters or totals.
7. The Advisor returns the same response object/content without mutation.
8. Logs do not contain prompt content, tool arguments, or tool results.

Use the project's existing test stack and conventions. Do not use a live LLM or live MCP connection in normal Maven tests.

If exact log-capture assertions would make tests brittle, test a small structured diagnostic event/collector boundary and keep the actual SLF4J formatting thin. Do not add a large abstraction hierarchy solely for tests.

## Validation

After implementation:

1. Run the module's relevant unit tests.
2. Run the appropriate Maven compilation/test command for `querymate-api` and required upstream modules, using the repository's Maven wrapper when present.
3. Confirm the application context still starts in the available test profile if the project has such a test.
4. Confirm no live company endpoint, API key, or MCP server is required by ordinary tests.
5. Review the diff for accidental prompt/payload logging.

Use a command equivalent to the following only after adapting it to the actual repository:

```bash
./mvnw -pl querymate-api -am test
```

## Acceptance criteria

The task is complete when:

- each LLM invocation inside the Spring AI tool loop produces one exact provider-usage log entry;
- round numbers restart for each user request;
- simultaneous requests cannot corrupt each other's round numbers or totals;
- tool names are visible without arguments or results;
- a final exchange summary reports the sum of measured rounds;
- diagnostics can be enabled/disabled through configuration;
- no user-visible response behavior changes;
- no prompt, reasoning, business data, credentials, raw tool arguments, or raw tool results are logged;
- existing tests and the new focused tests pass.

## Expected final report

After making the changes, provide:

1. A concise explanation of the implementation.
2. The files created or changed.
3. The Advisor order used and why it observes every tool-loop round.
4. An example sanitized log sequence from a mocked or safe local flow.
5. Tests and Maven commands run, with results.
6. Whether blocking, streaming, or both paths are covered.
7. Any limitation caused by the actual provider or Spring AI API.

Do not claim that Level 2 can split prompt tokens into system-prompt, tool-schema, history, and tool-result buckets. That breakdown belongs to a later Level 3 diagnostic.

## Primary references

- Spring AI 2.0.1 ToolCallingAdvisor: <https://docs.spring.io/spring-ai/reference/api/tools/tool-calling-advisor.html>
- Spring AI usage handling: <https://docs.spring.io/spring-ai/reference/api/usage-handling.html>
- Spring AI observability: <https://docs.spring.io/spring-ai/reference/observability/>


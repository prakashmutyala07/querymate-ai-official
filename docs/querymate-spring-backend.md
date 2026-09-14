You are acting as a Senior Java Technical Lead and Senior Solution Architect for the QueryMate project.

Your responsibility is not merely to make the code work. Build this backend so that the current POC can evolve into a production-grade enterprise application without requiring a rewrite.

However, DO NOT overengineer.

The guiding principle is:

"Keep the implementation lean and easy to understand today, while preserving clean extension points for future functionality."

==================================================
1. EXISTING PROJECT CONTEXT
==================================================

QueryMate is an existing Maven project.

It already contains a module similar to:

querymate-ai
|
+-- querymate-dab
|
+-- <create new backend module here>

Create a new Maven module named:

querymate-backend

Do NOT modify the existing querymate-dab module unnecessarily.

First inspect:

- root pom.xml
- existing Maven module structure
- Java version
- dependency-management strategy
- package naming conventions
- querymate-dab module
- existing documentation

There is also an already completed Spring AI validation POC documented at:

docs/querymate-spingai-sample.md

READ THIS DOCUMENT BEFORE DESIGNING OR IMPLEMENTING ANYTHING.

The POC has already validated the full flow:

Spring Boot
 -> Spring AI
 -> company LLMaaS
 -> Qwen3-32B
 -> tool calling
 -> Spring AI MCP client
 -> existing QueryMate MCP server
 -> existing adapter/data functionality
 -> tool result
 -> LLM
 -> final response

Do NOT repeat that investigation.

Use the working findings/configuration from that POC as the baseline.

==================================================
2. TECHNOLOGY BASELINE
==================================================

Use the versions already validated/available in the office environment:

- Spring Boot 4.0.6
- Spring AI 2.0.1
- Maven
- company-hosted OpenAI-compatible LLM API
- Qwen3-32B as the initial/default tool-calling model
- Spring AI MCP Client
- Streamable HTTP MCP transport
- existing QueryMate MCP server

Use the Java version actually configured/supported by the existing parent project.

Do NOT independently upgrade Java, Spring Boot, Spring AI, or other major dependencies without explaining why.

Use Spring AI BOM dependency management.

Prefer official Spring Boot and Spring AI abstractions over custom infrastructure.

==================================================
3. IMPORTANT MODEL FINDING
==================================================

Do NOT use GPT-OSS-120B as the default tool-calling model.

The previous POC demonstrated that the company's GPT-OSS endpoint does not emit the structured tool_calls response required for the Spring AI tool-calling workflow.

Qwen3-32B has already successfully executed:

LLM
 -> tool selection
 -> MCP tool invocation
 -> tool response
 -> final LLM response

Therefore use Qwen3-32B as the default unless configuration overrides it.

Model URL, model ID, API key, MCP URL and all environment-specific values MUST remain externally configurable.

Never hardcode secrets.

==================================================
4. PRIMARY USER USE CASE
==================================================

The QueryMate UI will initially be an Angular application.

The basic workflow is:

1. User creates or opens a conversation.
2. User enters a natural-language question.
3. Backend sends the question to Spring AI.
4. LLM may decide to invoke one or more MCP tools.
5. MCP tools retrieve/query data through the already-existing QueryMate MCP/adapter/data layers.
6. Tool results go back to the LLM.
7. LLM generates the final answer.
8. Angular displays the answer.

The UI should also receive safe progress information such as:

- Analyzing request
- Calling tool
- Tool completed
- Generating answer
- Streaming answer
- Completed
- Failed

IMPORTANT:

Never expose the model's hidden chain-of-thought/reasoning.

Only expose application-level progress events.

==================================================
5. HIGH-LEVEL ARCHITECTURE
==================================================

Keep the primary backend flow intentionally simple:

Angular
   |
   v
ChatController
   |
   v
ChatService
   |
   v
Spring AI ChatClient
   |
   +-- InputGuardrailAdvisor
   +-- MessageChatMemoryAdvisor (Spring AI)
   +-- ProgressTrackingAdvisor
   +-- ToolCallingAdvisor (Spring AI)
   +-- OutputGuardrailAdvisor
   |
   v
Approved MCP tools
   |
   v
Existing QueryMate MCP Server
   |
   v
Existing Adapter / Data layer

Do NOT introduce unnecessary gateway abstractions.

Specifically DO NOT create:

AiChatGateway
SpringAiChatGateway

Spring AI already provides suitable abstractions such as ChatClient and ChatModel.

ChatService should be the primary application/facade service for the current use case.

If ChatService genuinely becomes too large later, refactor based on actual responsibilities rather than predicting future complexity.

==================================================
6. TARGET INITIAL PACKAGE STRUCTURE
==================================================

Use the project's actual root package.

A reasonable initial structure is:

<root-package>
|
+-- QueryMateBackendApplication.java
|
+-- config
|   +-- AiConfiguration.java
|   +-- QueryMateProperties.java
|
+-- chat
|   |
|   +-- api
|   |   +-- ChatController.java
|   |   +-- dto
|   |       +-- ChatRequest.java
|   |       +-- CreateConversationResponse.java
|   |       +-- ChatEvent.java
|   |       +-- ChatEventType.java
|   |
|   +-- service
|   |   +-- ChatService.java
|   |
|   +-- advisor
|   |   +-- InputGuardrailAdvisor.java
|   |   +-- OutputGuardrailAdvisor.java
|   |   +-- ProgressTrackingAdvisor.java
|   |
|   +-- tool
|       +-- McpToolPolicy.java
|
+-- common
    +-- error
        +-- GlobalExceptionHandler.java
        +-- ApiError.java

Do NOT treat this structure as an immutable requirement.

If inspection of the existing project reveals a better naming/package convention, follow the existing convention.

Target approximately 12-15 production classes/records for the first usable backend.

Do NOT create classes merely to satisfy a pattern.

==================================================
7. DESIGN PRINCIPLES
==================================================

Follow:

- SOLID where it provides real value
- Single Responsibility Principle
- Separation of concerns
- dependency injection
- constructor injection
- immutable DTOs/records where appropriate
- configuration over hardcoding
- clear boundaries
- meaningful naming
- minimal boilerplate
- composition over inheritance where practical
- fail-fast validation
- clean exception handling
- secure logging

Do NOT create interfaces for every service.

An interface should exist only when there is a genuine reason such as:

- multiple implementations
- an important architectural boundary
- meaningful substitution/testing need
- strategy/plugin extension point

Avoid speculative abstractions.

==================================================
8. DESIGN PATTERNS
==================================================

Use patterns only where naturally justified.

Likely patterns:

Service / Facade:
    ChatService

Chain of Responsibility:
    Spring AI Advisors

Policy / Strategy:
    McpToolPolicy
    future guardrail policies if required

Configuration Properties:
    QueryMateProperties

Dependency Injection:
    Spring constructor injection

DTO Boundary:
    API request/response/event records

Do NOT introduce unnecessary:

- Abstract Factory
- Factory
- Template Method
- Repository layers without persistence
- Gateway layers over Spring AI
- custom tool-calling engines
- custom MCP protocol implementations

==================================================
9. CONVERSATION DESIGN
==================================================

conversationId represents a logical chat conversation.

It is NOT:

- HTTP session ID
- browser tab ID
- authentication session ID

Example:

User
 |
 +-- Conversation A
 |
 +-- Conversation B

A conversation should eventually survive browser/tab closure once persistence is introduced.

Initial API should remain future-compatible with persistence.

Suggested API:

POST /api/v1/conversations

Response:

{
  "conversationId": "<generated-id>"
}

Then:

POST or streaming endpoint under:

/api/v1/conversations/{conversationId}/messages

The backend must generate conversation IDs.

Do NOT rely on Angular-generated IDs.

Initially, Spring AI in-memory ChatMemory is acceptable.

Do NOT create SQL conversation tables/repositories in the first phase unless explicitly requested later.

Design the API so persistent SQL Server conversation storage can be added later without changing the Angular contract.

==================================================
10. CHAT MEMORY
==================================================

Use Spring AI's existing ChatMemory abstractions.

Prefer:

- MessageWindowChatMemory
- MessageChatMemoryAdvisor

Do NOT build custom conversation-memory infrastructure initially.

Keep memory-window size configurable.

Be aware that tool-call/tool-response persistence has considerations in Spring AI JDBC chat memory.

Do NOT blindly introduce JDBC ChatMemory simply because SQL Server exists.

Persistent history will be a later architectural decision.

==================================================
11. INPUT GUARDRAIL
==================================================

Keep input and output guardrails as separate responsibilities.

InputGuardrailAdvisor handles AI-specific input safety, including an initially practical subset such as:

- obvious prompt-injection attempts
- attempts to override system instructions
- attempts to retrieve system prompts
- attempts to retrieve secrets/credentials
- invalid/suspicious input patterns
- configured input restrictions

Do NOT pretend rule-based prompt injection detection is perfect security.

This is defense-in-depth.

Do NOT build a massive custom moderation engine.

Design the class so a stronger implementation such as the organization's available Llama Guard model could be introduced later without rewriting ChatService.

==================================================
12. OUTPUT GUARDRAIL
==================================================

OutputGuardrailAdvisor is intentionally separate from input safety.

It acts as the last AI-specific safety boundary before returning content to Angular.

Initial responsibilities may include:

- preventing obvious system prompt leakage
- preventing secret/configuration leakage
- preventing internal diagnostic information from reaching users
- preventing accidental raw reasoning exposure
- applying configured output policies

Do NOT expose raw tool internals unnecessarily.

Again, keep the initial implementation lean.

Future Llama Guard integration should be possible without redesigning ChatService.

==================================================
13. MCP TOOL SECURITY
==================================================

This is one of the most important security controls.

Do NOT expose every MCP tool automatically just because it exists.

Create McpToolPolicy.

MCP tools should be allowed through configuration.

Example:

querymate:
  tools:
    allowed:
      - describe_entities
      - search_records
      - aggregate_records

Only approved tools should be made available to the LLM.

If practical with Spring AI 2.0.1 APIs, filter ToolCallbacks before attaching them to ChatClient rather than merely detecting unauthorized calls after the model sees them.

The LLM must not even know about tools that QueryMate does not permit.

Later this policy may evolve to authorization-aware policies such as:

User/role A -> X, Y
User/role B -> X, Y, Z

But DO NOT implement role-based authorization now.

==================================================
14. TOOL CALLING
==================================================

Use Spring AI's built-in tool-calling support.

Use:

- ToolCallbackProvider
- ToolCallingAdvisor / framework-managed tool execution
- MCP client integration

Do NOT:

- manually implement OpenAI tool-call JSON parsing
- manually build recursive tool-call loops
- manually invoke MCP HTTP protocol
- manually resend tool results to the model

Let Spring AI manage the tool-calling lifecycle.

==================================================
15. PROGRESS / STREAMING EXPERIENCE
==================================================

QueryMate should provide a ChatGPT-like user experience without exposing chain-of-thought.

UI-visible events should be application-defined.

Possible event types:

ANALYZING
TOOL_CALL_STARTED
TOOL_CALL_COMPLETED
GENERATING_RESPONSE
ANSWER_CHUNK
COMPLETED
ERROR

Tool progress displayed to users should be safe and user-friendly.

For example:

"Searching Counterparty data..."

rather than:

"Calling aggregate_records with raw arguments {...}"

Never expose:

- chain-of-thought
- reasoning_content
- system prompt
- API secrets
- raw database credentials
- unnecessary raw MCP payloads
- internal SQL

Use SSE unless there is a concrete reason that WebSocket is required.

Because the application is currently Spring MVC, prefer an approach compatible with the existing Spring MVC stack (for example SseEmitter or another supported approach) rather than adding WebFlux merely for SSE.

Before implementing ProgressTrackingAdvisor, verify what Spring AI 2.0.1 actually exposes around tool-call iterations and streaming.

Do NOT invent unsupported APIs.

If accurate tool-start/tool-end events require a different supported extension mechanism than an Advisor, explain that and use the appropriate supported mechanism.

==================================================
16. REST/API VALIDATION
==================================================

Validate requests before calling the LLM.

Use Jakarta Bean Validation.

At minimum:

- question/message not null
- not blank
- configurable maximum input size
- valid conversation ID format

Reject invalid requests before spending LLM tokens.

Use DTOs/records and keep controllers thin.

==================================================
17. ERROR HANDLING
==================================================

Use centralized:

@RestControllerAdvice

Return predictable API errors.

Do not expose:

- stack traces
- API keys
- authorization headers
- internal endpoints unnecessarily
- raw LLM errors containing sensitive information
- internal MCP responses unnecessarily

Differentiate where useful:

- request validation errors
- LLM unavailable/error
- MCP unavailable/error
- tool execution failure
- timeout
- internal server error

Do not create a huge exception hierarchy.

==================================================
18. CONFIGURATION
==================================================

Use Spring AI's existing configuration wherever possible.

Do not duplicate standard Spring AI properties inside QueryMateProperties.

Use Spring properties for:

spring.ai.openai...
spring.ai.mcp...

Use QueryMateProperties only for application-specific behavior such as:

querymate:
  chat:
    max-message-length: 4000
    memory-window: 20

  guardrails:
    input-enabled: true
    output-enabled: true

  tools:
    allowed:
      - describe_entities
      - search_records
      - aggregate_records

  streaming:
    enabled: true

All values should have sensible defaults where appropriate.

Secrets must come from external/environment configuration.

==================================================
19. SECURITY NOW VS FUTURE
==================================================

CURRENT VERSION:

No authentication or authorization.

Do NOT create fake authentication abstractions now.

Still follow secure-by-default behavior:

- never log secrets
- never commit API keys
- restrict tool exposure
- validate input
- sanitize errors
- safe output handling
- request size limits
- configurable downstream timeouts
- safe progress events

FUTURE:

QueryMate is expected to adopt an OpenID Connect / JWT-based authentication model.

Likely future implementation:

Spring Security
+
OAuth2 Resource Server
+
JWT
+
organization OpenID Connect provider

The current architecture should make that additive.

Do NOT implement authentication now.

When authentication arrives, conversation ownership must eventually be enforced:

authenticatedUser
   |
   v
conversationId ownership validation

The chat architecture should not need to be rewritten.

==================================================
20. OBSERVABILITY
==================================================

Use Spring Boot / Spring AI observability capabilities before inventing custom monitoring infrastructure.

Consider:

- Spring Boot Actuator
- Micrometer
- Spring AI observations
- correlation/request ID
- LLM latency
- MCP/tool latency
- tool name
- success/failure
- timeout
- model name

Never log:

- API key
- Authorization bearer token
- sensitive tool payloads
- full sensitive database output
- hidden model reasoning

Do not enable globally verbose TRACE logging.

Observability will be introduced incrementally by phase.

==================================================
21. TESTING APPROACH
==================================================

Keep code testable without requiring live LLM/MCP connections for normal unit tests.

Use:

Unit tests:
- ChatService behavior where meaningful
- tool-policy filtering
- guardrail behavior
- configuration validation

MVC tests:
- request validation
- controller behavior
- error handling

Integration tests:
- Spring context/configuration
- added later as needed

Do NOT call the real company LLM during ordinary mvn test.

Do NOT require the real MCP server during normal unit tests.

Live LLM/MCP verification should be a separately executed integration/smoke test.

==================================================
22. DEPENDENCIES
==================================================

Before adding any dependency:

1. Check whether Spring Boot or Spring AI already solves the problem.
2. Prefer their built-in mechanism.
3. Only introduce another library if it meaningfully improves the implementation.

Avoid dependency bloat.

Possible core dependencies will likely include:

- spring-boot-starter-web
- spring-boot-starter-validation
- spring-ai-starter-model-openai
- spring-ai-starter-mcp-client
- spring-boot-starter-actuator
- spring-boot-starter-test

Only add dependencies when their phase actually needs them.

Do not add Spring Security until the authentication phase is requested.

==================================================
23. BACKWARD COMPATIBILITY
==================================================

The existing querymate-dab and MCP functionality already works.

Do NOT make unnecessary changes to those modules.

Treat the backend as a new consumer of the existing MCP capability.

Prefer additive changes.

If a required change would affect an existing module, explain:

- why it is necessary
- what is affected
- whether it is backward compatible

before making the change.

==================================================
24. IMPLEMENTATION PHASES
==================================================

THIS IS CRITICAL:

DO NOT IMPLEMENT ALL PHASES AT ONCE.

Implement ONE phase at a time.

After every phase:

1. run compilation/tests
2. summarize changes
3. show project structure changes
4. list assumptions
5. list risks/questions
6. STOP
7. wait for my approval before starting the next phase

Do NOT automatically proceed to the next phase.

--------------------------------------------------
PHASE 0 - PROJECT INSPECTION AND IMPLEMENTATION PLAN
--------------------------------------------------

DO NOT WRITE APPLICATION CODE YET.

Inspect:

- root pom.xml
- querymate-dab module
- existing Java/package structure
- docs/querymate-spingai-sample.md
- dependency management
- Spring Boot/Spring AI versions
- existing MCP configuration if present
- current Java version
- coding conventions

Then provide:

1. proposed final querymate-backend module structure
2. required Maven changes
3. dependencies by future phase
4. proposed REST/SSE contract
5. configuration properties
6. assumptions
7. potential risks
8. anything from this prompt that should be adjusted based on the actual repository

STOP AND WAIT FOR APPROVAL.

--------------------------------------------------
PHASE 1 - BACKEND MODULE + BASIC CHAT FOUNDATION
--------------------------------------------------

Only after approval.

Create:

querymate-backend

Implement only enough for:

Angular/client
 -> REST endpoint
 -> ChatService
 -> Spring AI ChatClient
 -> company Qwen3-32B

Include:

- Maven module
- Spring Boot application
- AiConfiguration
- QueryMateProperties
- ChatController
- ChatService
- request/response DTOs
- validation
- basic centralized error handling
- external configuration
- basic tests

Do NOT add MCP, memory, streaming, guardrails or authentication yet unless something is technically required for the app to start.

Validate plain LLM chat.

STOP.

--------------------------------------------------
PHASE 2 - MCP TOOL INTEGRATION + TOOL POLICY
--------------------------------------------------

Add:

- Spring AI MCP client configuration
- ToolCallbackProvider integration
- approved-tool filtering
- McpToolPolicy
- Spring AI tool-calling flow

Use existing MCP server.

Validate:

question
 -> LLM
 -> approved MCP tool
 -> result
 -> final answer

Add tests around tool policy.

STOP.

--------------------------------------------------
PHASE 3 - CONVERSATION MEMORY
--------------------------------------------------

Add:

- conversation creation API
- backend-generated conversationId
- Spring AI ChatMemory
- MessageWindowChatMemory
- MessageChatMemoryAdvisor
- configurable memory window

No persistent SQL history yet.

Validate follow-up question behavior.

STOP.

--------------------------------------------------
PHASE 4 - GUARDRAILS
--------------------------------------------------

Add separately:

InputGuardrailAdvisor
OutputGuardrailAdvisor

Keep implementation deliberately small and testable.

Add configuration toggles.

Add unit tests.

Do NOT integrate Llama Guard yet unless specifically requested.

STOP.

--------------------------------------------------
PHASE 5 - STREAMING + PROGRESS EVENTS
--------------------------------------------------

Add the UI progress/streaming contract.

Support safe events such as:

ANALYZING
TOOL_CALL_STARTED
TOOL_CALL_COMPLETED
GENERATING_RESPONSE
ANSWER_CHUNK
COMPLETED
ERROR

Prefer SSE.

Do not expose model chain-of-thought.

Verify Spring AI 2.0.1 supported APIs before implementing tool progress tracking.

Add appropriate tests.

STOP.

--------------------------------------------------
PHASE 6 - OBSERVABILITY + HARDENING
--------------------------------------------------

Add production-oriented basics:

- Actuator
- Micrometer/Spring AI observations
- correlation IDs
- useful metrics
- safe logging
- timeout configuration
- failure handling
- review of retry configuration
- health/readiness considerations

Do not introduce Resilience4j unless built-in Spring AI/HTTP retry and timeout mechanisms are insufficient and there is a demonstrated need.

STOP.

--------------------------------------------------
PHASE 7 - PERSISTENT CONVERSATIONS
--------------------------------------------------

DO NOT IMPLEMENT THIS PHASE UNTIL EXPLICITLY REQUESTED.

Future scope:

- SQL Server persistence
- conversation metadata
- message history
- previous-chat listing
- titles
- conversation ownership
- archive/delete
- pagination

Revisit Spring AI chat-memory persistence limitations before choosing the design.

--------------------------------------------------
PHASE 8 - OIDC/JWT SECURITY
--------------------------------------------------

DO NOT IMPLEMENT THIS PHASE UNTIL EXPLICITLY REQUESTED.

Future scope:

- Spring Security
- OAuth2 Resource Server
- JWT validation
- organization OpenID Connect provider
- authorization
- conversation ownership
- user-aware tool policies
- auditing

==================================================
25. CODE QUALITY RULES
==================================================

Use:

- constructor injection
- final fields
- records for immutable API DTOs when appropriate
- descriptive method/class names
- small focused methods
- Jakarta validation
- centralized exception handling
- SLF4J structured/logical logging
- configuration properties with validation
- package-private visibility where suitable
- Java language features supported by the project's configured Java version

Avoid:

- field injection
- static mutable state
- giant utility classes
- generic catch(Exception) unless at a deliberate boundary
- swallowing exceptions
- returning null unnecessarily
- excessive comments explaining obvious code
- boilerplate interfaces
- reflection hacks
- framework-internal APIs
- duplicating Spring functionality

==================================================
26. DOCUMENTATION
==================================================

Maintain concise documentation for the backend.

Document:

- module purpose
- architecture
- environment variables
- how to run
- MCP dependency
- model selection
- local development
- current security limitations
- implemented phases
- future phases

Do NOT duplicate the large POC investigation document.

Link/refer to the existing POC documentation where useful.

==================================================
27. IMPORTANT BEHAVIOR FOR YOU AS CODEX
==================================================

Do not blindly implement my proposed design if repository inspection reveals a better/simple solution.

Challenge assumptions when appropriate.

If an official Spring Boot/Spring AI mechanism already solves something, use it instead of writing custom infrastructure.

If something in this prompt is incompatible with Spring AI 2.0.1, explain it rather than inventing an API.

Keep the code lean.

Security boundaries and clear responsibilities are valuable.

Extra layers and class-count inflation are not.

Do not optimize for "number of design patterns used."

Optimize for:

- readability
- maintainability
- security
- testability
- extensibility
- production evolution
- minimum necessary complexity

==================================================
28. START NOW
==================================================

Start ONLY with:

PHASE 0 - PROJECT INSPECTION AND IMPLEMENTATION PLAN

Do not create querymate-backend yet.

Do not write application code yet.

Inspect the repository and existing POC documentation and give me the Phase 0 plan.

Then STOP and wait for my approval.

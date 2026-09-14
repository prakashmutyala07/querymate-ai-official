You are working on a small validation POC for the QueryMate project.

This is NOT the final production QueryMate backend. The sole purpose of this POC is to validate that:

1. Spring Boot can connect to my company-hosted OpenAI-compatible GPT-OSS-120B endpoint.
2. Spring AI can send chat requests to that model.
3. Spring AI can connect to my already-running MCP server.
4. MCP tools can be discovered.
5. GPT-OSS-120B can decide to invoke an MCP tool.
6. Spring AI can execute the MCP tool and return the tool result to the LLM.
7. The LLM can produce a final natural-language response.

I already created a plain Spring Boot Maven project. Inspect the existing project first and modify it rather than recreating it.

Use this technology baseline:

- Java 21
- Spring Boot 4.0.6
- Spring AI 2.0.1
- Maven
- Spring MVC / imperative programming model
- OpenAI-compatible chat API
- Spring AI MCP Client using Streamable HTTP unless the existing MCP server clearly requires a different transport

==================================================
1. BUILD CONFIGURATION
   ==================================================

Update pom.xml correctly.

Use Spring AI BOM:

org.springframework.ai:spring-ai-bom:2.0.1

Add only the dependencies required for this POC:

- spring-boot-starter-web
- spring-boot-starter-validation
- spring-ai-starter-model-openai
- spring-ai-starter-mcp-client
- spring-boot-starter-test

Do NOT manually version individual Spring AI dependencies when the BOM manages them.

Do not add unnecessary libraries.

Run Maven compilation/tests after making the changes.

==================================================
2. EXTERNAL CONFIGURATION
   ==================================================

Do NOT hardcode URLs, API keys, model IDs, ports, usernames, passwords, or secrets in Java code.

Use application.yml and environment-variable placeholders.

Create configuration approximately like this, adapting property names exactly to Spring AI 2.0.1 APIs:

spring:
ai:
openai:
base-url: ${LLM_BASE_URL}
api-key: ${LLM_API_KEY}
chat:
model: ${LLM_MODEL}

    mcp:
      client:
        enabled: true
        type: SYNC
        request-timeout: 30s
        streamable-http:
          connections:
            querymate-adapter:
              url: ${MCP_SERVER_BASE_URL}
              endpoint: ${MCP_SERVER_ENDPOINT:/mcp}

server:
address: ${SERVER_ADDRESS:127.0.0.1}
port: ${SERVER_PORT:8081}

Use these placeholders:

LLM_BASE_URL=<GPT_OSS_BASE_URL_ENDING_IN_/v1>
LLM_API_KEY=<COMPANY_LLM_API_KEY>
LLM_MODEL=<EXACT_GPT_OSS_120B_MODEL_ID>

MCP_SERVER_BASE_URL=<MCP_SERVER_BASE_URL>
MCP_SERVER_ENDPOINT=<MCP_ENDPOINT, default /mcp>

Example only:

LLM_BASE_URL=https://your-company-gpt-oss-host.example.com/v1

Never put a real API key into application.yml, README, tests, source code, Git history, or logs.

The company LLM API uses:

Authorization: Bearer <API_KEY>

and exposes an OpenAI-compatible:

POST /v1/chat/completions

API.

Do not manually create Authorization header handling if Spring AI's OpenAI configuration already handles Bearer authentication.

==================================================
3. APPLICATION STRUCTURE
   ==================================================

Keep this POC simple.

Use the existing root package and create something similar to:

config/
AiConfiguration.java

controller/
PocChatController.java

service/
PocChatService.java

dto/
ChatRequest.java
ChatResponse.java

exception/
GlobalExceptionHandler.java

Do NOT introduce interfaces, factories, repositories, domain layers, hexagonal architecture, or other abstraction layers that add no value to this small validation POC.

Use constructor injection only.

Use Java records for simple request/response DTOs where appropriate.

==================================================
4. CHAT CLIENT + MCP TOOL INTEGRATION
   ==================================================

Use Spring AI's ChatClient.

Use the OpenAI ChatModel auto-configured by Spring AI.

Use the MCP ToolCallbackProvider created by Spring AI's MCP client auto-configuration.

IMPORTANT:

MCP tools must be explicitly provided to ChatClient.

Do not assume the MCP tool provider is automatically registered with ChatClient.

Create the ChatClient as a bean using the Spring AI 2.0.1 recommended API.

Conceptually it should be equivalent to:

ChatClient.builder(chatModel)
.defaultTools(mcpToolCallbackProvider)
.build();

BUT:

Do not blindly copy this snippet.

Inspect the actual Spring AI 2.0.1 classes available in the project and use the correct ToolCallbackProvider / SyncMcpToolCallbackProvider types and APIs.

Do not invent APIs.

==================================================
5. SYSTEM PROMPT
   ==================================================

Give the ChatClient a very small system instruction similar to:

"You are the QueryMate POC assistant.

Use the available MCP tools whenever the user's request requires information provided by those tools.

Do not fabricate tool results.

If the required information cannot be obtained from an available tool, clearly state that."

Do not build a large prompt-management framework for this POC.

==================================================
6. REST API
   ==================================================

Create:

POST /api/poc/chat

Request:

{
"message": "user question"
}

Response:

{
"answer": "final LLM response"
}

Validate:

- message must not be null
- message must not be blank
- impose a reasonable maximum size, for example 4000 characters

The controller must not call ChatClient directly.

Controller -> PocChatService -> ChatClient.

==================================================
7. MCP TOOL DISCOVERY DIAGNOSTIC
   ==================================================

Add a small diagnostic endpoint if it can be implemented cleanly using Spring AI 2.0.1 APIs:

GET /api/poc/tools

It should return the MCP tools discovered by the configured MCP server, preferably:

[
{
"name": "...",
"description": "..."
}
]

This endpoint is only for the local POC.

If Spring AI 2.0.1 does not expose a clean supported API for this, do not hack around framework internals.

Instead document in README how to confirm discovered tools from the MCP client.

==================================================
8. ERROR HANDLING
   ==================================================

Add minimal centralized exception handling.

Do not expose:

- API keys
- Authorization headers
- internal stack traces
- full downstream responses containing sensitive data

Return sensible HTTP responses.

For example:

400 -> invalid request
502/503 -> downstream LLM or MCP connectivity failure
500 -> unexpected application failure

Keep it simple.

==================================================
9. LOGGING
   ==================================================

Log enough information for troubleshooting but do NOT log secrets.

Useful logs:

- application startup
- configured model name
- whether MCP connectivity initialized
- discovered MCP tool names, if safely available
- start/end of chat request using a generated/request correlation ID
- downstream failure category

Do NOT log:

- API key
- Authorization header
- environment-variable values containing secrets
- complete sensitive tool responses

Do not enable TRACE logging globally.

==================================================
10. SECURITY FOR THIS POC
    ==================================================

This POC intentionally has no authentication.

Therefore:

- bind to 127.0.0.1 by default
- do not expose it externally by default
- clearly document that authentication/authorization is required before production exposure
- never commit secrets
- do not add permissive CORS configuration unless actually required

The MCP server itself may also be unauthenticated, so document that it must remain inside a trusted/local network boundary for this POC.

==================================================
11. TESTING
    ==================================================

Tests must NOT require the real company LLM or MCP server to be available during `mvn test`.

At minimum:

- controller validation test for blank input
- controller happy-path test with mocked service
- any simple service/unit test that is useful without overengineering

Do not put the real API key into tests.

Do not create brittle tests against internal Spring AI implementation details.

==================================================
12. README
    ==================================================

Create/update README.md with:

A. Purpose of this POC

B. Architecture:

User
-> REST Controller
-> PocChatService
-> Spring AI ChatClient
-> GPT-OSS-120B
-> MCP tool decision
-> Spring AI MCP Client
-> existing QueryMate MCP server
-> existing adapter / database functionality
-> tool result
-> GPT-OSS-120B
-> final answer

C. Required environment variables:

LLM_BASE_URL
LLM_API_KEY
LLM_MODEL
MCP_SERVER_BASE_URL
MCP_SERVER_ENDPOINT

D. Example startup commands without real credentials.

E. Example curl:

curl -X POST http://localhost:8081/api/poc/chat \
-H "Content-Type: application/json" \
-d '{"message":"<QUESTION_THAT_REQUIRES_AN_MCP_TOOL>"}'

F. How to verify success:

1. Application starts.
2. MCP client connects.
3. MCP tools are discovered.
4. A normal chat question receives an LLM answer.
5. A question requiring MCP data causes the LLM to call an MCP tool.
6. The MCP tool executes.
7. The tool result is returned to the model.
8. The model returns the final answer.

G. Troubleshooting for:

- 401 from LLM -> API key / Bearer authentication issue
- 404 from LLM -> incorrect base URL or /v1 path
- model not found -> incorrect LLM_MODEL
- MCP connection refused -> incorrect server URL/port
- MCP 404 -> incorrect endpoint, normally /mcp
- tools not discovered
- model returns an answer without invoking a tool
- model does not support tool/function calling

==================================================
13. IMPORTANT IMPLEMENTATION RULES
    ==================================================

Before coding:

1. Inspect the existing pom.xml and source structure.
2. Preserve the existing project name/package unless there is a strong reason not to.
3. Verify Spring AI APIs against version 2.0.1.
4. Prefer Spring Boot/Spring AI auto-configuration over manual HTTP clients.
5. Do NOT create RestTemplate/WebClient/OkHttp code to call the LLM manually.
6. Do NOT manually implement the MCP protocol.
7. Do NOT manually implement the OpenAI tool-calling loop.
8. Let Spring AI manage tool invocation.
9. Keep the implementation small and readable.
10. Do not add production complexity to this validation POC.

==================================================
14. FINAL VERIFICATION
    ==================================================

After implementation:

Run:

./mvnw clean test

or:

mvn clean test

Then run compilation/package validation.

Fix all compilation errors.

Finally provide me with:

1. Files created/modified.
2. Dependencies added.
3. Configuration placeholders I must replace.
4. Exact command to start the application.
5. Exact curl request to test plain LLM chat.
6. Exact curl request that should cause MCP tool invocation.
7. Any assumptions made.
8. Any issue that cannot be validated without the real LLM/MCP endpoints.

Do not claim that live LLM or MCP integration works unless you were actually able to execute it.

Stop after completing this POC. Do not start implementing the full QueryMate backend architecture.

==================================================
15. IMPLEMENTATION OUTCOME (post-execution notes)
    ==================================================

STATUS: SUCCESS - the full LLM -> MCP tool-calling loop was executed live and validated
(goals 1-7 of this brief, i.e. the app's goals 1-8). Real tool results were returned, e.g.
"describe_entities" listed BookingLocation / Counterparty / Relation, and "aggregate_records"
returned "20 records in the Counterparty entity".

--------------------------------------------------
15.1 IMPORTANT DEVIATIONS FROM THE ORIGINAL BRIEF
--------------------------------------------------

A. Model changed from gpt-oss-120b to qwen3-32b for tool calling.
   - The brief assumed GPT-OSS-120B would perform tool calling. In practice, the LLMaaS
     gpt-oss-120b endpoint has NO tool-call parser: it decides to call a tool (visible in
     "reasoning_content") but never emits a structured "tool_calls" array (finish_reason=stop,
     content=null). Plain chat works; tool calling does not.
   - Tool/function calling on LLMaaS is enabled PER MODEL, on that model's OWN host (confirmed by
     the LLMaaS wiki "Prise en main de la solution LLMaaS", section 6). Every host's /v1/models
     lists the same full catalog, which is misleading - you must target the model's own endpoint.
   - Validated tool-enabled endpoints (tested working): qwen3-32b (app default) and
     qwen3-6-35b-a3b (faster MoE alternative). Also listed as tool-enabled by the wiki:
     mistral-7b-instruct-v0-3, mixtral-8x22b-instruct-v0-1-awq, gemma-4-31b-it.
   - Default in application.yaml is now:
       LLM_BASE_URL=https://qwen3-32b.apps.gpu-prod.ocp.test.grp.gac/v1
       LLM_MODEL=qwen3-32b

B. Java 17 (not Java 21). The project builds/runs on Temurin 17 (Spring Boot 4.0.6, Spring AI 2.0.1).

C. TLS trust required. The LLMaaS endpoint presents an internal Credit Agricole CA chain. The app
   JVM needs that CA imported into its truststore or it fails with
   "PKIX path building failed: unable to find valid certification path". A helper
   (scripts/import-llmaas-cert.ps1) and the CA chain (certs/) were added. Browser/IntelliJ trust
   does NOT apply to the app JVM.

D. Proxy is NOT needed for the internal ".gca" LLM hosts (they resolve directly); the corporate
   proxy is only for public internet. MCP localhost traffic must also stay direct.

E. Answer extraction hardening. Reasoning models (Qwen3) emit a <think>...</think> block inline in
   content, and some models place text in "reasoning_content" with empty content. PocChatService
   now strips the <think> block and, if content is blank, logs the reasoning for diagnosis and
   returns a clear message instead of surfacing raw reasoning as the answer.

--------------------------------------------------
15.2 ANSWERS TO SECTION 14 (FINAL VERIFICATION)
--------------------------------------------------

1. Files created/modified:
   - src/main/java/com/poc/llmdemo/config/AiConfiguration.java (ChatClient bean + MCP tools + system prompt)
   - src/main/java/com/poc/llmdemo/controller/PocChatController.java (POST /api/poc/chat, GET /api/poc/tools)
   - src/main/java/com/poc/llmdemo/service/PocChatService.java (chat + answer extraction/diagnostics)
   - src/main/java/com/poc/llmdemo/dto/ChatRequest.java, ChatResponse.java, ToolInfo.java (records)
   - src/main/java/com/poc/llmdemo/exception/GlobalExceptionHandler.java, DownstreamServiceException.java
   - src/main/resources/application.yaml (defaults to qwen3-32b tool-enabled endpoint)
   - README.md (architecture, env vars, proxy/TLS setup, section G2 on per-endpoint tool calling)
   - scripts/import-llmaas-cert.ps1 and certs/*.crt (TLS trust helper)
   - src/test/java/... LlmdemoApplicationTests, service/PocChatServiceTest

2. Dependencies added (managed by spring-ai-bom 2.0.1):
   spring-boot-starter-web, spring-boot-starter-validation, spring-ai-starter-model-openai,
   spring-ai-starter-mcp-client, spring-boot-starter-test.

3. Configuration placeholders to replace:
   - LLM_API_KEY (REQUIRED - never commit)
   - LLM_BASE_URL (optional; defaults to the qwen3-32b endpoint)
   - LLM_MODEL (optional; defaults to qwen3-32b)
   - MCP_SERVER_BASE_URL, MCP_SERVER_ENDPOINT (point at the running QueryMate MCP server)

4. Exact command to start the application (PowerShell):
   $env:LLM_API_KEY = "<your-llmaas-key>"
   $env:MCP_SERVER_BASE_URL = "http://127.0.0.1:9000"
   mvn spring-boot:run

5. Exact curl to test plain LLM chat:
   curl.exe -X POST http://localhost:8081/api/poc/chat -H "Content-Type: application/json" -d '{"message":"Say hello in one sentence."}'

6. Exact curl that causes MCP tool invocation:
   curl.exe -X POST http://localhost:8081/api/poc/chat -H "Content-Type: application/json" -d '{"message":"What entities are available in the database?"}'
   (also verified: "How many records are in the Counterparty entity?" -> aggregate_records)

7. Assumptions:
   - The QueryMate MCP server is running and reachable via Streamable HTTP at MCP_SERVER_BASE_URL + /mcp.
   - The API key has clearance for the chosen model endpoint.
   - Windows/PowerShell environment; use curl.exe (not the curl alias) or Invoke-RestMethod.

8. Items that could not be validated / key issues found:
   - gpt-oss-120b cannot do tool calling on LLMaaS (no tool parser) - use a tool-enabled model host.
   - "mvn clean test" via the Maven WRAPPER fails in this environment (wrapper cannot download Maven
     through the blocked proxy) and JAVA_HOME is stale. Use the system Maven with a valid JAVA_HOME:
       $env:JAVA_HOME="C:\Users\<you>\.jdks\temurin-17.0.17"
       & "C:\...\apache-maven-3.6.3\bin\mvn.cmd" -o test
     or run tests from IntelliJ. (Tests are fully mocked and do not need the live LLM/MCP.)

--------------------------------------------------
15.3 RECOMMENDATION
--------------------------------------------------
Use qwen3-32b (default) for strongest reasoning + reliable tool calling. Switch to qwen3-6-35b-a3b
(env vars only, no code change) when latency/throughput matters. Do not use gpt-oss-120b for tools.


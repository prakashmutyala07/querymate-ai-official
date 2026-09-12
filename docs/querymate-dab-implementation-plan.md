# QueryMate Data API Builder / SQL Server Implementation Plan

Status: Proposed for phased execution

Scope: `querymate-dab` only; no application implementation is part of this document

Last reviewed: 2026-09-12

## 1. Purpose and target outcome

This plan defines how to build and validate the QueryMate Data API Builder (DAB) and SQL Server layer as an independently runnable, read-only data access service. It is intentionally detailed enough to execute one phase at a time while leaving Spring Boot, Spring AI, and Angular implementation for separate plans.

The target flow is:

```text
Future QueryMate backend / Spring AI
                |
        controlled MCP / GraphQL
                |
        Microsoft Data API Builder
                |
     least-privileged SQL Server user
                |
             SQL Server
```

The following decisions are non-negotiable unless an architecture review changes them:

- DAB and its SQL Server account are read-only.
- An AI caller never receives an arbitrary-SQL capability.
- Every exposed entity, field, relationship, and permission is explicit and reviewed.
- Schema discovery produces candidates; it never publishes configuration automatically.
- SQL Server and DAB assets remain under `querymate-dab`.
- Angular uses Node tooling, the backend uses Maven, and DAB uses its CLI/container; neither Angular nor DAB becomes an artificial Maven module.
- DAB is tested and trusted independently before backend integration.

## 2. Review of the proposed approach

The proposed architecture is appropriate for an enterprise-oriented POC. The following refinements are required during implementation:

1. **Repository path:** the active repository is `/Users/prakashmanasa/Desktop/Learnings/QueryMate-AI-Official`, not the `Spring_AI` path in the original prompt. All future commands and artifacts should use the actual Git root discovered at execution time rather than a hard-coded developer-machine path.
2. **Version pinning:** select and pin one exact supported DAB 2.x CLI version, container tag/digest, and compatible configuration schema in Phase 4. Do not use `latest` in reproducible builds or deployments.
3. **Configuration grouping:** DAB supports top-level `data-source-files`, but relationships cannot cross child configuration files. Domain grouping must therefore follow relationship boundaries. If the schema is highly connected, prefer fewer larger native DAB configuration files over a custom merge framework.
4. **Credential separation:** the runtime DAB account needs only approved data reads. The developer-time schema generator may require catalog visibility; use a separate discovery identity or narrowly grant metadata visibility rather than expanding the runtime account.
5. **Authentication:** anonymous access and the DAB `Simulator` provider are local-only shortcuts. DEV and later environments must use authenticated identities and explicit roles.
6. **Limits:** DAB-native pagination and MCP aggregation timeouts should be configured. Any timeout or payload control not supported by the selected DAB version belongs at the hosting/reverse-proxy or database layer and must be verified rather than assumed.

## 3. Proposed monorepo structure

Only this planning document is created now. The following is the intended structure after the relevant implementation phases:

```text
QueryMate-AI-Official/
|-- .github/
|   `-- workflows/                    # Optional shared CI/CD
|-- docs/
|   `-- querymate-dab-implementation-plan.md
|-- querymate-ui/                     # Angular; native npm/Angular build
|   `-- Dockerfile
|-- querymate-backend/                # Spring Boot/Spring AI; Maven module
|   `-- Dockerfile
|-- querymate-dab/                    # All DAB and SQL Server ownership
|   `-- ...
|-- pom.xml                            # Aggregates Java modules only
|-- docker-compose.yml                # Cross-component local orchestration
|-- .gitignore
`-- README.md
```

The root Compose file may orchestrate SQL Server and DAB, but SQL initialization scripts, DAB configuration, DAB container assets, and schema tooling remain owned by `querymate-dab`.

## 4. Detailed `querymate-dab` structure

```text
querymate-dab/
|-- README.md
|-- Dockerfile
|-- .dockerignore
|-- config/
|   |-- dab-config.json               # Top-level stable runtime configuration
|   |-- domains/                      # Approved explicit entity definitions
|   |   |-- organization.json         # Example only; final groups follow schema/FKs
|   |   `-- projects.json
|   `-- environments/
|       |-- README.md
|       `-- *.example                 # Non-secret environment templates only
|-- database/
|   |-- ddl/                          # Database, schemas, tables, constraints
|   |-- data/                         # Deterministic dummy/test data
|   |-- indexes/                      # Non-constraint indexes
|   |-- security/                     # Logins/users/roles/grants/denies as approved
|   |-- verification/                 # Read/write-negative and integrity checks
|   `-- README.md                     # Order, prerequisites, rollback guidance
|-- tools/
|   `-- schema-generator/
|       |-- README.md
|       |-- src/
|       |-- tests/
|       |-- queries/                  # Versioned SQL metadata queries
|       |-- schemas/                  # Candidate-output schema
|       `-- samples/                  # Sanitized examples
|-- generated/
|   |-- README.md                     # Candidate-only warning and workflow
|   `-- .gitkeep                      # Final retention policy decided in Phase 6
|-- tests/
|   |-- contract/                     # REST/GraphQL/MCP behavior
|   |-- integration/                  # DAB + SQL Server
|   |-- security/                     # Negative authorization/write tests
|   `-- operational/                  # Health, failure, timeout, telemetry tests
`-- scripts/                           # Thin, documented developer commands only
```

Names are provisional until the real schema is reviewed. Do not create one child configuration file per table by default. Each relationship-connected group must remain in the same native DAB child file because DAB does not support relationships across child files.

## 5. Cross-cutting rules

### Security baseline

- Deny by default: objects absent from approved configuration are unavailable, and entities have no access without an explicit permission block.
- Grant DAB only `SELECT` on approved objects, preferably through a dedicated database role. Do not grant `db_datareader` blindly if it would expose excluded schemas/tables.
- Do not grant DAB DDL, DML, stored-procedure execution, ownership, impersonation, or broad metadata access unless separately reviewed.
- Disable MCP create, update, delete, and execute tools globally. Do not expose stored procedures as custom MCP tools during the read-only POC.
- Use explicit field inclusion for sensitive or mixed-sensitivity entities where practical; verify excluded fields through every enabled protocol.
- Never commit credentials, tokens, certificates, complete connection strings, or real production data.
- Use synthetic seed data. Logs and test fixtures must not contain sensitive source records.

### Configuration and versioning baseline

- Record the selected DAB CLI and container versions together and upgrade them through reviewed changes.
- Reference secrets using DAB-supported environment/secret resolution, not checked-in values.
- Treat the top-level runtime configuration as stable and reviewed domain/entity files as evolving configuration.
- Validate the complete resolved configuration for every target environment. Environment variants are alternatives, not implicit merged overlays.
- Keep generated candidates visibly separate from approved configuration.

### Definition of done for every phase

A phase is complete only when its deliverables are committed, automated checks pass where applicable, manual evidence is recorded for non-automatable checks, security acceptance criteria pass, and documentation identifies any remaining limitation. Later phases must not silently waive failed criteria from an earlier phase.

## 6. Implementation phases

### Phase 0 — Confirm inputs, conventions, and architecture baseline

**Objective:** remove avoidable ambiguity before implementation begins.

**Tasks**

- Confirm the repository root, default branch, pull-request policy, CI platform, naming conventions, and ownership/reviewers.
- Obtain authoritative table definitions or legible screenshots, expected relationships, representative query scenarios, and a data-classification owner.
- Record supported local/DEV SQL Server editions and versions, host platforms, and container constraints.
- Create an architecture decision record for explicit entities, read-only defense in depth, protocol choices, and candidate-only generation.
- Create a decision log for DAB version pinning and supported environments.

**Deliverables:** input inventory, assumptions/decision log, architecture decision record, and initial traceability matrix from requirements to phases/tests.

**Acceptance criteria**

- Every supplied table/column is traceable to an authoritative input or explicitly marked unknown.
- Owners exist for data classification, SQL access, security approval, and DAB configuration review.
- No schema is inferred from unreadable screenshots without confirmation.

**Security and testing:** define the classification vocabulary and threat-model scope; verify secret scanning and branch protection expectations before secrets or configs exist.

**Dependencies:** none. Blocks Phases 1, 2, 4, and 6.

### Phase 1 — Create the SQL Server POC schema

**Objective:** translate approved schema inputs into repeatable SQL Server scripts without yet exposing data through DAB.

**Tasks**

- Model database schemas, tables, columns, SQL Server types, nullability, defaults, checks, PKs, FKs, unique constraints, and required indexes.
- Resolve ambiguous types, lengths, precision/scale, temporal behavior, Unicode needs, identity/sequence behavior, and delete/update FK actions with the schema owner.
- Split ordered, rerunnable scripts under `database/ddl`, `database/indexes`, and `database/data`.
- Create deterministic, synthetic seed data covering relationship and boundary cases; never transcribe real sensitive values from screenshots.
- Define clean-database setup and developer reset procedures. Destructive reset must be explicit and local-only.

**Deliverables:** versioned DDL, constraints, indexes, seed scripts, execution-order documentation, and schema diagram or catalog summary.

**Acceptance criteria**

- A clean supported SQL Server instance can be built reproducibly from scripts.
- All scripts fail clearly on invalid prerequisites and produce the documented object set.
- PK/FK/check/unique constraints work; seed rows load without disabling integrity checks.
- Row counts and representative joins match expected fixtures.

**Security and testing:** avoid production data; set explicit database/schema ownership; test script execution from least-privileged deployment context where practical; scan scripts for embedded credentials and sensitive fixtures.

**Dependencies:** Phase 0 and approved schema input. Enables Phases 2, 3, and 6.

### Phase 2 — Create read-only SQL Server security

**Objective:** ensure a database compromise through DAB cannot become a write path.

**Tasks**

- Define a dedicated login/user strategy for local POC and a future managed/enterprise identity strategy.
- Create a dedicated database role granting `SELECT` only on individually approved schemas/objects; prefer grants narrow enough to exclude sensitive objects.
- Create a separate schema-discovery identity if metadata visibility exceeds runtime needs.
- Keep passwords in local environment/secret stores and provide only non-secret templates.
- Add verification scripts for allowed reads and forbidden `INSERT`, `UPDATE`, `DELETE`, `MERGE`, `EXECUTE`, `ALTER`, `CREATE`, `DROP`, ownership changes, and cross-database access.

**Deliverables:** security scripts, grant matrix, credential-handling guide, and automated positive/negative verification suite.

**Acceptance criteria**

- The DAB identity can select only approved objects.
- Every tested write, DDL, procedure execution, and unapproved-object read fails.
- The DAB identity is not a member of broad built-in roles such as `db_owner`, `db_ddladmin`, or `db_datawriter`.
- Credential rotation requires configuration/secret changes, not source changes.

**Security and testing:** capture error codes rather than merely expecting nonzero output; run tests using the actual DAB credential; ensure test cleanup does not require granting extra rights to DAB.

**Dependencies:** Phase 1. Required by Phases 4, 10, and 13.

### Phase 3 — Validate and classify the database schema

**Objective:** verify that the database model is safe and usable before API design.

**Tasks**

- Review PK coverage, composite keys, FKs, cardinality, nullability, uniqueness, checks, default constraints, index selectivity, and likely query paths.
- Identify missing or suspicious relationships; never invent business relationships merely because column names match.
- Classify every table and column: approved, conditional/restricted, internal-only, or prohibited.
- Identify tables/views lacking stable keys, high-cardinality/large-object columns, audit/history tables, and fields likely to leak secrets or personal data.
- Produce a review checklist and issue register; route undocumented business relationships to the data owner.

**Deliverables:** signed schema-review checklist, classification matrix, relationship map, exposure denylist/allowlist proposal, and remediation backlog.

**Acceptance criteria**

- Every candidate table and column has an owner and exposure classification.
- Every relationship is backed by an FK or explicitly marked for manual review.
- Blocking integrity/key issues are resolved before entity generation; nonblocking issues have owners and rationale.
- Expected query patterns have supporting indexes or a recorded follow-up.

**Security and testing:** validate classifications with security/data owners; include indirect inference risks (salary, identifiers, notes, audit data), not just obvious names such as `ssn`.

**Dependencies:** Phase 1; informs Phases 6–10.

### Phase 4 — Set up the independently runnable DAB runtime

**Objective:** establish a reproducible DAB 2.x developer/runtime foundation connected with the read-only SQL identity.

**Tasks**

- Select an exact supported DAB 2.x release; pin CLI/tool manifest, container tag (and digest where supported), and compatible configuration schema.
- Add `querymate-dab/Dockerfile`, `.dockerignore`, minimal config scaffold, and non-secret environment templates.
- Connect through an environment-resolved read-only connection string with appropriate encryption/certificate settings per environment.
- Add DAB and SQL Server services to root Docker Compose without coupling them to Spring Boot.
- Document CLI and container startup, networking, readiness order, configuration validation, and troubleshooting.
- Run `dab validate` and verify the `/health` endpoint against SQL Server.

**Deliverables:** pinned runtime decision, Dockerfile, Compose integration, config scaffold, environment template, and runbook.

**Acceptance criteria**

- DAB starts independently by CLI and container using documented commands.
- `dab validate` succeeds against the selected schema/database.
- `/health` returns healthy only when required data sources are responsive.
- Startup fails safely and diagnostically with a missing secret or unavailable database.
- No secret is present in image layers, Git, command examples, or logs.

**Security and testing:** run the container as non-root where supported, minimize image contents, scan the image, restrict published ports to local interfaces in POC, and test with the Phase 2 identity—not `sa`.

**Dependencies:** Phases 0–2. Enables runtime validation in later phases.

### Phase 5 — Establish stable DAB runtime configuration

**Objective:** separate platform behavior from evolving entity definitions.

**Tasks**

- Configure authentication provider, role model, REST/GraphQL/MCP enablement, host mode, CORS, pagination defaults/maxima supported by the pinned version, and error detail behavior.
- Enable health with restricted operational roles outside local POC; set appropriate cache TTL, parallelism, data-source thresholds, and selected entity checks.
- Configure structured logging and OpenTelemetry endpoint/service metadata through environment settings; prevent payload/record logging.
- Configure MCP DML tools globally: enable `describe-entities`, `read-records`, and `aggregate-records`; disable `create-record`, `update-record`, `delete-record`, and `execute-entity`.
- Set aggregation query timeout. Verify other timeout/body-size/rate controls in the pinned DAB version; place unsupported controls at SQL Server, reverse proxy, or hosting platform.
- Decide whether REST adds a justified operational/user use case. Disable it if not required.
- Document environment variants without secret duplication or an unverified merge assumption.

**Deliverables:** reviewed top-level config, environment matrix, role matrix, limits/timeouts table, telemetry/logging policy, and configuration tests.

**Acceptance criteria**

- Runtime settings remain stable when a new approved entity is added.
- Only intentionally enabled protocols and MCP tools are discoverable.
- Page/aggregation limits constrain large requests, and malformed requests fail safely.
- Production-like environments do not use anonymous or simulator authentication.
- Configuration changes are validated by the pinned CLI.

**Security and testing:** test CORS allowlists, authentication failures, role selection, safe error responses, disabled endpoints/tools, excessive results, malformed filters, and telemetry redaction.

**Dependencies:** Phase 4; supports Phases 9–15.

### Phase 6 — Implement the schema discovery/candidate generator

**Objective:** automate metadata collection without automating approval or publication.

**Tasks**

- Choose the smallest maintainable implementation stack already supported by the team; document its runtime and dependency policy.
- Accept a connection via secret/environment reference plus optional schema/table allowlists; never accept or emit credentials in output.
- Query SQL Server catalogs including `sys.schemas`, `sys.tables`, `sys.columns`, `sys.types`, `sys.key_constraints`, `sys.indexes`, `sys.index_columns`, `sys.foreign_keys`, and `sys.foreign_key_columns`.
- Capture schemas, tables, columns, native types, lengths/precision/scale, nullability, defaults where useful, PK/unique keys, FKs, and review-relevant indexes.
- Define a deterministic, versioned intermediate candidate schema, canonical ordering, source database fingerprint, generation timestamp kept outside semantic comparison, and tool version.
- Add `--check`/diff behavior that reports schema drift without writing approved config.
- Produce warnings for keyless objects, unsupported types, composite relationships, untrusted/disabled FKs, ambiguous names, and metadata access failures.
- Ensure the output path is candidate-only and the tool has no code path that overwrites `config/domains`.

**Deliverables:** generator source, metadata queries, candidate JSON Schema (or equivalent), CLI contract, sanitized samples, tests, and usage/security documentation.

**Acceptance criteria**

- Repeated runs against unchanged metadata are semantically identical.
- Unit tests cover mapping/naming/error behavior; integration tests cover representative SQL Server features.
- Composite keys and FKs retain column order and direction correctly.
- Partial discovery failures exit nonzero and do not masquerade as a complete candidate set.
- Approved configuration cannot be overwritten by normal generator operation.

**Security and testing:** use a separate least-privileged metadata identity; sanitize logs; test hostile/unusual identifiers and output escaping; perform dependency and secret scans.

**Dependencies:** Phases 1 and 3; the runtime itself is not required. Enables Phase 7.

### Phase 7 — Generate candidate entities and relationships

**Objective:** convert normalized metadata into reviewable DAB-shaped candidates without granting exposure.

**Tasks**

- Map each candidate to entity name, fully qualified source, source type, fields, type/nullability metadata, PK/key fields, and FK-derived relationships.
- Generate deterministic candidate relationship names with collision warnings; do not infer relationships without constraints.
- Add review placeholders for descriptions, exposure status, field classifications, permissions, and domain assignment.
- Show changed/added/removed database objects relative to the prior candidate snapshot.
- Validate candidate documents against their own schema; optionally check their structural compatibility with the pinned DAB schema without turning them into runtime config.

**Deliverables:** candidate entity set, relationship report, drift report, warnings report, and candidate validation results.

**Acceptance criteria**

- Candidate output accounts for all in-scope discovered tables and columns.
- PKs/FKs match SQL metadata exactly; unsupported/ambiguous cases are warnings, not fabricated configuration.
- Naming is repeatable and collisions are explicit.
- No candidate is reachable through the running DAB instance.

**Security and testing:** candidate output is treated as potentially sensitive metadata; confirm it contains no values or connection details; test exclusion/allowlist inputs and prohibited-schema handling.

**Dependencies:** Phase 6 and current Phase 3 classifications. Enables Phase 8.

### Phase 8 — Perform manual entity and security review

**Objective:** turn candidates into explicitly approved exposure decisions.

**Tasks**

- For every table, decide expose/reject/defer, owner, domain, public entity name, and purpose/description.
- For every field, decide include/exclude, sensitivity, alias where supported and helpful, and field description.
- Review every FK-derived relationship, cardinality, naming, and navigation requirement.
- Assign roles/actions/policies and decide protocol enablement per entity.
- Require data-owner and security approval for conditional fields; use four-eyes review for all approved config.
- Record rejected objects so future regeneration does not repeatedly reopen settled decisions without schema/policy change.

**Deliverables:** signed exposure matrix, approved entity specifications, approved relationship map, permission matrix, and rejection/defer register.

**Acceptance criteria**

- No entity or field remains implicitly approved.
- Every exposed item has a business purpose and description suitable for tool/schema discovery.
- Prohibited fields/tables are excluded; conditional exposure has explicit policy and owner approval.
- Every permission is read-only and mapped to a defined role.

**Security and testing:** conduct privacy/security review and abuse-case walkthroughs, including inference through aggregates, relationships, filters, and error messages.

**Dependencies:** Phases 3 and 7. Enables Phases 9 and 10.

### Phase 9 — Organize approved entity configuration

**Objective:** create readable native DAB configuration for 20–30+ evolving tables.

**Tasks**

- Build a relationship graph and identify connected components/business domains.
- Group strongly related entities into native child data-source files referenced by the top-level `data-source-files` setting.
- Keep both ends of every configured relationship in the same child file; entity names must be globally unique.
- Keep runtime settings only in the top-level file because child runtime settings are not used.
- Prefer a small number of coherent files. If cross-domain relationships create one large connected set, accept a larger file rather than introducing a custom merger.
- Document naming, ordering, ownership, review, and schema-drift update conventions.

**Deliverables:** approved domain configs, top-level references, configuration ownership map, and maintenance guide.

**Acceptance criteria**

- Full configuration passes `dab validate` against the database.
- Every relationship target resolves within its child file.
- No globally duplicated entity name or unapproved candidate appears.
- Adding an unrelated entity does not require redesigning stable runtime settings.

**Security and testing:** diff generated candidates against approved config to detect accidental additions; require security review for domain-file changes; verify excluded objects remain absent.

**Dependencies:** Phases 5 and 8. Enables Phases 10–12.

### Phase 10 — Configure DAB read-only authorization

**Objective:** make DAB itself a second enforced read-only boundary.

**Tasks**

- Add explicit `read` permissions per entity and role; never use wildcard action permissions.
- Configure field include/exclude rules from the approved exposure matrix.
- Configure database/request policies only for reviewed row restrictions; document SQL Server RLS as a future/conditional deeper control.
- Disable an entity on protocols where it is not needed.
- Verify role inheritance/effective-role behavior for the pinned version; do not assume permissions from multiple roles combine.
- Align DAB roles with future JWT/enterprise identity claims without hard-coding a specific provider prematurely.

**Deliverables:** approved authorization config, role-to-entity/field matrix, row-policy extension design, and authorization test suite.

**Acceptance criteria**

- Anonymous access exists only if explicitly accepted for local POC and cannot be enabled accidentally in DEV/production profiles.
- Each role reads exactly its approved entities/fields/rows and no others.
- DAB exposes no create/update/delete/execute action even if the database were misconfigured.
- SQL read-only negative tests from Phase 2 still pass.

**Security and testing:** test direct field selection, nested GraphQL selection, filters/order clauses on excluded fields, MCP descriptions, role spoofing, missing/invalid tokens, and policy boundary rows.

**Dependencies:** Phases 2, 8, and 9. Enables protocol/security testing.

### Phase 11 — Configure and validate MCP

**Objective:** prove safe, structured AI-facing read and aggregation operations.

**Tasks**

- Enable MCP transport/settings supported by the pinned DAB release.
- Confirm only `describe_entities`, `read_records`, and `aggregate_records` are registered globally and only for approved entities.
- Test descriptions, structured filters, sorting if supported, pagination/continuation, aggregation (`count`, `sum`, `avg`, `min`, `max` as applicable), grouping, `having`, and aggregation timeout.
- Test roles, row policies, field restrictions, malformed calls, unavailable SQL Server, and result limits.
- Document that MCP calls operate on configured entities and do not provide arbitrary joins or arbitrary SQL.
- Document when callers should use GraphQL for reviewed relationship traversal.

**Deliverables:** MCP config, protocol contract/examples, automated integration/security tests, limitation guide, and test evidence.

**Acceptance criteria**

- Tool discovery shows only approved tools, entities, operations, and fields for the effective role.
- Write and execute tools are absent, not merely expected to fail.
- Reads/aggregations honor pagination, policies, field restrictions, and timeouts.
- No parameter can escape the structured DAB operation into arbitrary SQL.

**Security and testing:** include prompt/tool misuse scenarios, injection-like filter strings, oversized calls, inference-sensitive aggregates, authentication failures, and redacted observability checks.

**Dependencies:** Phases 5, 9, and 10.

### Phase 12 — Configure and validate GraphQL relationships

**Objective:** prove relationship-heavy, nested read scenarios before backend integration.

**Tasks**

- Configure approved one-to-one, one-to-many, many-to-one, and justified many-to-many relationships.
- Test nested reads, multiple nesting levels, filtering, sorting, pagination, aggregation where useful, null relationships, and composite keys.
- Add negative tests for missing targets, reversed mappings, incompatible fields, cross-file relationships, cyclic/deep queries, and unauthorized nested fields.
- Define representative business queries and measure generated request/database performance.
- Consider a reviewed database view only when DAB entity/relationship modeling cannot express a proven read use case; do not shift joins into Spring Boot by default.

**Deliverables:** relationship config, GraphQL contract/examples, automated tests, performance findings, and documented limitations.

**Acceptance criteria**

- All approved relationship scenarios return correct fixtures and obey authorization at every nested entity.
- Invalid relationships fail validation before deployment.
- Page/result controls prevent unbounded traversal.
- Representative queries meet provisional POC latency/resource targets or have documented remediation.

**Security and testing:** test unauthorized traversal, excluded-field introspection/selection, expensive query shapes, errors, and aggregation inference boundaries.

**Dependencies:** Phases 9 and 10; can run in parallel with Phase 11 once prerequisites pass.

### Phase 13 — Execute defense-in-depth security testing

**Objective:** demonstrate with evidence that no supported path bypasses read-only and exposure controls.

**Tasks**

- Build a security matrix across direct SQL, REST if enabled, GraphQL, MCP, health, and administrative/diagnostic surfaces.
- Prove SQL `INSERT`/`UPDATE`/`DELETE`/DDL/execute fail under the DAB identity.
- Prove DAB mutations fail or are absent across all enabled protocols.
- Prove sensitive fields, rejected entities, and unauthorized rows/roles are inaccessible through selection, filtering, ordering, aggregation, relationships, introspection, and errors.
- Test excessive data requests, malformed inputs, injection payloads, brute-force/rate scenarios, secret leakage, image contents, logs, traces, and repository history.
- Threat-model the DAB trust boundary and track findings by severity and owner.

**Deliverables:** threat model, automated security suite, evidence report, vulnerability/remediation register, and security sign-off.

**Acceptance criteria**

- All required negative cases fail closed with safe responses.
- No critical/high finding remains open for POC handover.
- No secret or sensitive fixture exists in Git history, images, logs, traces, or build artifacts.
- Limits constrain abusive requests and recovery is documented.

**Security and testing:** this phase is the consolidated security gate; tests must run with real role configurations and the actual runtime SQL credential.

**Dependencies:** Phases 10–12. Blocks Phase 16.

### Phase 14 — Validate observability and operations

**Objective:** make failures diagnosable without exposing data.

**Tasks**

- Configure structured logs with environment, service version, request/correlation context, outcome, and duration where supported.
- Export OpenTelemetry traces and metrics for REST, GraphQL, MCP, database calls, middleware, errors, request duration, and active requests to a local/test collector.
- Configure `/health` roles, caching, parallelism, data-source thresholds, and selected entity probes so health checks remain bounded.
- Exercise database unavailable/slow/authentication failure, DAB invalid config/startup failure, telemetry backend unavailable, timeout, and graceful shutdown scenarios.
- Define dashboards/alerts and operational runbooks for DEV/production evolution.

**Deliverables:** telemetry config, local validation setup, dashboard/alert specification, failure runbooks, and operational test evidence.

**Acceptance criteria**

- Operators can distinguish startup/config, authentication, SQL connectivity, query timeout, and caller errors.
- Traces correlate a request/tool call to its database operation without recording result data or secrets.
- Health is useful and bounded; telemetry-backend failure does not break data service behavior.
- Shutdown allows a reasonable telemetry flush window.

**Security and testing:** verify log/trace redaction with canary secrets and sensitive fixture fields; restrict detailed health/diagnostics to operational roles outside local POC.

**Dependencies:** Phases 4, 5, 11, and 12. Blocks Phase 16.

### Phase 15 — Add CI/CD and configuration validation

**Objective:** reject invalid, unsafe, or unreproducible changes before deployment.

**Tasks**

- Add fast checks for JSON/schema formatting, candidate schema, documentation links, generated drift, and secret scanning.
- Run the pinned `dab validate` against an ephemeral representative SQL Server because validation includes connectivity/entity metadata checks.
- Build and scan the pinned DAB image; verify its effective user, exposed ports, and absence of secrets.
- Run SQL migrations/setup, integrity checks, DAB contract tests, and security negative tests in an isolated environment.
- Require reviewed approval for changes under approved config/security paths; publish test evidence and version metadata.
- Define promotion using immutable images plus external environment secrets/configuration; include rollback and compatibility checks.

**Deliverables:** CI workflow, ephemeral integration environment, quality gates, artifact/version manifest, promotion/rollback guide, and ownership rules.

**Acceptance criteria**

- Invalid config, missing entity source, broken relationship, failed security test, secret finding, vulnerable image above policy, or failed image build blocks promotion.
- CI uses no production credentials/data and destroys isolated resources after the run.
- Rebuilding the same revision uses the same pinned DAB/tool versions.
- Approved configuration cannot be replaced directly by unreviewed generator output.

**Security and testing:** use short-lived CI identities/secrets, minimal permissions, protected environments, signed/provenanced artifacts when available, and retained audit evidence.

**Dependencies:** automate each earlier phase as its checks mature; final gate depends on Phases 4–14.

### Phase 16 — Complete DAB layer and hand over to backend integration

**Objective:** establish a trusted, versioned contract for the future Spring AI/backend work.

**Tasks**

- Execute the complete acceptance suite from a clean environment.
- Publish endpoint/tool/schema contracts, role/auth expectations, examples, limits, timeout behavior, error model, and known limitations.
- Record entity/config version and compatibility/change policy for backend consumers.
- Provide local/DEV runbooks, dependency/startup order, operational ownership, and escalation paths.
- Hold architecture, data-owner, security, operations, and backend-consumer readiness review.

**Deliverables:** release candidate, signed readiness checklist, API/MCP/GraphQL contract pack, test/security evidence, operations handover, and deferred-work backlog.

**Acceptance criteria**

- DAB works independently of Spring Boot from clean setup through representative queries.
- Required MCP and GraphQL reads pass; writes and unapproved access fail at both DAB and SQL layers.
- Health, telemetry, failure handling, limits, and CI gates pass.
- Backend integration requires only the documented authenticated contract—no database credential and no arbitrary SQL path.
- Required stakeholders approve handover and all residual risks are explicitly accepted.

**Security and testing:** rerun secret/history scan, dependency/image scan, least-privilege verification, protocol authorization suite, and representative load/resilience tests.

**Dependencies:** all prior phases, with Phases 13–15 as hard gates.

## 7. Phase dependency and execution summary

```text
Phase 0
  |-- Phase 1 -- Phase 2 ------------------------|
  |      `------ Phase 3 -- Phase 6 -- Phase 7 -- Phase 8
  `-------------- Phase 4 -- Phase 5 --------------------|
                                                         v
                                                     Phase 9
                                                         |
                                                     Phase 10
                                                      /     \
                                             Phase 11       Phase 12
                                                      \     /
                                              Phase 13 and Phase 14
                                                         |
                                                     Phase 15
                                                         |
                                                     Phase 16
```

Phases 11 and 12 can proceed in parallel after Phase 10. CI work in Phase 15 should begin incrementally rather than waiting until the end, but the complete pipeline is a handover gate.

## 8. Testing strategy

| Layer | Required tests |
|---|---|
| SQL schema | Clean build, repeatability, constraints, keys, relationships, indexes, seed integrity, supported SQL Server versions |
| SQL security | Approved reads succeed; DML, DDL, execute, unapproved objects, cross-database access fail |
| Generator | Metadata mapping, deterministic output, composite keys/FKs, drift, odd identifiers, partial failure, no config overwrite |
| DAB config | JSON/schema validation, live metadata validation, environment resolution, relationship integrity, pinned-version compatibility |
| Authorization | Roles, missing/invalid JWT, entity/field/row restrictions, nested access, introspection/discovery, protocol parity |
| MCP | Tool discovery, reads, filters, pagination, aggregates/groups, timeout, malformed/oversized calls, writes absent |
| GraphQL | Relationship cardinalities, nested reads, filters/sort/pagination, authorization, invalid and expensive queries |
| REST (if enabled) | Projection, filter/sort/page behavior, role/field restrictions, allowed methods, malformed requests |
| Operations | Health, slow/down SQL, startup/config failure, telemetry export/downstream failure, safe logs, graceful shutdown |
| Supply chain | Secret/dependency/image scans, pinned/reproducible versions, artifact provenance where available |

Tests should use synthetic data designed to make authorization leakage obvious—for example, rows owned by different principals and sentinel prohibited fields. Negative tests are first-class acceptance evidence.

## 9. Key risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Screenshots omit constraints or type detail | Incorrect schema and relationships | Require authoritative confirmation; track unknowns; never silently infer |
| Runtime account is overprivileged | DAB flaw/misconfiguration can modify or expose data | Object-level `SELECT`, separate discovery identity, automated SQL negative tests |
| Candidate output is mistaken for approval | Sensitive object becomes exposed | Separate directories/formats, approval status, protected config paths, no overwrite path |
| Domain files split related entities | Invalid/unavailable relationships | Build relationship graph first; keep targets together; validate live configuration |
| Floating DAB versions change behavior | Non-reproducible or insecure builds | Pin CLI, schema, image tag/digest; controlled upgrade tests |
| Anonymous POC settings escape to DEV | Unauthorized access | Environment gates, authenticated DEV profile, CI assertions, no permissive defaults |
| Large/nested AI-generated requests overload SQL | Availability/cost problem | Pagination/result caps, aggregation timeout, query/index testing, proxy/host limits, monitoring |
| Sensitive data leaks indirectly | Privacy/security incident | Classification plus inference review; field/row restrictions; aggregate/nested tests; redaction |
| Missing FKs hide business relationships | Incomplete query capability | Flag for owner review; add approved constraints/views only with clear semantics |
| Custom configuration tooling grows prematurely | Maintenance burden and drift | Use native DAB multi-file support; accept fewer larger files; revisit only with evidence |
| DAB feature assumptions are version-specific | Invalid plan/config | Verify pinned-version Microsoft docs/schema during each implementation phase |
| Health/telemetry exposes internals | Reconnaissance or data leak | Operational roles, bounded probes, safe details, no record/payload logging |

## 10. Acceptable POC shortcuts

The following are acceptable only when documented and prevented from silently becoming production defaults:

- A local SQL Server container or developer instance with synthetic data.
- SQL authentication for local development when secrets stay outside Git; DEV should test the intended enterprise identity path early.
- DAB `Unauthenticated` or `Simulator` authentication on loopback-only local environments.
- A local OpenTelemetry collector and simple trace UI rather than enterprise monitoring.
- Manual data/security approval recorded in repository documentation before a formal governance workflow exists.
- Provisional latency/load thresholds based on representative POC queries.
- REST disabled or minimally enabled until a concrete consumer requires it.
- One or a few larger entity configuration files when relationship boundaries make finer domain separation invalid.

Shortcuts may simplify infrastructure, not the read-only controls, explicit exposure review, secret handling, or negative security tests.

## 11. Changes required before production

- Replace local/anonymous authentication with approved enterprise identity/JWT validation, explicit role claims, rotation, and workload identity where possible.
- Integrate an approved secret manager; eliminate long-lived SQL passwords where supported.
- Validate network isolation, TLS/certificate trust, private connectivity, ingress/WAF/reverse-proxy controls, rate limits, payload limits, and denial-of-service protections.
- Establish formal data classification, privacy/legal review, access recertification, segregation of duties, and auditable approvals.
- Define SQL HA/DR, backups, restore testing, patching, capacity, connection pooling, and performance baselines.
- Add production SLOs, dashboards, alerting, log retention/redaction/access policies, trace sampling, and incident runbooks.
- Add load, soak, concurrency, failover, recovery, penetration, and dependency/supply-chain testing.
- Sign and attest immutable images, generate an SBOM, enforce vulnerability policy, and define upgrade/rollback cadence.
- Define backward-compatible entity/schema change policy and consumer contract/version management.
- Evaluate row-level security and database policies against real tenancy/authorization requirements.
- Reassess whether REST is needed and minimize the exposed surface.
- Complete environment-specific hardening and production readiness review.

## 12. Explicitly deferred decisions

These decisions should not block the plan today, but must be resolved in the named phase:

| Decision | Resolve by | Trigger/input |
|---|---:|---|
| Exact DAB 2.x CLI/image/schema version | Phase 4 | Current supported release and organization policy |
| Final database/table/domain names | Phase 1/3 | Authoritative schema inputs |
| Schema generator language/runtime | Phase 6 | Team supportability and existing stack |
| Whether generated candidate metadata is committed or CI-only | Phase 6 | Sensitivity, review workflow, diff value |
| Final domain-file boundaries | Phase 9 | Actual FK/relationship graph |
| REST enabled or disabled | Phase 5 | Proven non-AI consumer/operational need |
| Enterprise identity provider and claim mapping | Before DEV deployment | Organization identity architecture |
| DAB database policy versus SQL Server RLS | Phase 10 / pre-production | Row-level authorization requirements and threat model |
| Views/read-only stored procedures | After Phase 12 evidence | A specific query DAB cannot model acceptably |
| Hosting platform and external gateway controls | Pre-production | Enterprise platform standards and scale |
| Exact SLOs, rate limits, page sizes, and timeouts | Phase 5 then production review | Data volume and measured workload |

## 13. Open questions

Only the following inputs are genuinely required before their dependent phases can complete:

1. What are the authoritative SQL Server table definitions, keys, relationships, and data classifications behind the forthcoming screenshots?
2. Which SQL Server edition/version must local development and company DEV support?
3. Who approves entity/field exposure and conditional sensitive fields?
4. Which enterprise identity provider, token issuer/audience, and role claims are expected in DEV/production?
5. What representative MCP/GraphQL questions and provisional data-volume/latency targets define POC success?

## 14. Current official DAB references

Implementation must re-check these references against the pinned release rather than copying examples blindly:

- [Data API Builder documentation](https://learn.microsoft.com/en-us/azure/data-api-builder/)
- [DAB CLI configuration validation](https://learn.microsoft.com/en-us/azure/data-api-builder/command-line/dab-validate)
- [Multiple data sources and configuration files](https://learn.microsoft.com/en-us/azure/data-api-builder/concept/config/multi-config)
- [Authorization overview](https://learn.microsoft.com/en-us/azure/data-api-builder/concept/security/authentication-local)
- [SQL MCP Server overview](https://learn.microsoft.com/en-us/azure/data-api-builder/mcp/overview)
- [MCP DML tools and aggregation](https://learn.microsoft.com/en-us/azure/data-api-builder/mcp/data-manipulation-language-tools)
- [GraphQL aggregation](https://learn.microsoft.com/en-us/azure/data-api-builder/how-to/aggregate-data)
- [Health checks](https://learn.microsoft.com/en-us/azure/data-api-builder/concept/monitor/health-checks)
- [OpenTelemetry](https://learn.microsoft.com/en-us/azure/data-api-builder/concept/monitor/open-telemetry)
- [Run DAB in a container](https://learn.microsoft.com/en-us/azure/data-api-builder/deployment/local-container)

## 15. Recommended first execution checkpoint

Begin implementation with Phase 0 only. Its review should approve the actual schema inputs, classification ownership, SQL Server target, DAB version-selection criteria, and representative query scenarios. Then execute Phases 1–3 before defining any approved DAB entity configuration. This order prevents API exposure decisions from being built on guessed schema details.

# QueryMate Data API Builder / SQL Server Implementation Plan

Status: Proposed for phased execution

Scope: QueryMate Data API Builder (DAB) and SQL Server planning only. This document does not implement the application.

## 1. Goal

Build a small, independently runnable, read-only DAB layer that can later be used by the QueryMate backend through MCP and/or GraphQL.

```text
Future QueryMate backend / Spring AI
                |
        controlled MCP / GraphQL
                |
        Microsoft Data API Builder
                |
       read-only SQL Server user
                |
             SQL Server
```

The POC should be simple. Add structure only when the current solution becomes difficult to maintain. Production concerns are recorded so the design does not create a dead end, but production infrastructure will not be built during the POC without a concrete need.

## 2. Agreed architecture

- DAB is read-only, and its SQL Server account is also read-only.
- The future AI/backend cannot send arbitrary SQL.
- Only explicitly reviewed entities, fields, relationships, and permissions are exposed.
- Schema discovery produces candidates for human review; it never changes approved DAB configuration automatically.
- DAB remains independently runnable from Spring Boot.
- Angular uses Node, Spring Boot uses Maven, and DAB uses the DAB CLI/container.
- Sensitive tables and fields are excluded by explicit review.
- Foreign keys are the primary source for relationships. Missing business relationships are flagged, not invented.

### Schema-generator placement

**The schema generator is a developer utility, not part of DAB or any other application component. It must live outside `querymate-dab`, must not be copied into an application image, and must not run as a production or Docker Compose service.**

It runs manually or in a developer/CI workflow, reads SQL Server metadata, and writes candidate output to a temporary or ignored location. A developer reviews the output and manually transfers only approved definitions into `querymate-dab/dab-config.json`.

## 3. Minimal proposed repository structure

This is the target POC structure, not a requirement to create every file immediately:

```text
QueryMate-AI-Official/
|-- docs/
|   `-- querymate-dab-implementation-plan.md
|-- querymate-ui/                     # Future Angular application
|-- querymate-backend/                # Future Spring Boot/Spring AI application
|-- querymate-dab/
|   |-- README.md
|   |-- Dockerfile
|   |-- dab-config.json
|   |-- .env.example                  # Placeholders only; never real secrets
|   `-- database/
|       |-- 01-schema.sql
|       |-- 02-seed-data.sql
|       |-- 03-readonly-security.sql
|       `-- verify.sql
|-- schema-generator/                 # Developer utility; not an application
|   |-- README.md
|   `-- <small set of source files>   # Exact files depend on chosen language
|-- docker-compose.yml                # Local SQL Server + DAB when needed
|-- pom.xml                            # Java modules only
|-- .gitignore
`-- README.md
```

### Why this is enough for the POC

- Four SQL files are easier to understand than separate `ddl`, `data`, `indexes`, `security`, and `verification` folders.
- One `dab-config.json` is the default. Do not create domain files until its size causes a real maintenance problem.
- Do not create a top-level automated-test folder now. Keep SQL checks in `verify.sql` and initially use a small documented DAB smoke test.
- Do not create `src`, `tests`, `queries`, `schemas`, or `samples` subfolders inside the generator unless its implementation actually needs them.
- Do not add wrapper frameworks, configuration-merging code, or abstractions for possible future scale.

If DAB configuration later becomes hard to review, native `data-source-files` may split it into a few logical groups. Before splitting, verify the selected DAB version because relationships cannot cross child configuration files and entity names must remain globally unique.

## 4. Implementation principles

1. **Start with the minimum:** create only the artifact needed by the current phase.
2. **Keep tests close to behavior:** a verification SQL file and small protocol smoke test are enough initially.
3. **Do not scaffold the future:** an empty folder or unused abstraction has no POC value.
4. **Pin versions:** choose one exact supported DAB 2.x CLI/container version instead of `latest`.
5. **Keep secrets external:** use environment variables or an approved secret store.
6. **Deny by default:** an entity or role without explicit permission is not exposed.
7. **Separate generation from approval:** candidate output cannot overwrite runtime configuration.
8. **Add complexity from evidence:** split files or add infrastructure only when a measured problem justifies it.

## 5. Implementation phases

### Phase 0 — Confirm inputs

**Tasks:** obtain readable table definitions; confirm uncertain types, lengths, nullability, keys, and relationships; identify the exposure approver; confirm the SQL Server version and representative POC questions.

**Deliverables:** confirmed inputs, genuine open questions, and named data/security approver.

**Acceptance criteria:** no schema detail is silently guessed; unknowns have an owner; initial scope is agreed.

**Security/testing:** classify obviously sensitive fields before creating synthetic seed data.

**Dependencies:** none. Phase 1 waits for adequate schema input.

### Phase 1 — Create the SQL Server POC schema

**Tasks**

- Add tables, columns, types, PKs, FKs, constraints, and justified indexes to `01-schema.sql`.
- Add deterministic synthetic data to `02-seed-data.sql`.
- Document execution order in `querymate-dab/README.md`.
- Keep files readable; split only if size later becomes a real problem.

**Deliverables:** the two SQL files and short setup instructions.

**Acceptance criteria:** scripts build a clean database; seed data loads with constraints enabled; representative joins return expected rows.

**Security/testing:** no copied production data, credentials, or unnecessary sensitive columns. Test PK, FK, nullability, unique, and check constraints.

**Dependencies:** Phase 0.

### Phase 2 — Create SQL Server read-only security

**Tasks**

- Create a dedicated DAB login/user and role in `03-readonly-security.sql`.
- Grant only required `SELECT` access. Avoid broad roles if they expose unwanted objects.
- Keep passwords outside SQL and Git.
- Add checks to `verify.sql` proving reads succeed and DML, DDL, and procedure execution fail.

**Deliverables:** read-only security SQL and verification SQL.

**Acceptance criteria:** approved reads succeed; writes, DDL, execution, and unapproved-object reads fail using the real DAB identity.

**Security/testing:** do not grant `db_owner`, `db_ddladmin`, or `db_datawriter`. The generator may later use a separate metadata-only identity.

**Dependencies:** Phase 1.

### Phase 3 — Review schema and exposure scope

**Tasks:** check keys, relationships, constraints, nullability, and useful indexes; mark every table/field expose, exclude, or needs-decision; flag missing FKs; record sensitive and internal-only data.

**Deliverables:** concise schema/exposure review table.

**Acceptance criteria:** every in-scope table/field has a decision; relationships are backed by FKs or clearly marked for review.

**Security/testing:** consider indirect sensitivity such as salary, notes, identifiers, audit data, and aggregate inference.

**Dependencies:** Phase 1. Informs Phases 6–10.

### Phase 4 — Set up DAB runtime

**Tasks**

- Pin one supported DAB 2.x CLI and container version.
- Create minimal `dab-config.json`, Dockerfile, `.env.example`, and run instructions.
- Read the SQL connection string from an environment variable.
- Add SQL Server and DAB to Compose only if it improves the local workflow.
- Run `dab validate`, start DAB independently, and check `/health`.

**Deliverables:** minimal runnable DAB files and README commands.

**Acceptance criteria:** DAB runs without Spring Boot; validation passes; health reflects database availability; no secret is committed or included in the image.

**Security/testing:** connect with the Phase 2 account, not an administrator account; bind local ports conservatively.

**Dependencies:** Phases 1 and 2.

### Phase 5 — Configure the stable DAB foundation

**Tasks**

- Configure authentication, GraphQL, MCP, and REST only if REST has a concrete use.
- Configure pagination/result limits, safe errors, health, logging, and supported OpenTelemetry.
- Enable MCP entity description, reads, and aggregation; disable create, update, delete, and execute.
- Set the MCP aggregation timeout and document any necessary host/database limit.

**Deliverables:** stable runtime settings in `dab-config.json` and a short settings table.

**Acceptance criteria:** only intended protocols/tools are available; large or malformed requests are constrained; shared environments cannot silently use local anonymous/simulator settings.

**Security/testing:** verify disabled tools/endpoints, authentication failures, page limits, timeouts, safe errors, and log redaction.

**Dependencies:** Phase 4.

### Phase 6 — Build the external schema generator

**Tasks**

- Implement a small developer command in root-level `schema-generator` using a team-supported language.
- Read its connection string from an environment variable.
- Query SQL Server metadata for schemas, tables, columns, types, nullability, PKs, FKs, unique constraints, and useful indexes.
- Produce deterministic candidate JSON or Markdown containing metadata only, not row data.
- Warn about keyless objects, unsupported types, composite keys, and relationships missing constraints.
- Write output to a temporary/ignored path and require human review.

**Deliverables:** generator source, one README, an example only if useful, and a basic repeatability check.

**Acceptance criteria:** unchanged schemas produce equivalent output; PK/FK direction is correct; failures are clear; the utility cannot overwrite `dab-config.json`.

**Security/testing:** use a separate metadata-only identity if needed; never log its connection string; never include the utility in application builds, images, or runtime services.

**Dependencies:** Phases 1 and 3. It does not depend on DAB runtime.

### Phase 7 — Generate candidate entities and relationships

**Tasks:** generate candidate names, qualified sources, fields, keys, FK relationships, and relationship names; mark descriptions/exposure/permissions for review; report schema changes; never infer joins or publish output to DAB.

**Deliverables:** candidate output and warnings.

**Acceptance criteria:** in-scope metadata is accurate; collisions and ambiguity are visible; generated content is not runtime-accessible.

**Security/testing:** output contains no credentials or row data and is treated as potentially sensitive schema information.

**Dependencies:** Phase 6.

### Phase 8 — Manually approve entities and fields

**Tasks:** decide expose/reject for each table and field; confirm names, descriptions, relationships, roles, and protocols; obtain required approval; manually place only approved definitions in `dab-config.json`.

**Deliverables:** approved entity/field list and reviewed DAB definitions.

**Acceptance criteria:** nothing is implicitly exposed; every entity has a purpose and read-only permission; excluded items remain absent.

**Security/testing:** consider inference through filters, aggregates, and relationships.

**Dependencies:** Phases 3 and 7.

### Phase 9 — Keep configuration readable

**Tasks:** start with one `dab-config.json`; use consistent ordering; split into a few native child files only if needed; keep related entities together because relationships cannot cross files.

**Deliverables:** readable approved config and, only when necessary, a documented native multi-file layout.

**Acceptance criteria:** `dab validate` passes; names are unique; relationships resolve; no custom merge framework exists without demonstrated need.

**Security/testing:** compare config to the approved list so candidate/excluded entities cannot slip in.

**Dependencies:** Phases 5 and 8.

### Phase 10 — Enforce DAB read-only permissions

**Tasks:** configure explicit `read` actions per role, field includes/excludes, confirmed row policies if needed, and per-entity protocol enablement.

**Deliverables:** DAB permissions and concise permission matrix.

**Acceptance criteria:** roles see only approved entities/fields/rows; no DAB write/execute action exists; SQL negative tests still pass.

**Security/testing:** test invalid identity, role selection, excluded-field selection/filtering, nested access, and row boundaries.

**Dependencies:** Phases 2, 8, and 9.

### Phase 11 — Validate MCP

**Tasks:** test description, read, filter, pagination, aggregation, grouping, timeouts, permissions, and field exclusions; document that MCP does not provide arbitrary SQL or automatic joins.

**Deliverables:** small smoke-test script or documented commands/results.

**Acceptance criteria:** only approved read tools/entities/fields appear; write/execute tools are absent; limits and permissions work.

**Security/testing:** include malformed, oversized, injection-like, unauthorized, and unavailable-database cases.

**Dependencies:** Phases 5, 9, and 10.

### Phase 12 — Validate GraphQL relationships

**Tasks:** test applicable relationship types, nested reads, filters, sorting, pagination, null relationships, and invalid definitions; measure representative queries and add indexes/views only when evidence supports them.

**Deliverables:** documented GraphQL queries/results and confirmed limitations.

**Acceptance criteria:** approved relationships return correct synthetic data; nested permissions work; invalid relationships fail validation; results are bounded.

**Security/testing:** test unauthorized traversal, excluded fields, and expensive query shapes.

**Dependencies:** Phases 9 and 10. May run alongside Phase 11.

### Phase 13 — Perform the security check

**Tasks:** run direct SQL write/DDL failures; test writes through every enabled protocol; try rejected data through reads, filters, ordering, relationships, and aggregates; scan Git, image, logs, and traces for secrets; confirm request limits.

**Deliverables:** concise pass/fail security checklist and remediation items.

**Acceptance criteria:** writes fail closed; rejected data is inaccessible; no high-severity finding or secret leak remains.

**Security/testing:** use the real DAB role/configuration rather than mocks.

**Dependencies:** Phases 10–12.

### Phase 14 — Validate health and observability

**Tasks:** verify structured logs, `/health`, durations, and OpenTelemetry where enabled; test slow/down SQL, invalid config, authentication failures, and timeouts; write troubleshooting notes.

**Deliverables:** working health/telemetry settings and troubleshooting guidance.

**Acceptance criteria:** failures are distinguishable without logging records/secrets; telemetry failure does not stop DAB.

**Security/testing:** restrict detailed health outside local POC and verify redaction with sentinel values.

**Dependencies:** Phases 4, 5, 11, and 12.

### Phase 15 — Add proportionate CI validation

**Tasks:** begin with DAB validation, secret scanning, and Docker build; add SQL verification and protocol smoke tests when an ephemeral SQL Server is practical; do not build deployment infrastructure before selecting a target environment.

**Deliverables:** small CI workflow containing currently valuable checks.

**Acceptance criteria:** selected checks run repeatably with pinned versions and no production credentials/data; invalid config or failed security checks block the build.

**Security/testing:** use short-lived/minimal CI credentials and scan images/config changes.

**Dependencies:** checks are added incrementally; the complete POC gate follows Phases 4–14.

### Phase 16 — Complete DAB and hand over

**Tasks:** run the POC checklist cleanly; document MCP/GraphQL contracts, authentication, limits, errors, and limitations; give setup/troubleshooting instructions to the backend team.

**Deliverables:** tested DAB release candidate, examples, readiness checklist, and known issues.

**Acceptance criteria:** DAB works without Spring Boot; approved reads pass; writes and unapproved access fail; health and validation pass; backend needs no SQL credential or arbitrary-SQL path.

**Security/testing:** rerun secret scan, least-privilege checks, protocol authorization, and representative load checks.

**Dependencies:** all earlier phases; Phases 13–15 are final gates.

## 6. Dependencies at a glance

```text
Phase 0 -> Phase 1 -> Phase 2
              |        |
              v        v
           Phase 3 -> Phase 6 -> Phase 7 -> Phase 8
              |                              |
              `-> Phase 4 -> Phase 5 --------+-> Phase 9 -> Phase 10
                                                              |       |
                                                              v       v
                                                        Phase 11   Phase 12
                                                              \       /
                                                        Phase 13 and 14
                                                              |
                                                          Phase 15
                                                              |
                                                          Phase 16
```

## 7. Minimal testing approach

The POC does not need a large testing framework or many test folders.

- `database/verify.sql`: schema integrity, allowed reads, and forbidden SQL writes/DDL.
- One small smoke-test script or documented commands: DAB validation, health, MCP reads/aggregates, and GraphQL relationships.
- CI: secret scan, DAB validation, Docker build, then smoke tests when feasible.
- Synthetic sentinel fields/rows make accidental exposure easy to detect.

Add a dedicated test project/folder only when tests become numerous enough that this approach is difficult to maintain.

## 8. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Screenshots omit schema detail | Confirm unknowns; do not infer silently |
| DAB SQL account is too broad | Narrow `SELECT` grants and automated failure checks |
| Candidate output is treated as approved | External generator, temporary output, manual review/copy only |
| Configuration becomes too large | Improve ordering first; then use a few native child files |
| DAB behavior changes | Pin/test an exact version and upgrade intentionally |
| Local anonymous settings reach shared environments | Separate settings and CI checks; require enterprise auth |
| AI requests too much data | Pagination, result caps, timeouts, and representative tests |
| Sensitive data leaks indirectly | Field/row restrictions and negative protocol tests |
| POC becomes overengineered | Do not create unused folders, frameworks, services, or pipelines |

## 9. Acceptable POC shortcuts

- Local SQL Server and synthetic data.
- Local SQL authentication with secrets outside Git.
- DAB unauthenticated/simulator mode only in a local, non-shared environment.
- One DAB config file and four SQL files.
- Manual approval notes instead of a governance platform.
- Documented smoke-test commands instead of a full test framework.
- Local telemetry or structured logs only at first.
- REST disabled unless a concrete need appears.

Read-only security, explicit exposure review, secret handling, and negative tests are not optional shortcuts.

## 10. Before production

- Add enterprise authentication, role/claim mapping, and approved secret management/workload identity.
- Add TLS, private networking, ingress/rate/payload controls, and platform hardening.
- Complete formal data/privacy review and access recertification.
- Define SQL backup/restore, HA/DR, patching, capacity, and performance plans.
- Define SLOs, alerts, retention/redaction, and incident runbooks.
- Add load, soak, recovery, and security tests.
- Add image/SBOM signing, vulnerability policy, and upgrade/rollback process.
- Define backward-compatible schema/entity changes and row-level security where required.

## 11. Deferred decisions

| Decision | Resolve when |
|---|---|
| Exact DAB 2.x version | Phase 4 |
| Generator language | Phase 6, based on team familiarity |
| Final entity/domain names | After actual schema review |
| Splitting `dab-config.json` | Only when one file is demonstrably hard to maintain |
| REST enablement | When a concrete consumer requires it |
| Enterprise identity provider | Before shared DEV deployment |
| DAB row policy vs SQL Server RLS | When row requirements are known |
| Views/read-only procedures | Only after a proven query cannot be modeled acceptably |
| Hosting platform/full CI/CD | Before production planning |
| Exact limits and SLOs | After measuring real POC queries |

## 12. Open questions

1. What are the authoritative table definitions, keys, and relationships?
2. Which SQL Server version must the POC and company DEV support?
3. Who approves tables and fields for exposure?
4. Which identity provider and roles will shared DEV use?
5. Which representative MCP/GraphQL questions define POC success?

## 13. Official DAB references

Verify syntax against the selected DAB version:

- [Data API Builder documentation](https://learn.microsoft.com/en-us/azure/data-api-builder/)
- [DAB configuration validation](https://learn.microsoft.com/en-us/azure/data-api-builder/command-line/dab-validate)
- [Multiple configuration files](https://learn.microsoft.com/en-us/azure/data-api-builder/concept/config/multi-config)
- [Authorization](https://learn.microsoft.com/en-us/azure/data-api-builder/concept/security/authentication-local)
- [SQL MCP Server](https://learn.microsoft.com/en-us/azure/data-api-builder/mcp/overview)
- [MCP DML tools](https://learn.microsoft.com/en-us/azure/data-api-builder/mcp/data-manipulation-language-tools)
- [Health checks](https://learn.microsoft.com/en-us/azure/data-api-builder/concept/monitor/health-checks)
- [OpenTelemetry](https://learn.microsoft.com/en-us/azure/data-api-builder/concept/monitor/open-telemetry)

## 14. Recommended start

Start with Phase 0 and stop until schema questions are answered. Then implement Phases 1–3 with the four simple SQL files. Do not build the generator, DAB entities, extra folders, or CI infrastructure until the preceding phase actually requires them.

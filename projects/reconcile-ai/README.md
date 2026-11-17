> **Copy of a private repository document**, snapshot taken 2026-11-17. The source code is private, so links to files in the repository are shown as plain paths. Status labels are the author's own, as written at the time of the snapshot. All data is synthetic.

# reconcile-ai — Reconciliation Break Resolution Copilot

![Java](https://img.shields.io/badge/Java-21-orange.svg)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-3.5.16-brightgreen.svg)
![Spring AI](https://img.shields.io/badge/Spring_AI-1.1.8-blue.svg)
![Spring MVC](https://img.shields.io/badge/Spring%20MVC-Spring%20Boot-6DB33F?logo=spring&logoColor=white)
![Spring Security OAuth2](https://img.shields.io/badge/Spring%20Security-OAuth2-6DB33F?logo=springsecurity&logoColor=white)
![Spring Data JPA](https://img.shields.io/badge/Spring%20Data-JPA-6DB33F?logo=spring&logoColor=white)
![PostgreSQL 16](https://img.shields.io/badge/PostgreSQL-16-4169E1?logo=postgresql&logoColor=white)
![Apache Tika](https://img.shields.io/badge/Apache-Tika-D22128?logo=apache&logoColor=white)
![Flyway](https://img.shields.io/badge/Flyway-Migrations-CC0200?logo=flyway&logoColor=white)
![Maven](https://img.shields.io/badge/Maven-Build-C71A36?logo=apachemaven&logoColor=white)
![JUnit 5](https://img.shields.io/badge/JUnit-5-25A162?logo=junit5&logoColor=white)
![Testcontainers](https://img.shields.io/badge/Testcontainers-Integration%20Tests-2496ED?logo=testcontainers&logoColor=white)

> **Notice: independent demonstration project.**
> This is a generic, fictitious application built to demonstrate software engineering and banking-domain
> functional knowledge. It does not contain, reflect or derive from any client's data, code, functionality,
> architecture, designs or confidential information, and it is not affiliated with or endorsed by any current
> or former employer or client. All data (accounts, amounts, references, documents) is synthetic. Any
> resemblance to real people,organizations or transactions is coincidental. The development keys in this
> repository are throwaway, for local use only. The software is provided "as is" under the Apache License 2.0
> and is not intended for production use without independent security review and hardening.

A copilot for banking reconciliation operations. When a ledger amount and an external
(counterparty or bank) amount disagree, a **break** is raised. Analysts attach **evidence** such as
statements or SWIFT confirmations. An investigation retrieves that evidence, produces a resolution
proposal with citations, and a policy gate routes it to human approvers. **No break is resolved
without a human checker**, and the model never writes to the ledger.

---

## 1. Implementation approach

**Incremental, test-gated tiers.** Each tier has a done-when test, and the next tier does not start
until that test is green.

| Stage | Scope | Done-when |
|---|---|---|
| Foundation (Tier 0–1) | Break CRUD, evidence upload with hash dedup, JWT auth, RBAC, tenant isolation | `SecurityAndTenantIsolationIT` green (release-blocking) |
| AI investigation (Tier 2) | Evidence ingestion, hybrid retrieval, cited proposals, policy gate, maker-checker, hash-chained audit, SSE progress | `InvestigationFlowIT`, `GoldenPathIT`, `AuditChainIT` green |
| Production hardening | Retry + circuit breaker, PII masking (identifiers + Presidio names), metrics and traces, Angular UI, Redis semantic cache, Kafka outbox, Postgres RLS, cross-instance SSE, daily digest, Keycloak OIDC, audit anchoring, Kubernetes + HPA | Each feature has its own IT on real infrastructure (see section 6); manifests validated on kind |
| Gap closure (4F) | Original evidence file kept and downloadable, per-tenant rate limit on investigations, optional Swagger UI, SBOM and image scan in CI | `EvidenceContentIT`, `RateLimitIT`, `SwaggerEnabledIT`, `TokenBucketRateLimiterTest` |
| Evaluation harness (4A) | 28 labelled synthetic breaks run through the real pipeline; accuracy, citation and retrieval numbers per leg; mock-llm regression gate in the build; real-model run on request | `EvalCaseLoaderTest`, `InvestigationEvalIT`, `RetrievalEvalIT` |
| Agents and cloud (rest of Phase 4) | LangChain4j investigation agent with read-only audited tools, Alertmanager e-mail alerts, Amazon Bedrock adapter, AWS deployment with Terraform | 4B, 4C, 4D and 4E built (4D and 4E not yet run against a real AWS account) — see section 9 |
| Supervisor agents | Specialist agents with a reviewer, compared with the existing modes on the evaluation set | Built and measured on gpt-4o-mini (4G): the decision rule fixed in advance was **not met**; single call stays the default, supervised modes stay off |
| Platform integration | Shared token issuer, internal knowledge search, recommendation intake | Planned — see BRD platform requirements |

**Principles applied in the code** (each is commented at the point of use):

1. **Security by construction.** Repositories extend Spring Data's empty `Repository` interface, so tenant-blind
   methods like `findById` do not exist. No API accepts a tenant id.
2. **Fail closed.** Unknown JWT roles are dropped. A token without `tenant_id` is rejected with 401.
3. **No existence oracles.** Another tenant's resource returns 404, the same response as a missing resource.
4. **Deterministic gate.** The model proposes; Java code decides routing. Confidence, citation validity and
   variance thresholds are enforced in `PolicyGateService`, never by the model.
5. **Single resolution path.** Only `ResolutionCommitService` can move a break to RESOLVED, and only
   `ApprovalService` calls it. An architecture test enforces this.
6. **Money is exact.** `BigDecimal` via the `Money` value object. Requests and Kafka messages carry amounts as JSON
   strings; responses serialize `BigDecimal` as JSON numbers with the exact digits (a JavaScript client that parses
   them as numbers loses precision beyond about 15 significant digits). `double` is never used.
7. **Flyway owns the schema.** Hibernate only validates it (`ddl-auto: validate`).
8. **Real infrastructure in tests.** Postgres runs in Testcontainers (pgvector image) and is never mocked.

---

### Status at a glance

Updated 2026-11-10. Three groups, so nothing is presented as more finished than it is.

**Built and verified locally** (unit, integration on real containers, UI specs; CI green)
Break and evidence CRUD, hybrid retrieval, cited proposals, policy gate, maker-checker, hash-chained audit with signed
anchors, resilience, PII masking (identifiers and Presidio), Redis cache and SSE fan-out, Kafka outbox, Postgres RLS,
Keycloak OIDC, daily digest, Kubernetes manifests and HPA (validated on kind), evaluation harness, single-tool-using
agent mode (off by default), Prometheus alert rules, rate limiting and stored evidence bytes.

**Built, waiting for a test with real credentials or accounts**

| Item | What was verified | What is still needed |
|---|---|---|
| E-mail alerts (Gmail or any real SMTP) | Rules pass `promtool` tests; routing proven against a local Mailpit sink; outage drill recorded | Real SMTP credentials in `alertmanager.yml` (kept out of git) and one live delivery |
| Amazon Bedrock live run (`BedrockEvalIT`, `BedrockChatSmokeIT`, `BEDROCK_LIVE=true`) | Adapter proven against a WireMock Converse API; `BedrockFakeEvalIT` scores match mock-llm | AWS credentials and Bedrock model access; fills the last row of the provider comparison |
| AWS deployment (`envs/demo`: ECS Fargate, RDS, ALB, Secrets Manager, S3 Object Lock, Budgets) | `terraform validate` and `test` with mock providers, tflint, checkov; `envs/localstack` applied and destroyed in CI | A real account, AWS credit, and your explicit go-ahead for one apply, record, destroy session (about 0.20 USD per hour) |
| Shared stream fan-out between two Fargate tasks | Redis fan-out tested locally | Redis in the AWS stack (not built there) |
| KEDA autoscaling (`k8s/overlays/keda`) | Overlay rendered and schema-checked with kubeconform in CI | A cluster with KEDA installed and Prometheus scraping the app |
| Per-tenant gateway credentials | Tested against a fake gateway and a fake realm | A real two-tenant run needs a second Keycloak client |

**Pending implementation** (designed, not started)

| Item | Status |
|---|---|
| Expose the read-only tools over MCP for other services | Not started; needs a token-based tenant design |
| Platform integration (shared token issuer, knowledge search, recommendation intake) | Planned |
| Trivy scan that fails the build | Listed under Known limitations |

---

## 2. Features

| Area | Feature | Where |
|---|---|---|
| Breaks | Create, get, list (tenant-scoped, newest first, size capped) | `BreakController`, `BreakService` |
| Evidence | Multipart upload with SHA-256 dedup (201 new / 200 duplicate), original file kept (`evidence_content`, RLS) and downloadable as an attachment, list, hybrid search | `EvidenceController`, `EvidenceSearchController` |
| Ingestion | Tika text extraction, chunking, embeddings into pgvector | `infrastructure/ai/ingestion/EvidenceIngestor` |
| Retrieval | Dense (pgvector) + sparse (Postgres `tsvector`) fused with Reciprocal Rank Fusion | `infrastructure/ai/retrieval/HybridRetriever`, `RrfFusion` |
| Investigation | Async job with heartbeat lease, recovery of orphaned jobs, SSE progress (cross-instance via Redis pub/sub when `SSE_FANOUT=redis`) | `InvestigationController`, `InvestigationJobRunner`, `InvestigationRecovery` |
| Proposals | Structured output with citations; citations verified against retrieved chunk ids | `ProposalService`, `CitationVerifier` |
| Policy gate | Routes to MANUAL_REVIEW, L1 or L2 (auto-resolve implemented but off by default) | `PolicyGateService` |
| Approvals | L1/L2 queues, maker-checker, stricter-of-stored-and-recomputed tier | `ApprovalController`, `ApprovalService` |
| Audit | Hash-chained, append-only `audit_event`; verify endpoint for SRE_ADMIN; optional Ed25519-signed chain heads anchored in S3 Object Lock (MinIO locally) with an anchor verify endpoint | `HashChainAuditRecorder`, `AuditController`, `application/audit/anchor/` |
| Safety | Injection scanner before any model call; advisor chain (input guard, token usage) | `InputGuardAdvisor`, `TokenUsageAdvisor`, `InjectionScanner` |
| Security | RS256 JWT resource server, mandatory `tenant_id`, closed role set; real IdP mode (Keycloak realm included; JWKS, issuer and audience checks) or demo keys (signature, expiry and `tenant_id` only; no issuer or audience check) | `SecurityConfig`, `TenantContext`, `Role`, `docker/keycloak/` |
| Tenant isolation (DB) | Postgres Row-Level Security on every tenant table and `vector_store`; app runs as a non-owner role; connections bound to the caller's tenant on checkout; fail closed | `V6__row_level_security.sql`, `TenantAwareDataSource`, `TenantScope` |
| Resilience | LLM retry (transient only, exponential backoff) inside a circuit breaker; fail fast when open | `infrastructure/ai/resilience/LlmResilience` |
| Privacy | PII masking of accounts, IBANs, cards (Luhn), emails and phones before every model call; optional Presidio entity recognition masks person names and locations (fail-open or fail-closed) | `PiiMaskingAdvisor`, `PiiMasker`, `PresidioNerDetector` |
| Observability | Prometheus metrics (tokens, estimated cost, LLM latency, breaker, retries, PII counts); OpenTelemetry traces via OTLP; Grafana dashboard | `/actuator/prometheus`, `docker/observability/` |
| Messaging | Transactional outbox (`outbox_event`, same tx as the state change) relayed to Kafka; event-driven break intake with idempotency and a dead-letter topic; off by default | `infrastructure/messaging/`, `BreakIngestionService` |
| Cost | Redis semantic cache of verified model drafts (exact + semantic, per tenant, evidence-snapshot bound, fail-open); off by default | `infrastructure/ai/cache/` |
| UI | Angular 20 console: breaks, evidence + hybrid search, live SSE investigation progress, cited proposals, approvals, audit verify | `ui/` |
| Reporting | Daily digest per tenant (activity from the audit chain, backlog, AI calls, tokens, estimated cost, root causes); SQL-built, advisory-locked so one instance runs it; `DAILY_DIGEST_READY` outbox event | `application/digest/`, `DigestController` |
| Cost control | Per-tenant token bucket on `POST /investigations` (default burst 20, 20 per minute); 429 + `Retry-After` before any DB or model work. Per JVM by default; `RATE_LIMIT_STORE=redis` shares one bucket across replicas | `TokenBucketRateLimiter`, `RedisTokenBucketRateLimiter`, `RateLimitProperties` |
| API docs | Optional springdoc Swagger UI with a bearer scheme; paths are not opened unless enabled | `OpenApiConfig`, `SecurityConfig` |
| Supply chain | CycloneDX SBOM artifact and Trivy image scan (report-only) in CI | `.github/workflows/ci.yml` |
| Ops | Actuator health and Prometheus only; no error-message leakage | `/actuator/health`, `/actuator/prometheus` |
| Deployment | Multi-stage Dockerfile (non-root JRE); Kubernetes manifests with Kustomize: probes, CPU-based HPA (2 to 6), PDB, hardened pod security; kind overlay with Postgres and Redis; KEDA overlay (CPU plus queued investigations) | `Dockerfile`, `k8s/` |

Endpoints are under `/api/v1`. Swagger UI is optional and off by default: set `SWAGGER_ENABLED=true` and open `/swagger-ui.html` (use the Authorize button with a JWT).

---

## 3. Tech stack

| Layer | Technology |
|---|---|
| Language | Java 21 (records, pattern matching, text blocks; virtual threads for request handling, see `docs/adr/ADR-virtual-threads.md`) |
| Framework | Spring Boot 3.5.16, Spring MVC + `SseEmitter` (no WebFlux) |
| Security | Spring Security OAuth2 Resource Server, Nimbus JOSE (RS256) |
| Persistence | Spring Data JPA / Hibernate, PostgreSQL 16 with pgvector |
| Migrations | Flyway |
| AI | Spring AI 1.1.8 (OpenAI embeddings; mock-llm default chat model; Ollama profile) |
| Parsing | Apache Tika |
| Resilience | Resilience4j 2.4 (retry, circuit breaker), programmatic, no AOP |
| Ops | Spring Boot Actuator, Micrometer + Prometheus, Micrometer Tracing + OpenTelemetry (OTLP), Jaeger, Grafana |
| Frontend | Angular 20 (standalone components, signals, RxJS), Karma/Jasmine |
| Testing | JUnit 5, Testcontainers, AssertJ, Awaitility, spring-security-test |
| Build | Maven (surefire for `*Test`, failsafe for `*IT`) |
| Deploy | Docker (multi-stage, Temurin 21 JRE, non-root), Kubernetes 1.31 + Kustomize, HPA (autoscaling/v2), KEDA ScaledObject (overlay), kind for local validation, kubeconform in CI (the KEDA overlay uses the datreeio CRDs-catalog schema) |
| Integrations | Keycloak 26 (OIDC), Microsoft Presidio analyzer, AWS SDK v2 S3 (MinIO locally) |
| Phase 4 | LangChain4j 1.x (core libraries, no starter), Spring AI Bedrock Converse, Alertmanager + Mailpit, WireMock (test), Terraform 1.13 (AWS) with tflint and checkov |

---

## 4. Architecture

```
api/            REST controllers, DTOs, GlobalExceptionHandler       (HTTP in/out only)
application/    BreakService, EvidenceService, InvestigationService,
                ProposalService, PolicyGateService, ApprovalService,
                ResolutionCommitService, audit recorder              (use cases, transactions)
domain/         BreakItem, Evidence, InvestigationJob, ResolutionProposal,
                ApprovalTask, Money, enums                           (no Spring web/AI imports)
infrastructure/
  persistence/  tenant-scoped repositories
  security/     SecurityConfig, TenantContext, CurrentUser, Role
  ai/           ChatClient config, advisors, retrieval, ingestion, prompt builder, mock LLM
  streaming/    SSE broadcaster and progress events
```

Request evaluation order for protected endpoints:

```
Bearer token ─► JwtDecoder: signature (RS256) + exp/nbf + tenant_id present ──✗──► 401
            ─► roles claim ─► Role enum (unknown dropped) ─► @PreAuthorize ──✗──► 403
            ─► Service: tenantId = TenantContext (JWT) ─► tenant-scoped query ──∅──► 404
            ─► 200 / 201 / 202
```

Investigation flow:

```
evidence (READY) ─► POST /investigations ─► job QUEUED
  ─► RETRIEVING (hybrid, tenant-filtered) ─► CALLING_MODEL (input guard, token budget)
  ─► VERIFYING (citations ⊆ retrieved ids) ─► ROUTING (policy gate)
      ├─ MANUAL_REVIEW
      ├─ L1 / L2 ─► approval task ─► maker-checker ─► ResolutionCommitService ─► RESOLVED
      └─ AUTO_RESOLVE (config off by default)
```

---

## 5. Run locally (Windows 11)

Requires JDK 21, Maven 3.9+ and Docker Desktop.

```powershell
# 1. Database (Postgres + pgvector)
docker compose -f docker/docker-compose.yml up -d

# 2. Full test gate: unit (surefire) + integration (failsafe, needs Docker)
mvn clean verify

# 3. Run the app (default profile: mock-llm, no external model calls)
mvn spring-boot:run

# 4. Mint a local token and call the API
mvn -q test-compile
$t = .\scripts\generate-local-jwt.ps1 -Subject analyst-1 -Tenant TENANT_A -Role RECON_ANALYST
curl.exe -s -X POST http://localhost:8080/api/v1/breaks `
  -H "Authorization: Bearer $t" -H "Content-Type: application/json" `
  -d '{\"breakReference\":\"BRK-1\",\"ledgerAmount\":\"50500.00\",\"externalAmount\":\"50000.00\",\"currency\":\"USD\"}'
```

The OpenAI key is read from `OPENAI_API_KEY` at startup outside the test profile. The automated suite does not
need it, because it uses a fake embedding model and the mock chat model.

### Run with a real model (OpenAI)

The default profile uses a deterministic mock model for chat, so the tests need no key. Embeddings always call
OpenAI `text-embedding-3-small` (a locked decision), so a live run of the app or the demo needs `OPENAI_API_KEY`,
which costs a fraction of a cent per run. To also let a real chat model draft the proposals, use the `openai` profile:

```powershell
$env:OPENAI_API_KEY = "<your key>"
mvn spring-boot:run "-Dspring-boot.run.profiles=openai"
```

Optional: `$env:OPENAI_CHAT_MODEL = "gpt-4o-mini"` (the default). The real-model smoke test is opt-in:

```powershell
mvn verify "-Dfailsafe.excludedGroups=ollama" "-Dit.test=OpenAiChatSmokeIT"
```

### Run with Amazon Bedrock

The `bedrock` profile uses the Bedrock Converse API for chat (embeddings stay on OpenAI). Credentials come from the default
AWS chain (SSO profile, environment variables or an ECS task role); no key is ever stored in the repo, and the model must be
enabled in the Bedrock console first.

```powershell
$env:AWS_REGION = "us-east-1"; $env:BEDROCK_CHAT_MODEL = "amazon.nova-lite-v1:0"
mvn spring-boot:run "-Dspring-boot.run.profiles=bedrock"
```

The default test run uses `bedrock-fake` instead: the same code against a local WireMock server, so it needs no AWS account.
`BedrockAdapterIT` proves a structured answer is parsed (also when wrapped in a markdown fence), tokens and cost are
recorded under the model name, the 700-token output cap reaches Bedrock, throttling (429) is retried once by our policy
(the AWS SDK's own retries are off), a bad request and non-JSON text are not retried, and a sustained outage opens the
circuit breaker so later jobs never call Bedrock. The live tests are opt-in and cost a fraction of a cent:

```powershell
$env:BEDROCK_LIVE = "true"
mvn verify "-Dfailsafe.excludedGroups=openai,ollama" "-Dit.test=BedrockChatSmokeIT"
mvn verify "-Dfailsafe.excludedGroups=openai,ollama" "-Dit.test=BedrockEvalIT"    # same 28 cases as the OpenAI run
```

Prices for the cost metric are per profile (`application-bedrock.yml`, Nova Lite list price); override with
`LLM_PROMPT_USD_PER_MILLION` and `LLM_COMPLETION_USD_PER_MILLION`. Agent mode runs on OpenAI only for now.

### Deploy to AWS (Terraform)

`infra/` holds Terraform modules (network, database, storage, secrets, app, ci, budget) and a `demo` environment: ECS Fargate
behind a load balancer that accepts only your IP, RDS Postgres 16 with pgvector, an S3 Object Lock bucket for audit anchors,
Secrets Manager, a GitHub OIDC deploy role that can only roll out an image, and a budget alarm. It is built for
deploy, record, destroy (about 0.20 USD per hour). It is validated by `terraform fmt`, `validate`, 38 mock-provider tests,
tflint and checkov in CI, with no AWS account. **It has not been applied to a real account yet.** Runbook, costs and limits:
`infra/README.md` (`infra/README.md`).

### Production hardening: try it

**Row-Level Security (second tenant guard).** Flyway runs as the schema owner (`DB_USER`/`DB_PASSWORD`, default
`reconcile`); the app connects as `reconcile_app` (`DB_APP_USER`/`DB_APP_PASSWORD`), which V6 creates with DML grants
only. Each connection checkout runs `set_config('app.tenant_id', <tenant from the JWT or an explicit job scope>)`;
policies then hide other tenants' rows even from a query with no tenant filter, and reject writes for another tenant.
No tenant bound means no rows. Cross-tenant platform work (outbox relay, recovery sweeper) uses an explicit, named
system scope that no JWT can claim. Test: `RowLevelSecurityIT`.

**Live progress across instances.** Run every instance with `SSE_FANOUT=redis` (and Redis from the `cache`
profile). Each progress event is published on the Redis channel `reconcile:progress`; every node delivers it to its
own open streams, so a browser may be connected to any node behind the load balancer. If Redis is unavailable the
node falls back to local delivery; the snapshot-on-subscribe and polling API still make the client correct.
Metric: `reconcile_sse_fanout_total{result}`. Test: `SseFanoutIT` (two real app instances, job on one, stream on the other).

**Daily digest.** At 00:15 UTC (`reconcile.digest.cron`) one instance (Postgres advisory lock) builds yesterday's
digest for every tenant: activity counted from the audit chain, open backlog and oldest age, investigations and failure
reasons, model calls, cache hits, tokens and estimated cost, root causes and routes. Numbers are SQL aggregates and the
text is a fixed template, so the digest never mis-states a number. Re-runs overwrite (idempotent). A
`DAILY_DIGEST_READY` event goes to the outbox for downstream e-mail or chat delivery. API: `GET /api/v1/digests?days=7`,
`GET /api/v1/digests/{date}` (approvers, SRE), `POST /api/v1/digests/{date}/generate` (SRE backfill). Disable with
`DIGEST_ENABLED=false`. Test: `DailyDigestIT`.

**Real identity provider (Keycloak).** `docker compose -f docker/docker-compose.yml --profile keycloak up -d`, then
run the API with `OIDC_ISSUER_URI=http://localhost:8180/realms/reconcile`. The API then validates tokens against the
realm's JWKS (key rotation without redeploy), requires `iss` = that issuer, `aud` ∋ `reconcile-api` and a `tenant_id`
claim; demo-key tokens are refused. The realm (`docker/keycloak/reconcile-realm.json`) maps a admin-editable
`tenant_id` user attribute and realm roles into the access token, so nothing downstream changes. The UI signs in with
Authorization Code + PKCE (see `ui/README.md`). Entra ID works the same way: set its v2.0 issuer and map `tenant_id`
and `roles` as optional/app-role claims. Test: `KeycloakIT` (real Keycloak container).

**Audit anchoring (S3 Object Lock).** The hash chain catches an edited row, but someone with database-owner rights
could disable the trigger and rewrite the chain consistently. Every 15 minutes (`reconcile.audit.anchor.cron`, one
instance via advisory lock) each tenant whose chain moved gets an anchor: `(tenant, sequence, headHash, anchoredAt)`,
signed with Ed25519, written to a bucket with Object Lock (COMPLIANCE mode by default: nobody can delete or overwrite
it before `retain-until`), and recorded in the append-only `audit_anchor` table. `GET /api/v1/audit/anchors/verify`
(SRE_ADMIN) re-reads each stored object, checks the signature with the public key, and checks the live chain still has
that hash at that sequence. The S3 client defaults to the light URL-connection HTTP client; if you see transient `Unexpected end of file from server` errors against your S3 endpoint, set `reconcile.audit.anchor.s3.http-client=apache` (pooled client that validates idle connections). Try it locally:

```powershell
docker compose -f docker/docker-compose.yml --profile anchor up -d      # MinIO on :9000 (throwaway credentials)
# one-off key pair (openssl ships with Git for Windows); keep the private key in a secret manager outside a laptop
openssl genpkey -algorithm ed25519 -out anchor.pem
openssl pkcs8 -topk8 -nocrypt -in anchor.pem -outform DER -out anchor-priv.der
openssl pkey -in anchor.pem -pubout -outform DER -out anchor-pub.der
$env:AUDIT_ANCHOR_PRIVATE_KEY = [Convert]::ToBase64String([IO.File]::ReadAllBytes("$PWDnchor-priv.der"))
$env:AUDIT_ANCHOR_PUBLIC_KEY  = [Convert]::ToBase64String([IO.File]::ReadAllBytes("$PWDnchor-pub.der"))
$env:AUDIT_ANCHOR_ENABLED = "true"; $env:AUDIT_ANCHOR_S3_ENDPOINT = "http://localhost:9000"
$env:AUDIT_ANCHOR_S3_ACCESS_KEY = "minio-local"; $env:AUDIT_ANCHOR_S3_SECRET_KEY = "minio-local-only"
$env:AUDIT_ANCHOR_S3_PATH_STYLE = "true"; $env:AUDIT_ANCHOR_S3_CREATE_BUCKET = "true"
mvn spring-boot:run
```

On AWS leave the endpoint and keys blank (default credential chain, for example IRSA on EKS) and point
`AUDIT_ANCHOR_S3_BUCKET` at a bucket created with Object Lock. Test: `AuditAnchorIT` (real MinIO: anchor written,
idempotent re-run, locked object cannot be deleted, a consistent chain rewrite passes the chain check but fails the
anchor check, roles and tenant isolation, forged or re-used signatures rejected).

**Kubernetes and autoscaling.** `k8s/` holds a production-shaped `base/` (Deployment with startup, liveness and
readiness probes, HPA on CPU from 2 to 6 replicas, PodDisruptionBudget, non-root read-only pod) and an
`overlays/kind/` that adds in-cluster Postgres and Redis. Several replicas are safe because SSE fan-out goes through
Redis and the scheduled jobs take Postgres advisory locks. Validated on a local kind cluster: clean rollout,
authenticated API call through the Service, and a scale-up from 2 to 4 pods under load. Steps: `k8s/README.md` (`k8s/README.md`).
`overlays/keda/` removes the CPU HPA and adds a KEDA ScaledObject (2 to 6 replicas) with a Prometheus trigger on active plus queued investigations (threshold 3 per pod) and a CPU 70% trigger, and sets `RATE_LIMIT_STORE=redis` so replicas share one rate-limit bucket. That overlay is validated only: rendered and schema-checked, not run on a cluster with KEDA.
CI renders the base and both overlays, validates them with kubeconform and builds the image.

**LLM retry and circuit breaker.** With the default mock model, evidence containing the marker text
`simulate transient glitch` fails the first model call and succeeds on retry; `simulate provider outage` fails
every attempt, so the job ends `FAILED` with reason `LLM_UNAVAILABLE` and the break becomes `FAILED_RETRYABLE`.
Five such failures in the window open the breaker: further investigations fail at once with `LLM_CIRCUIT_OPEN`,
without calling the model, until it half-opens after 30 s. Settings: `reconcile.ai.resilience.*`.
Fixtures: `src/test/resources/fixtures/evidence/timing-*.txt`. Tests: `LlmResilienceTest`, `LlmResilienceIT`.

**PII masking.** Every prompt passes `PiiMaskingAdvisor` (after the injection guard, before token accounting).
`reconcile_pii_masked_total{type=...}` counts what was masked; the values are never logged. Disable with
`reconcile.ai.pii-masking.enabled=false`. Tests: `PiiMaskerTest`.

**Names and locations (Presidio).** Regexes cannot recognise a person or a place, so an optional second pass sends the
already-masked text to a Microsoft Presidio analyzer sidecar (spaCy model) inside the same network, never to the model
provider. `docker compose -f docker/docker-compose.yml --profile presidio up -d`, then run with `PII_NER_ENABLED=true`
(`PII_NER_URL` defaults to `http://localhost:5002`). Detected `PERSON` and `LOCATION` spans become `[PERSON]` /
`[LOCATION]`; business references and chunk ids are left alone. If the sidecar is down: by default the call proceeds
with identifier masking only and `reconcile_pii_ner_unavailable_total` counts it (fail-open); with
`PII_NER_FAIL_CLOSED=true` the model call is refused instead, so unmasked names never leave the process.
Tests: `NerMaskingTest`, `PresidioNerIT` (real Presidio container).

**Semantic cache (Redis).** `docker compose -f docker/docker-compose.yml --profile cache up -d`, then run with
`CACHE_ENABLED=true`. Only citation-verified drafts are stored. A hit replaces the model call only; normalisation,
citation check and the policy gate still run. Hits need the same tenant AND the identical retrieved-chunk set, so new
evidence always forces a fresh call; a suspect analyst note bypasses the cache. Redis errors are misses, never
failures. Metrics: `reconcile_llm_cache_total{result=hit_exact|hit_semantic|miss|store|error}`; hits are recorded in
`llm_call` as `CACHE_HIT` with 0 tokens. Tests: `SemanticCacheIT` (real Redis), `SemanticCacheMathTest`.

**Kafka outbox + intake.** `docker compose -f docker/docker-compose.yml --profile kafka up -d`, run with
`KAFKA_ENABLED=true`. Break creation (REST or Kafka), proposal routing and resolution write an `outbox_event` row in the
same transaction; `OutboxRelay` publishes them to `reconcile.break.events` (key = break id, headers `event-id`,
`event-type`, `tenant-id`; at-least-once, `FOR UPDATE SKIP LOCKED`, per-key order preserved). Send a JSON message to
`reconcile.break.intake` to create a break:

```json
{"tenantId":"TENANT_DEMO","breakReference":"KAFKA-1","ledgerAmount":"1200.50","externalAmount":"1000.00","currency":"USD"}
```

Duplicates are ignored (idempotent on tenant + reference); malformed messages go to `reconcile.break.intake.dlt`
without retry. Metrics: `reconcile_outbox_published_total`, `reconcile_intake_total{outcome}`. Tests: `KafkaOutboxIT`
(real broker). With Kafka off, events still accumulate in the outbox table and nothing else changes.

**Web UI.** See `ui/README.md` (`ui/README.md`). In short: run the API with
`-Dspring-boot.run.profiles=mock-llm,demo-login`, then `cd ui; npm ci; npm start` and open http://localhost:4200.

**Observability.**

```powershell
docker compose -f docker/docker-compose.yml --profile observability up -d   # Postgres, Jaeger, Prometheus, Grafana, Alertmanager, Mailpit
$env:TRACING_ENABLED = "true"; mvn spring-boot:run
curl.exe -s http://localhost:8080/actuator/prometheus | Select-String reconcile_llm
```

- Grafana: http://localhost:3000, dashboard *reconcile-ai: LLM cost, latency and resilience* (estimated spend,
  tokens/min, LLM p95, breaker state, retries, PII masked, HTTP p95, LLM errors).
- Jaeger: http://localhost:16686, service `reconcile-ai`. Each investigation job is one span (`investigation-job`)
  with the Spring AI chat-client and model spans nested under it, linked to the `POST /investigations` request.
- Cost prices: `LLM_PROMPT_USD_PER_MILLION`, `LLM_COMPLETION_USD_PER_MILLION` (defaults: gpt-4o-mini list price).

**Alerts by e-mail.** Prometheus evaluates `docker/observability/alerts.yml` and hands firing alerts to Alertmanager,
which e-mails them. Locally the mail goes to Mailpit, a catch-all inbox at http://localhost:8025, so nothing real is
sent. Alerts: `ReconcileAiDown` (critical), `LlmCircuitOpen` (critical), `LlmErrorRateHigh`, `LlmSpendBurnHigh`, `InvestigationFailuresHigh`,
`OutboxBacklogGrowing`, `PiiNerUnavailable`, `AgentToolDenials`, `AuditAnchorWriteFailing`, and `AuditAnchorMismatch`
(critical). Critical alerts go out immediately; warnings are grouped for 5 minutes; an open breaker mutes the
error-rate and failed-job warnings it explains. To try it, run the app with `mock-llm`, add `simulate provider outage`
evidence to 5 breaks and start investigations: the breaker opens and, once it has stayed open for a minute, the e-mail arrives. This was run end to end against the real app on 2026-11-10 (`docs/proof/4c-outage-drill-2026-11-10.md`).
- Every rule has a unit test in `docker/observability/alerts.test.yml` (`promtool test rules`, 16 cases covering both
  firing and staying quiet); CI runs it and `amtool check-config` in the `alerts` job.
  `ObservabilityIT.seriesUsedByTheAlertRulesAreExposed` fails if a metric the rules depend on is renamed.
- The spend threshold (5 USD/hour) is a literal in the rule because Prometheus cannot read the app's environment;
  change it there. Real SMTP: edit `global.smtp_*` in `alertmanager.yml`, keep credentials out of git.
- Proof: `docs/proof/4c-alert-email.txt` (`docs/proof/4c-alert-email.txt`).

---

## Demo

With the app running, this script walks the whole golden path against the live API (break, evidence,
investigation, policy route, approval, RESOLVED, audit chain check):

```powershell
.\scripts\demo-golden-path.ps1 -Pause      # -Pause waits for Enter before each step, for screen recording
```

---

## 6. Test suite

Current counts in the default build: **123 unit tests** (surefire), **179 integration tests** (failsafe, real Postgres, Redis, Kafka, Keycloak, Presidio and MinIO via Testcontainers) and **15 UI specs** (Karma).
Tests tagged `openai`, `ollama` and `bedrock` need a real key, a running Ollama or AWS credentials and are excluded from the default build. `gateway` needs the platform spike stack (Keycloak and ai-gateway with a stub provider, free) and `gateway-live` is the one paid run (`docs/proof/q2-7-gateway-live-runbook.md`); both are excluded too.

| Class | Type | Proves |
|---|---|---|
| `MoneyAndVarianceTest` | unit | BigDecimal money, currency mismatch rejected, variance = abs difference |
| `RoleTest` | unit | Only the four known roles parse |
| `PolicyGateServiceTest` | unit | Routing thresholds: citations, root cause, confidence, currency, variance |
| `CitationVerifierTest`, `InjectionScannerTest` | unit | Citation validity; injection pattern detection |
| `ResolutionPathArchitectureTest` | unit | `ResolutionCommitService` has exactly one caller |
| `SecurityAndTenantIsolationIT` | IT | 401/403/404 rules, client-supplied tenant ignored — **release-blocking** |
| `VirtualThreadsTenantIsolationIT`, `TenantScopeVirtualThreadTest` | IT, unit | Under Java 21 virtual threads, 120 interleaved requests never see another tenant's rows and every request runs on a virtual thread; `TenantScope` is per thread and never inherited — **release-blocking** |
| `BreakAndEvidenceIT` | IT | Create/fetch, duplicate reference 409, evidence dedup |
| `EvidenceContentIT` | IT | Uploaded bytes download identically; duplicate keeps one copy; other tenant 404; unknown id 404; pre-V9 row reports CONTENT_NOT_AVAILABLE |
| `TokenBucketRateLimiterTest`, `RateLimitIT` | unit, IT | Burst then refill, per-tenant buckets; third request of a burst gets 429 + `Retry-After`; other tenant unaffected |
| `SwaggerEnabledIT`, `SwaggerDisabledByDefaultIT` | IT | Spec and UI served only when enabled; default config exposes nothing |
| `EvalCaseLoaderTest` | unit | Case files validate (enums, evidence exists, ids unique); every root cause has a case; metrics, percentiles and cost arithmetic |
| `InvestigationEvalIT` | IT | All cases through the real pipeline on mock-llm; report written; metrics stay at or above `thresholds-mock.json`; nothing routes to AUTO_RESOLVE |
| `RetrievalEvalIT` | IT | recall@5, MRR and hit@1 for the dense leg, the keyword leg and the fused result; hybrid is at least as good as either leg |
| `BedrockFakeEvalIT` | IT | All 28 eval cases through the Bedrock adapter against a fake Converse endpoint using the mock-llm rules; scores must equal the mock-llm floor (adapter fidelity) |
| `BedrockAdapterIT` | IT | Bedrock adapter against a fake Converse endpoint: parsing, usage and cost, output cap, throttle retry, no retry on bad input, breaker opens on outage |
| `ObservabilityIT` | IT | Prometheus exposes the series the alert rules query (a rename would silently disable an alert) |
| `alerts.test.yml`, `infra/**/tests/*.tftest.hcl` | promtool, terraform test | 14 alert-rule cases; 38 Terraform checks on a mock AWS provider (run in CI, outside Maven) |
| `EvidenceIngestionIT`, `HybridRetrievalIT` | IT | Ingestion into pgvector; tenant-filtered hybrid retrieval |
| `InvestigationFlowIT`, `GoldenPathIT` | IT | Evidence → cited proposal → route → approval → RESOLVED; injection quarantine |
| `InvestigationRecoveryIT`, `InvestigationStreamingIT` | IT | Orphaned job recovery; SSE lifecycle and tenant isolation |
| `AuditChainIT` | IT | Hash chain verifies; append-only trigger rejects updates |
| `AuditAnchorIT` | IT | Real MinIO with Object Lock: signed anchor written once, locked object cannot be deleted, consistent chain rewrite detected, SRE-only and tenant-scoped, signatures cannot be forged |
| `LlmResilienceTest`, `LlmResilienceIT` | unit, IT | Transient-only retry, breaker opens and fails fast, outage fails the job retryably |
| `PiiMaskerTest` | unit | Identifiers masked; amounts, dates, references and chunk ids untouched; advisor masks what the model sees |
| `NerMaskingTest`, `PresidioNerIT` | unit, IT | Real Presidio masks names and places, keeps references and chunk ids; overlapping spans; fail-open keeps identifier masking, fail-closed refuses the call |
| `KafkaOutboxIT` | IT | Outbox rows published with headers; intake creates one break despite redelivery; malformed to DLT |
| `SemanticCacheIT` | IT | Exact and semantic hits; tenant isolation; evidence-snapshot rule; suspect note bypass |
| `DemoLoginIT` | IT | Persona tokens pass normal JWT validation; roles still enforced; unknown persona rejected |
| `RowLevelSecurityIT` | IT | Unscoped connection sees nothing; queries without a tenant filter still isolated; cross-tenant insert rejected by Postgres; app role cannot disable RLS; a JWT cannot claim system scope |
| `SseFanoutIT` | IT | Two app instances share Postgres and Redis; a stream on node 2 receives every stage of a job running on node 1 |
| `DailyDigestIT` | IT | Digest numbers match a scripted day; idempotent re-run; outbox event; tenant isolation and roles; second instance skips while the lock is held |
| `KeycloakIT` | IT | Real Keycloak: tokens carry tenant/roles/audience; tenant comes from the IdP; roles enforced; no-tenant user, other realm and dev-key tokens rejected |
| `ObservabilityIT` | IT | Prometheus exposes LLM/breaker/HTTP series and no tenant ids; other actuator endpoints closed |
| `HttpGatewayChatAdapterTest`, `IdempotencyKeysTest`, `GatewayConfigTest`, `GatewayFailureTest` | unit | ai-gateway client: request matches `chat-request.v1.json`, every fake reply matches `chat-response.v1.json` or `error.v1.json`; token with the gateway audience scope, cached; one retry only on not-sent codes, never on an unknown outcome or budget block; one call-log attempt per request; refusal codes map to typed reasons, unknown-outcome and policy-blocked sets are disjoint |
| `GatewayUnknownOutcomeIT` | IT | 504 `UNKNOWN_OUTCOME` and 409 `IDEMPOTENCY_UNKNOWN_OUTCOME` through the whole app: exactly one request under `recon:<jobId>:investigate:1`, job FAILED and flagged, re-run refused (409 `OUTCOME_UNRECONCILED`) until a second person reconciles; the re-run is a new job with a new key — **release-blocking for the gateway route** |
| `GatewayOutcomeGuardIT` | IT | The reconcile endpoint: roles, a different person than the starter, identifier-shaped reference, other tenant 404, recovery on the direct route never flags a job |
| `GatewayLedgerIT` | IT | `gateway_call` links `jobId → key → invocationId` for success, budget, risk and sensitivity (each resolvable on the gateway ledger); auth and schema rejections are logged without an invocation id; a success also writes the `llm_call` row (`ai-gateway`) with the gateway's token counts; the token form carries `scope=aud-ai-gateway` |
| `GatewaySensitiveDataIT` | IT | An IBAN and a card number in the evidence reach the gateway as `[IBAN]` and `[CARD_NUMBER]`, never raw; `SENSITIVITY_BLOCKED` and `RISK_BLOCKED` fail the job with the typed reason, create no proposal and send the break to MANUAL_REVIEW_REQUIRED (a person), not FAILED_RETRYABLE; a re-run is allowed without the reconcile step |
| `GatewayBudgetIT` | IT | 429 `BUDGET_BLOCKED` (with `Retry-After`) and `RATE_LIMITED`: one request, no spend, job FAILED with the typed reason, break FAILED_RETRYABLE, not outcome-unknown; a plain re-run succeeds once the budget is back |
| `GatewayReconcileLedgerIT` (6) | IT | Reconcile reads the gateway ledger before recording anything: the ledger fields and audit detail on success, the refusals (`INVOCATION_NOT_FOR_JOB`, `LEDGER_ENTRY_NOT_FOUND`, `GATEWAY_LEDGER_UNAVAILABLE`), a ticket reference that skips the read, and the `gateway-calls` view |
| `TenantGatewayTokensTest` (6) | unit | Per-tenant client choice and the refusal codes `TOKEN_TENANT_MISMATCH`, `TOKEN_TENANT_UNVERIFIED` and `TENANT_CREDENTIALS_MISSING`, with no fallback to a shared client |
| `GatewayPerTenantCredentialsIT` (4) | IT | Per-tenant credentials through the whole app against a fake gateway and realm; a token for the wrong tenant or a tenant with no entry fails before sending (`NOT_SENT`, http 0, break `FAILED_RETRYABLE`) |
| `RedisTokenBucketRateLimiterIT` (4) | IT | Real Redis (Testcontainers): two replicas share one bucket, 40 concurrent requests allow exactly 10, TTL on the key, fallback to the local bucket when Redis is unreachable |
| `SharedRateLimitIT` (2) | IT | The app answers 429 after another replica has spent the burst |
| `InvestigationQueueMetricsTest` (1) | unit | The gauges `reconcile.investigations.executor.active`, `.queued` and `.capacity` |
| `RealGatewayIT`, `GatewayLiveProviderIT` | IT, opt-in (`gateway`, `gateway-live`) | The same flow against the real ai-gateway: the realm's token is accepted and every stored invocation id resolves on the gateway ledger under our token. Stub provider (free) and one approved paid run; both write an evidence block to `target/` |

## Quality: evaluation harness

`src/test/resources/eval/cases/` holds 28 labelled synthetic breaks: three or more per root cause, plus hard ones
(conflicting evidence, a quietly worded instruction hidden in evidence, an obvious injection, vague notes, a variance
large enough for L2, a non-USD currency). Each case says what a careful reviewer would decide: root cause, route and
which evidence file must be cited. `InvestigationEvalIT` runs every case through the public API and the real pipeline
(retrieval, prompt, citation check, policy gate), then writes `target/eval/report-<label>.md` and `.json`.

| Run | Chat model | Used for | Fails the build? |
|---|---|---|---|
| `InvestigationEvalIT`, `RetrievalEvalIT` | deterministic `mock-llm` | Regression gate on retrieval, prompt, verifier and gate | Yes, below the floor in `thresholds-mock.json` |
| `OpenAiInvestigationEvalIT` | real OpenAI (`OPENAI_API_KEY`) | Real quality, latency and cost numbers | No, it reports only |

The mock only recognises fee and timing wording, so its accuracy is low by design (42.9% root cause on the current set).
Those numbers guard the pipeline; they say nothing about model quality. Retrieval is measured with five distractor files
from other root causes in the same break; on the test hashing embeddings the current numbers are:

| Leg | recall@5 | MRR | hit@1 |
|---|---|---|---|
| Dense (vector) | 0.42 | 0.35 | 0.29 |
| Keyword (BM25) | 0.83 | 0.49 | 0.25 |
| Hybrid (RRF) | 0.83 | 0.57 | 0.38 |

```powershell
mvn verify -Dit.test=InvestigationEvalIT,RetrievalEvalIT          # mock run, free, ~2 minutes
$env:OPENAI_API_KEY = "..."                                       # real run: a few cents; reports, never gates
mvn verify -Dfailsafe.excludedGroups=ollama -Dit.test=OpenAiInvestigationEvalIT
.\scripts\eval-compare.ps1 target\eval\report-mock-llm.json target\eval\report-openai-gpt-4o-mini.json
```

Optional human scoring: fill `eval/human-scores.csv` (clarity and faithfulness, 1 to 5); averages are added to the report.
Real-model run (2026-11-09, gpt-4o-mini, temperature 0, 28 cases, test hashing embeddings so only chat is real):

| | Single call | Agent |
|---|---|---|
| Root-cause accuracy | 96.4% | 100% |
| Route accuracy | 89.3% | 57.1% (right cause, low confidence, so manual review) |
| Citations verified / required evidence cited | 96% / 100% | 100% / 100% |
| Latency p50 | 2.5 s | 9.5 s |
| Cost per case | $0.00015 | $0.0012 |

Single call stays the default: on this data the agent costs 8x for no real gain. The first agent run exposed a bug (model-written search queries found nothing), fixed and covered by `AgentSearchQueryIT`. Details: `docs/proof/phase4-openai-eval.md`. 28 synthetic cases is a sanity check, not a benchmark.

**Supervisor agents (4G), measured 2026-11-13 on gpt-4o-mini, 28 easy + 12 hard cases:** the rule fixed in advance (halve wrong causes reaching an approver, at most 5x single-call cost) was not met. No mode let a wrong cause reach an approver on the hard set, so there was nothing to halve; the supervised modes cost 7.9x and 12.3x single call; the reviewer correctly challenged the unsupported-citation case but sent more cases to manual review (route 41.7% vs 75.0%). The model-planned variant was weakest (50% root cause on the hard set). The traces show why: after the reviewer agreed, the supervisor model often started the analyst again, the second analyst ran out of budget, and its empty draft replaced the reviewed one. The code now refuses any delegation once the current draft is reviewed and agreed (`ALREADY_REVIEWED`, covered offline by `SupervisedInvestigationIT`); this fix has not been re-measured live. Single call stays the default and the supervised modes stay off. One model, one run, 40 cases. The first run used a budget that was too small and the budget was raised afterwards; both runs are kept. Details: `docs/proof/phase4g-supervised-eval.md`.

---

## 7. Key design decisions

| Decision | Why |
|---|---|
| `Repository` instead of `JpaRepository` | Removes tenant-blind inherited methods at compile time |
| Custom `JwtDecoder` bean | A token without a tenant would otherwise cause a 500 |
| `@EnableMethodSecurity` | Without it, `@PreAuthorize` is silently ignored |
| No catch-all exception handler | It would turn security 403s into 500s |
| 404 instead of 403 cross-tenant | 403 confirms that the resource exists |
| Model proposes, Java decides | One hallucinated sentence must not cause a ledger write |
| Hybrid RRF retrieval | Narrative text favours dense search; reference codes and amounts favour exact match |
| Singleton Testcontainers base | `@Container` in a shared base class breaks the second IT class through context caching |
| Amounts as JSON strings on input (requests, Kafka) | Avoids binary floating-point when parsing; responses still return JSON numbers |
| Self-signed RSA dev keys | No authorization server to run locally; swapping in a real IdP only changes the public key config |
| Virtual threads for requests, bounded platform pool for investigations | Requests are I/O-bound; the investigation pool's full queue is the 503 back-pressure and must stay bounded. Isolation under virtual threads: `VirtualThreadsTenantIsolationIT` |
| `GatewayChatPort` behind `GATEWAY_ENABLED` (default off) | The investigation model call can go through ai-gateway (`POST /api/v1/chat`, service token, idempotency key `recon:<jobId>:<step>:<attempt>`). The key is fixed per job step, so a re-claimed job or a transport retry presents the same key and the gateway replays instead of billing twice. One retry at most, only on codes where nothing reached the provider, never on an unknown outcome. Every attempt is written to `gateway_call` (V13) so a job reconciles as `jobId → key → invocationId` against the gateway ledger; a success also lands in `llm_call` as model `ai-gateway`. A policy refusal (`RISK_BLOCKED`, `SENSITIVITY_BLOCKED`) sends the break to manual review, not the retry sweeper; a budget block stays retryable; an unknown outcome blocks the re-run until a second person reconciles it. Identifiers are masked before the request leaves. Tests: the gateway rows in §6, against a fake gateway validated on the contract schemas, plus the opt-in runs against the real gateway |
| Reconcile reads the gateway ledger itself | `POST /investigations/{jobId}/reconcile` (APPROVER_L2, SRE_ADMIN). When the reference is a canonical UUID and the gateway route is on, the app calls `GET /api/v1/invocations/{id}` with the tenant's token before it records anything. The id must be one returned to this job's own gateway calls, otherwise 409 `INVOCATION_NOT_FOR_JOB` and no read. Ledger 404 gives 409 `LEDGER_ENTRY_NOT_FOUND`; an unreachable ledger or a non-200 answer gives 503 `GATEWAY_LEDGER_UNAVAILABLE` with `Retry-After: 30` and nothing is recorded. On success the job stores `ledger_execution_certainty` and `ledger_status` (V14) and the audit event `INVESTIGATION_OUTCOME_RECONCILED` records `ledgerChecked` plus certainty, status, tokens and cost. A non-UUID ticket reference still works and skips the read (`ledgerChecked=false`). The person still decides, the starter still cannot reconcile (409 `RECONCILER_MUST_DIFFER`), and an `EXECUTED` ledger answer does not block reconciling. `GET /investigations/{jobId}/gateway-calls` (APPROVER_L2, SRE_ADMIN) lists each `gateway_call` row with `ledgerResult` (`FOUND`, `NOT_FOUND`, `UNAVAILABLE`, `NOT_CHECKED`) and the ledger fields; another tenant's job is 404. The job response carries `outcomeUnknown`, `reconciledAt`, `reconciledBy`, `reconcileRef`, `ledgerExecutionCertainty` and `ledgerStatus`. Test: `GatewayReconcileLedgerIT` |
| Per-tenant gateway credentials | `reconcile.gateway.tenants` maps a tenant to its own client, for example `"[TENANT_A]": { client-id: ..., client-secret-env: GATEWAY_SECRET_TENANT_A }`. The secret is read from the named environment variable and never from YAML. An empty map keeps the old shared client. When the map is set, each job's call uses its tenant's client and the token's `tenant_id` claim must equal the job tenant exactly (`TOKEN_TENANT_MISMATCH`, or `TOKEN_TENANT_UNVERIFIED` when no claim can be read; the signature is not checked locally because the gateway verifies it). A tenant with no entry fails with `TENANT_CREDENTIALS_MISSING`; there is no fallback to a shared client. These fail before anything is sent: job failure reason `GATEWAY_<code>`, break `FAILED_RETRYABLE`, `gateway_call` outcome `NOT_SENT` with http 0. Startup fails fast when an entry is incomplete, a secret variable is unset, or two keys collide after case and separators are ignored; the message names tenants and variable names only. Tests: `TenantGatewayTokensTest`, `GatewayPerTenantCredentialsIT` (fake gateway and realm only) |
| Rate limit shared through Redis | `reconcile.ratelimit.store` is `memory` (default, per JVM) or `redis` (`RATE_LIMIT_STORE`). In Redis mode there is one token bucket per tenant shared by all replicas, updated by one atomic Lua script on the Redis server clock. The key is `reconcile:ratelimit:investigations:{tenant}` and it expires when the bucket would be full. If Redis is unreachable the call falls back to the local per-JVM bucket and `reconcile.ratelimit.redis.fallback` counts it. Tests: `RedisTokenBucketRateLimiterIT`, `SharedRateLimitIT` |
| Autoscale on queued work (KEDA overlay) | Investigations wait mostly on the model, so CPU is a weak load signal. Gauges `reconcile.investigations.executor.active`, `.queued` and `.capacity` expose the pool. `k8s/overlays/keda` removes the base CPU HPA and adds a KEDA ScaledObject (2 to 6 replicas) with a Prometheus trigger on active plus queued (threshold 3 per pod) and a CPU 70% trigger, and sets `RATE_LIMIT_STORE=redis` so replicas share one bucket. CI renders the overlay and checks it with kubeconform against the datreeio CRDs-catalog schema. Validated only, not run on a cluster with KEDA. Test: `InvestigationQueueMetricsTest` |

## 8. Known limitations

- SSE fan-out over Redis pub/sub is fire-and-forget: an event missed during a Redis blip is recovered by the next event or the snapshot on reconnect, not replayed.
- Kafka delivery is at-least-once; consumers must dedupe on the `event-id` header. Intake trusts the tenant named in the message, so topic ACLs are the trust boundary.
- The Keycloak realm in `docker/keycloak/` is a local demo (synthetic users, shared password, `start-dev`); a production IdP needs HTTPS, its own hostname and managed users.
- PII masking is not DLP. Without the Presidio sidecar only structured identifiers are masked; with it, names and locations are masked with model accuracy (it can miss unusual names and occasionally mask a non-name). Free-form disclosures (for example a medical detail) are not detected.
- Audit anchors protect history up to the latest anchor; events written since then are covered only by the in-database chain until the next cycle (15 minutes by default). Anchors prove the chain was not changed afterwards, not that each event was true when written.
- The base HPA scales on CPU only. The `k8s/overlays/keda` overlay replaces it with a KEDA ScaledObject that also scales on active plus queued investigations, but it has been rendered and schema-checked (kubeconform, datreeio CRDs-catalog schema) only. It has not been run on a cluster with KEDA installed.
- The circuit breaker is per instance; the cost metric is an estimate from list prices, not an invoice.
- The investigation rate limit is held in memory per instance by default (`RATE_LIMIT_STORE=memory`), so with N replicas a tenant can reach N times the configured rate. With `RATE_LIMIT_STORE=redis` the bucket is shared by all replicas. If Redis is unreachable, each replica falls back to its own local bucket and counts `reconcile.ratelimit.redis.fallback`, so the shared limit is not enforced during an outage.
- Evidence bytes are stored in Postgres (`bytea`), which suits the 10 MB upload cap. Evidence uploaded before migration V9 has no stored original. Object storage is the upgrade for larger files.
- The Trivy image scan in CI reports findings but does not fail the build yet.
- Auto-resolve is implemented but disabled by configuration (ADR-9).
- The ai-gateway route (`GATEWAY_ENABLED=true`) is proven end to end through the whole application against a fake gateway whose every reply is validated on the gateway's published v1 schemas; the runs against the real gateway (`RealGatewayIT`, free, and `GatewayLiveProviderIT`, paid) are built and opt-in but **have not been run yet**. On that route the gateway's own governance applies and the local advisor chain and circuit breaker are bypassed; identifier masking still runs before the request leaves. A gateway refusal fails the job with reason `GATEWAY_<code>`; a policy refusal sends the break to manual review; the audit entry records the gateway invocation id. After an unknown outcome (504, 409 `IDEMPOTENCY_UNKNOWN_OUTCOME`, a transport failure after sending, or a node lost mid-call on the gateway route) the job is flagged `outcome_unknown` and `POST /breaks/{id}/investigations` answers 409 `OUTCOME_UNRECONCILED` until an L2 approver or SRE admin other than the starter posts `/investigations/{jobId}/reconcile` with a gateway invocation id or ticket id (V12). When the reference is an invocation id, the app reads the gateway ledger itself before it records anything (see section 7); the person still decides, and an `EXECUTED` ledger answer does not block reconciling. A ticket reference skips the read. The ledger read is proven only against the fake gateway so far; `RealGatewayIT` reads the real ledger when it is run.
- Per-tenant gateway credentials (`reconcile.gateway.tenants`) are tested against a fake gateway and a fake realm only. A real two-tenant run needs a second Keycloak client. The token's `tenant_id` claim is compared locally without verifying the signature; the gateway verifies the token. With the map empty, one shared client serves every tenant and the gateway bills the tenant in that client's token, not the job's tenant.
- The RSA keys in `src/test/resources/jwt` and `scripts/jwt` are throwaway development material. Do not reuse them outside local development.
- Answer quality is measured on 28 synthetic cases (see Quality above), not on real customer data, and the real-model runs use test hashing embeddings, so the dense retrieval leg is not semantic there. The Bedrock live run and the AWS deployment have not been run against a real account.

---

## 9. Roadmap: Phase 4

Each item is off by default, gets its own test on real or faithfully faked infrastructure, and ships with a measured
number. Nothing calls a paid API or creates cloud resources unless explicitly enabled.

| Order | Feature | Design in one line | Local testing | Real mode |
|---|---|---|---|---|
| 4A | **Evaluation harness** (built; live run done) | 24–30 labelled synthetic breaks run through the real pipeline; root-cause and route accuracy, citation precision/recall, retrieval recall@5 and MRR per leg, p95 latency, tokens and cost per investigation | `mock-llm` baseline as a CI regression gate | OpenAI run, numbers published in `docs/proof/` |
| 4B | **LangChain4j investigation agent** (built; live run done) | Optional `AGENT` mode: the model picks read-only tools (`getBreakSummary`, `listEvidence`, `searchEvidence`, `findSimilarResolvedBreaks`); tenant bound from the job, never a tool argument; tool output masked and injection-scanned; every call audited; the agent only drafts, so citation check, policy gate and maker-checker are unchanged | Scripted fake model (19 agent ITs, 9 unit tests); enable with `AGENT_ENABLED=true`, start with `{"mode":"AGENT"}`; tool calls at `GET /api/v1/investigations/{id}/tool-calls` | OpenAI; `OpenAiAgentEvalIT` single-call vs agent comparison |
| 4C | **Alert delivery** (built) | Alertmanager routes Prometheus alerts (circuit open, LLM error rate, spend burn, outbox backlog, NER down, anchor failures) to e-mail | Mailpit SMTP sink; `promtool` rule tests (16) in CI; real-app outage drill recorded | Real SMTP: edit `alertmanager.yml`. **Live delivery to a real mailbox not yet tested** |
| 4D | **Amazon Bedrock** (built; adapter-fidelity eval done, **live run pending AWS credentials**) | `bedrock` chat profile next to `openai` and `ollama`; embeddings stay on OpenAI | WireMock-stubbed Converse API; `BedrockFakeEvalIT` scores identical to mock-llm through the adapter | AWS credentials for the live row of the provider comparison |
| 4E | **AWS deployment** (built; applied and destroyed against LocalStack, **real apply pending AWS credit and your go-ahead**) | Terraform: ECS Fargate, RDS Postgres + pgvector, S3 Object Lock anchors, Secrets Manager, GitHub OIDC; deploy, record, destroy | `terraform validate`/`test` with mock providers, tflint, checkov; `envs/localstack` real apply + destroy in CI (network, secrets, storage, ci modules) | Manual, approval-gated apply |
| 4G | **Supervisor agents** (built; live comparison done, rule not met) | `SUPERVISED` (fixed order) and `SUPERVISED_PLANNED` (supervisor model) modes: analyst, risk checker and reviewer under one shared budget; the reviewer can only escalate to manual review through a deterministic gate rule; off by default | Scripted models offline (12 hard cases; `SupervisedInvestigationIT` (10), `SupervisedDisabledByDefaultIT`, `SupervisedEvalIT`, `ReviewerVerdictTest` (9), `EvalRunnerCitedFilesIT` (2)); live run 2026-11-13 on gpt-4o-mini, 40 cases x 4 modes | Live result: hard set, wrong causes to an approver 0 in every mode, so nothing to halve; cost 7.9x (supervised) and 12.3x (planned) single call against a 5x limit; planned mode only 50% root cause on the hard set. Published as a negative result: `docs/proof/phase4g-supervised-eval.md`. One model, one run, 40 cases |
| 4C | **Alert delivery** (built) | Alertmanager routes Prometheus alerts (circuit open, LLM error rate, spend burn, outbox backlog, NER down, anchor failures) to e-mail | Mailpit SMTP sink; `promtool` rule tests (16) in CI; real-app outage drill recorded | Real SMTP: edit `alertmanager.yml` |
| 4D | **Amazon Bedrock** (built; adapter-fidelity eval done, live run pending) | `bedrock` chat profile next to `openai` and `ollama`; embeddings stay on OpenAI | WireMock-stubbed Converse API; `BedrockFakeEvalIT` scores identical to mock-llm through the adapter | AWS credentials for the live row of the provider comparison |
| 4E | **AWS deployment** (built; applied and destroyed against LocalStack, never against a real account) | Terraform: ECS Fargate, RDS Postgres + pgvector, S3 Object Lock anchors, Secrets Manager, GitHub OIDC; deploy, record, destroy | `terraform validate`/`test` with mock providers, tflint, checkov; `envs/localstack` real apply + destroy in CI (network, secrets, storage, ci modules) | Manual, approval-gated apply |

Architecture after Phase 4:

```
InvestigationService ─┬─ SINGLE_CALL ─► InvestigationAiClient (Spring AI advisors) ─┐
                      └─ AGENT ───────► InvestigationAgent (LangChain4j, read-only   ├─► ProposalDraft ─► CitationVerifier
                                        tools, AgentRunContext, audited calls)    ──┘      ─► PolicyGate ─► maker-checker
Chat provider by profile: mock-llm | ollama | openai | bedrock | bedrock-fake      (embeddings: OpenAI, unchanged)
Prometheus ─► Alertmanager ─► e-mail                AWS: ALB ─► Fargate ─► RDS pgvector · S3 Object Lock · Secrets Manager
```

---

## License

Copyright (c) 2026 Yellaiah Mekala. Licensed under the Apache License 2.0; see [LICENSE](LICENSE).

---

## Author

**Yellaiah Mekala** — Senior Java backend engineer focused on AI integration.

- GitHub: [github.com/YellaiahMekala](https://github.com/YellaiahMekala)

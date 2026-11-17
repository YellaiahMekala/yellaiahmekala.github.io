> **Copy of a private repository document**, snapshot taken 2026-11-17. The source code is private, so links to files in the repository are shown as plain paths. Status labels are the author's own, as written at the time of the snapshot. All data is synthetic.


# Technical Design Document
## AI Gateway — Enterprise LLM Governance Control Plane

| Field | Value |
|---|---|
| Document | TDD — version 3 (canonical) |
| Supersedes | All earlier TDD drafts and `AI_GATEWAY_PROJECT_BRIEF.md` §4 |
| Companion docs | `BRD.md`, `ARCHITECTURE.md`, `IMPLEMENTATION_PLAN.md`, `SYSTEM_DESIGN.md` |
| Status | Design baseline, written before implementation. Built so far (see `PHASE_TRACKER.md`): Phases 0–13 committed (Phase 11 offline; Phase 12 in the `ops-agent` repository); Phase 14 committed (`8bf83df` and `2670535`, both CI green; `2670535` passed only on re-run, see the flaky outbox test in `PHASE_TRACKER.md`) (2026-11-14) |

---

## 1. Module Structure

```
ai-gateway/
├── pom.xml                 parent — dependency management only
├── gateway-core/           domain, ports, policies, normalized contracts,
│                           budget state machine, routing decision logic,
│                           cost arithmetic, failure taxonomy.
│                           NO Spring MVC/WebFlux, NO JPA, NO Redis,
│                           NO provider SDK, NO transport classes.
├── gateway-adapters/       port implementations: provider adapters,
│                           JPA repositories + entities, Flyway migrations,
│                           Redis cache and projection, pgvector store,
│                           Kafka producer. Depends on gateway-core.
├── gateway-api/            Spring Boot application: security, controllers,
│                           SSE, orchestration, transaction boundaries,
│                           wiring. Depends on core + adapters.
├── gateway-eval/           datasets, loader, CI tests (evaluators live in core; baseline comparison is a future phase)
│                           comparison CLI. Test/CI module, NOT a deployable.
│                           Depends on core contracts only.
├── usage-aggregator/       separate deployable (Phase 8, built) — created only when Kafka
│                           arrives. Event contract only, no compile
│                           dependency on the gateway.
├── gateway-console/        React 19 + Vite + TypeScript operator console (Phase 9, built).
│                           A client of the protected API only; Vitest unit
│                           tests, Playwright golden path. Not a Maven module.
├── docker/ deploy/aws/ demo/ load/ docs/
```

**Dependency direction, ArchUnit-enforced:**

```
gateway-eval ──► gateway-core ◄── gateway-adapters ◄── gateway-api
usage-aggregator ──(published event contract only)
```

The rule that makes "framework-independent governance" a real claim rather than a slogan:
`gateway-core` compiles with no Spring, no JPA, no Redis client and no provider SDK on its
classpath. If a governance decision cannot be expressed without one of those, it is not a
governance decision.

**Honest limit.** ArchUnit enforces dependency direction; it does not create distributed boundaries.
`core` + `adapters` + `api` build one deployable with enforced internal seams. Only
`usage-aggregator` is a second process, and only from the Kafka phase. See ADR-2 and ADR-17.

---

## 2. Consistency Model — read this before anything else

**There is no atomic transaction across Postgres and Redis.** Any design that implies one is wrong.
Budget enforcement therefore works as a *recoverable dual write* with Postgres authoritative:

```
1. BEGIN  (Postgres)
2. SELECT ... FOR UPDATE on budget_window_usage for (tenant, window)
   — or optimistic version check
3. Compute available = limit − settled − active_reservations
4. If insufficient → abort, return BUDGET_BLOCKED (no provider call)
5. INSERT budget_reservation (state RESERVED, expires_at, idempotency_key)
   INSERT/UPDATE budget_window_usage (active_reservations += estimate)
6. COMMIT                            ← authoritative reservation now exists
7. Project synchronously to Redis (admission counter)
8. Projection failed? → do NOT call the provider.
   Return GOVERNANCE_UNAVAILABLE. Reconciler repairs the projection.
9. Call the provider
10. Settle: BEGIN → reservation → SETTLED with actual usage,
    active_reservations -= estimate, settled += actual → COMMIT → project
```

Redis is a **fast admission projection**, not the decider of record. Postgres decides availability.
A reconciler rebuilds the projection from durable rows and resolves stranded reservations. Redis
data loss cannot incorrectly permit or deny spend, because the authoritative numbers never lived
there.

This is stated as **recoverable dual-write consistency**, never as atomicity. ADR-16.

---

## 3. Domain Model

### 3.1 Canonical types (`gateway-core`)

```java
record TenantContext(String tenantId, String subject, Set<String> roles) {}
record NormalizedCompletionRequest(TaskClass taskClass, String promptId, int promptVersion,
                                   String resolvedPromptChecksum, List<Message> messages,
                                   GenerationSettings settings, ResponseSchema schema) {}
record CompletionResult(String output, FinishReason finishReason, UsageReport usage,
                        String providerRequestRef, SafetyMetadata safety,
                        ExecutionCertainty certainty) {}
record UsageReport(int inputTokens, int outputTokens, int cachedInputTokens,
                   UsageSource source) {}          // PROVIDER_REPORTED | ESTIMATED
record ModelDescriptor(String modelKey, ProviderId provider, String providerModelId,
                       Set<Capability> capabilities, int contextWindow, int maxOutput,
                       String region, Lifecycle lifecycle, String pricingVersion,
                       boolean fallbackEligible) {}
record RoutingDecision(ModelDescriptor selected, ModelDescriptor fallbackCandidate, String reason) {}
record Cost(BigDecimal amountUsd, String pricingVersion) {}
```

### 3.2 Persistent entities (`gateway-adapters`)

| Entity | Purpose | Notes |
|---|---|---|
| `budget_window_usage` | **Authoritative availability** per `(tenantId, window)` | `limit`, `settled`, `active_reservations`, `version`. Row-locked on reservation |
| `budget_reservation` | Reservation lifecycle | `RESERVED → SETTLED \| RELEASED \| UNKNOWN_OUTCOME → EXPIRED \| SETTLED`. Carries estimate, actual, expiry, idempotency key, outcome reference |
| `invocation_ledger` | Minimised immutable evidence | Checksums, model/prompt/policy versions, routing decision, status, timing, usage, cost, pricing version, region, retention class. **No payload.** Partitioned by month, indexed `(tenant_id, created_at)` |
| `idempotency_record` | Request deduplication | `IN_PROGRESS \| COMPLETED \| UNKNOWN_OUTCOME`, request fingerprint, response reference, expiry. Unique on `(tenant_id, idempotency_key)`; the fingerprint is also bound to the token subject |
| `prompt_template` | Immutable versioned prompts | Checksum over system prompt + user template + generation settings. Partial unique index enforces one `ACTIVE` per `(tenant, promptId)` |
| `eval_case` / `eval_run` | Promotion evidence | Run records candidate checksum, pinned settings, pass/fail, threshold, `promotionAllowed` |
| `security_audit` | Detection events | Detector type, match count, risk score, action. **Never the matched value** |
| `model_descriptor` / `price_entry` | Capability catalogue and versioned pricing | Pricing carries `pricingVersion`, region, `effectiveFrom` |
| `semantic_cache_entry` | Opt-in semantic cache | Encrypted, minimised. Tenant filter **inside** the pgvector query |
| `outbox_event` | Event publication | Written in the ledger transaction; deleted only after a durable publish marker plus replay window |

### 3.3 Enumerations

```
Capability        CHAT, JSON_SCHEMA_OUTPUT, TOOL_CALLING, STREAMING, LONG_CONTEXT, EMBEDDING
TaskClass         SUMMARY, EXTRACTION, REASONING, CLASSIFICATION, EMBEDDING
InvocationStatus  OK, FALLBACK_OK, CACHE_HIT_EXACT, CACHE_HIT_SEMANTIC, IDEMPOTENT_REPLAY,
                  BUDGET_BLOCKED, RISK_BLOCKED, NOT_SENT, RETRYABLE_FAILURE,
                  PROVIDER_CONTRACT_FAILURE, UNKNOWN_OUTCOME, PARTIAL_STREAM,
                  CACHE_DEGRADED, GOVERNANCE_UNAVAILABLE
                  (+ SENSITIVITY_BLOCKED, built in Phase 13.2, §15.9)
DataSensitivity   PUBLIC, INTERNAL, RESTRICTED (built, Phase 13.2)
ReservationState  RESERVED, SETTLED, RELEASED, UNKNOWN_OUTCOME, EXPIRED
UsageSource       PROVIDER_REPORTED, ESTIMATED
ExecutionCertainty EXECUTED, NOT_EXECUTED, UNKNOWN
PromptStatus      DRAFT, CANDIDATE, ACTIVE, ROLLED_BACK
```

`UNKNOWN_OUTCOME`, `ExecutionCertainty` and `UsageSource` exist because the honest states are the
ones most systems omit, and they are what distinguishes this design in a technical interview.

---

## 4. Invocation Flow

| Step | Component | Behaviour |
|---|---|---|
| 1 | API + security | Validate JWT (issuer, audience, expiry, clock skew; RS256 only). Derive tenant and roles. Reject any tenant override. Validate idempotency key. |
| 2 | Idempotency | Claim the key as `IN_PROGRESS`. Concurrent same key → `409 IDEMPOTENCY_IN_PROGRESS`; no `202`, no polling. Completed key → replay stored outcome, status `IDEMPOTENT_REPLAY`. |
| 3 | Policy preparation | Resolve immutable prompt version; validate template variables; render deterministically; classify injection risk; apply masking; compute the resolved-prompt checksum. |
| 4 | Budget **eligibility** | Read-only availability check. Insufficient → `BUDGET_BLOCKED`, no reservation written. |
| 5 | Cache | Exact cache by full policy identity; semantic only for approved task classes. Hit → ledger row, no reservation ever taken. |
| 6 | Routing | Filter the catalogue by capability, residency, context limit and policy. Select primary and a fallback candidate. None eligible → `PROVIDER_CONTRACT_FAILURE` with the unmet capability named. |
| 7 | Budget **reservation** | The §2 sequence. Postgres authoritative, then Redis projection. Projection failure → `GOVERNANCE_UNAVAILABLE`, no provider call. |
| 8 | Provider executor | Invoke under timeout, bulkhead, bounded queue, per-tenant concurrency cap and circuit breaker. Classify any failure **before** deciding to retry. |
| 9 | Settlement | Persist outcome, actual usage and cost, reservation transition, ledger row and outbox event in one Postgres transaction. Project to Redis. |
| 10 | Response + telemetry | Return the result or an explicit status. Privacy-safe logs, traces and metrics. Best-effort console hint, published only after the ledger transaction commits (Phase 9). |

**Ordering rationale** — the likely deep-dive question:

- *Idempotency first* — nothing chargeable happens before the key is claimed.
- *Risk screening before masking* — masking alters text; screening must see what was sent.
- *Masking before cache* — unmasked text must never enter a cache tier.
- *Budget eligibility before cache, reservation after* — quota is never silently elastic for cached traffic, but a free hit takes no reservation and causes no reserve/release churn or race.
- *Routing before reservation* — the estimate depends on the selected model's pricing.
- *Ledger last and unconditional* — every terminal path writes exactly one row.

---

## 5. Provider Boundary

```java
public interface ModelProvider {                       // port, in gateway-core
    ProviderId id();
    Set<Capability> capabilities(String modelKey);
    CompletionResult complete(NormalizedCompletionRequest request);
    void stream(NormalizedCompletionRequest request, StreamListener listener);
    FailureClassification classify(Throwable t, ExecutionCertainty certainty);
}
```

Adapters translate **only at the provider edge**. Provider, model and capability are three
different things: routing selects an eligible *model*; the provider is merely the transport it lives
behind.

**Abstraction success criterion**, declared in advance: adding a provider requires a new adapter,
new capability and pricing metadata, and passing contract tests — nothing else. Any change to the
pipeline, the normalized contract or a caller is a leak, and ADR-15 records it rather than hiding a
quiet refactor.

### 5.1 Failure and resilience policy

| Failure class | Retry / failover | Recorded status |
|---|---|---|
| DNS, TLS, connection refused, breaker open — before send | May retry or fail over within bounded policy | `NOT_SENT` |
| Provider 429 or selected 5xx with retry semantics | May retry or fail over where policy and idempotency permit | `RETRYABLE_FAILURE` |
| Timeout after bytes sent; stream interrupted | **Never blind-fallback** | `UNKNOWN_OUTCOME` / `PARTIAL_STREAM` |
| Invalid or schema-mismatched response | No cross-provider retry unless the canonical contract makes it safe | `PROVIDER_CONTRACT_FAILURE` |
| Redis unavailable — cache | Continue without cache | `CACHE_DEGRADED` |
| Redis unavailable — budget projection | Reject before the provider call | `GOVERNANCE_UNAVAILABLE` |

Resilience is not claimed on circuit breakers alone. Required before the claim: breaker per
provider-model, bulkhead isolating provider pools, **explicit queue limits and timeout budgets**,
database / Redis / HTTP connection-pool ceilings, per-tenant concurrency caps, a defined overload
response, and cancellation propagated on client disconnect.

---

## 6. Prompt Lifecycle and Template Safety

Versions are immutable and checksummed over system prompt, user template and generation settings.

**Template safety** — a prompt registry is an injection surface if built carelessly:

- Variables are validated against a declared schema; unknown variables are rejected.
- No arbitrary expression evaluation in templates. Substitution only.
- Rendering is deterministic; the checksum covers the **fully resolved** prompt identity.
- System-message capabilities are restricted; a prompt owner cannot introduce unreviewed system-level instructions.

**Promotion** is one transaction: row-lock the active pointer, verify the candidate checksum matches
the evaluated artifact, verify the referenced `eval_run` permits promotion, verify policy approval,
flip the pointer, write an audit event. Any failure aborts the whole transaction. Rollback is the
same mechanism in reverse.

---

## 7. Caching

**Exact-cache identity must include**: tenant, `modelKey`, resolved-prompt checksum, prompt id and
version, **all** generation settings (temperature, top-p, max tokens, stop sequences, seed),
response schema or tool definitions, safety settings, locale, and policy version. Any omitted field
is a correctness bug.

**Semantic cache**: off by default, opt-in per tenant and task class, prohibited for regulated and
action-taking classes (enforced in code, not configuration), versioned by embedding model and
threshold, entries encrypted and minimised, tenant filter applied **inside** the pgvector query and
verified by negative tests. HNSW parameters (`m`, `ef_construction`, `ef_search`) are tuned with a
published recall-versus-latency curve.

**Both tiers store content.** That is the documented exception to the no-raw-payload rule (BRD §7):
tenant-namespaced, short TTL, encrypted at rest, excluded from backups, per-tenant purgeable, never
an audit source.

---

## 8. Usage, Cost and Estimation Honesty

Token counts computed before invocation are **provider-specific estimates**. Settlement uses
provider-reported usage. Where a provider reports none, the record carries `UsageSource.ESTIMATED`
and retains pricing provenance. **No invoice-level accuracy is claimed anywhere.**

Structured output policy: schema size limits enforced; schema-invalid output is distinguished from
unsafe content; retry only where pre-send safe or the provider supports idempotent replay.

---

## 9. Security

| Concern | Control |
|---|---|
| Authentication | JWT resource server — issuer, audience (`aud` must contain the configured value), expiry, clock skew 30 s, RS256 only, JWK set URI (refreshed by the decoder) or a fixed public key. One audience for users and services; no service-to-service audience separation |
| Authorisation | RBAC on every admin endpoint, with an authorization test per role |
| Identity | `TenantContext` only. A tenant header is never trusted, including from internal callers |
| Isolation | Tenant in every repository signature, cache key, budget key, event payload and operator filter; inside every vector query. Cross-tenant → `404` |
| Egress | Fixed provider base URLs · no caller-supplied URL or model · DNS/IP allow-list validation · TLS hostname validation · egress proxy and security-group rules · request-size limits |
| Secrets | Environment or secret manager only; documented rotation; never in source, config or image |
| Payloads | No prompt or response in logs or traces. Debug capture requires synthetic data, a feature flag, encryption, short retention, access audit, and **startup failure outside approved environments** |
| Supply chain | Pinned dependencies, SBOM, image scanning, documented vulnerability triage (scanner output is triaged, not treated as truth), signed images and provenance for cloud |

**Named honestly.** Masking is pattern-based detection, not DLP. Injection screening produces an
allow / annotate / block decision and is risk signalling, not containment — tool execution is out of
scope, so containment cannot be claimed from it. Model output is untrusted data and is schema-
validated where structure is required.

---

## 10. Observability and Readiness

Metrics use **bounded labels** — provider, model, status, task class. Never tenant id, request id,
prompt id or prompt text. Per-tenant detail lives in the ledger and rollups, queried on demand.

Traces carry redacted correlation ids; payload capture off by default. Dashboards cover budget
denials, failover safety, unknown outcomes, cache state, error rate, latency, reconciliation lag and
outbox lag.

**Liveness** proves process health. **Readiness** requires Postgres, the budget-control path, and
**at least one policy-eligible provider or healthy circuit** — never every provider, which would
turn an intentional failover into an outage. Cache loss does not affect readiness; budget-controller
loss does.

### 10.1 Console and live feed (Phase 9, built)

- **Explorer.** A read-only core port `LedgerSearch`, separate from the append-only `LedgerStore`, implemented by
  `JdbcLedgerSearch`: tenant in the `WHERE` clause, time range at most 31 days, page at most 200, keyset cursor
  `(created_at, id) < (:at, :id)` encoded as `<epoch-micros>_<uuid>`, newest first. Bounds are checked in the core
  query type, so the API cannot forget them.
- **Feed.** `LiveFeed` is a `LedgerCommitListener`: `ObservedLedgerStore` (the `@Primary` decorator over the ledger
  store; Phase 10 generalised Phase 9's `FeedingLedgerStore` into it so metrics use the same hook) calls every listener
  from an after-commit synchronisation, so a hint never announces a row that rolled back. Publishing goes through one
  bounded queue that discards on overflow or when Redis is down; serving never waits for it. Channel
  `gw:feed:<tenant>`; message: id, status, task class, model key, time.
- **Subscriptions.** `GET /api/v1/feed` (SSE, consumer or operator) subscribes to the caller's tenant only. At most 5
  per tenant per replica (`429 FEED_LIMIT`, `Retry-After: 5`); at the cap, connections are probed twice 50 ms apart
  and dead ones dropped; 15 s heartbeat; 10 min maximum life. A gone client is logged at DEBUG.
- **Reconcile.** The console keeps a watermark, the last confirmed ledger time, and reads the ledger from one second
  before it after every connect, reconnect and hint burst. A hint is unconfirmed until the ledger returns its row.
- **CORS.** Only `gateway.console.allowed-origins` (empty by default), bearer header, no credentials.

### 10.2 Metrics, traces, logs and alerts (Phase 10, built)

- **Metrics from the ledger.** `ObservedLedgerStore` calls each `LedgerCommitListener` after the ledger transaction
  commits; `InvocationMetrics` is one. Every terminal path writes one row, so every invocation is counted once and a
  rolled-back call is not counted. `gateway_invocations_total`, `gateway_invocation_duration_seconds` and
  `gateway_provider_duration_seconds` (histograms, fixed buckets 25 ms to 60 s) carry `provider, model, status,
  task_class`; tokens carry `provider, model, task_class`; `gateway_cache_hits_total{tier}`,
  `gateway_cache_degraded_total{task_class}`, `gateway_api_errors_total{code}`. The model label passes through
  `BoundedLabel` (at most 20 values, the rest `other`). **Cost is not a metric**: a meter is a double. Counters agree
  with the ledger (`MetricLedgerAgreementIT`); the series count does not grow with tenants (`MetricCardinalityIT`,
  `MetricCardinalityTest`, release-blocking).
- **Budget control as metrics.** `BudgetMetrics`: reconcile runs by `outcome` (ok, partial, failed), repairs by `kind`,
  seconds since the last clean run, reservations held by `state`, age of the oldest unknown outcome. A failed read
  keeps the last values rather than reporting zero.
- **Management port.** Prometheus and the probes are served on `MANAGEMENT_SERVER_PORT` (9464 gateway, 9465
  aggregator), behind the same security chain: `/actuator/env` is refused on both ports (`PrometheusEndpointIT`,
  `AggregatorPrometheusIT`).
- **Traces.** Spring Observation with the OpenTelemetry bridge, OTLP over HTTP when an endpoint is set. Span tree:
  server span → `gateway.invocation` → `gateway.cache.lookup` → `gateway.budget.reserve` → one `chat <model>` CLIENT
  span per attempt (`gen_ai.operation.name`, `gen_ai.provider.name`, `gen_ai.request.model`, `gen_ai.usage.*`,
  `gen_ai.response.finish_reasons`, `error.type`) → `gateway.ledger.write`. Gateway spans may carry only the keys in
  `InvocationTracing.ALLOWED_ATTRIBUTES`; exception events carry a type name, never a message. The server span's URL
  is the route template. Retry then failover shows as four attempt spans in order; an unknown outcome as exactly one
  (`InvocationTraceIT`). Markers sent through every path never appear in any span (`TracePrivacyIT`). The collector's
  `redaction` processor is a second allow-list.
- **Logs.** ECS JSON on the console (`GATEWAY_LOG_FORMAT`, default `ecs`; empty for plain text); `traceId` and
  `spanId` from the MDC on every line; an exception stays on one line in `error.*` (`LogFormatTest`).
- **Alerts.** `deploy/prometheus/rules/gateway.rules.yml`: twelve rules, each with `owner`, `severity` and a runbook
  link (`docs/RUNBOOK.md`); promtool tests in CI. Rare-event rules read recording rules that count a series born
  inside the window at its whole value, because the gateway creates counter series on first use and Prometheus 3 does
  not extrapolate `increase()` to zero. Alertmanager routes on `owner` to team mailboxes; locally the SMTP server is
  Mailpit, which captures and delivers nothing.
- **Fault demonstration.** `demo/run_faults.sh` forces each scenario with real faults (stopped Redis, broker, gateway)
  plus two injected records, asserts the alert fires and its mail arrives, and restores everything.

---

## 11. Data Operations

- Flyway migrations in `gateway-adapters`, versioned per phase, never edited after CI application.
- Repositories extend Spring Data's marker `Repository`, never `JpaRepository`/`CrudRepository`. Tenant in every read signature.
- Ledger and audit tables append-only; no update or delete path exists in the repository layer.
- **Partition maintenance is a real job**: future partitions created ahead of time, archive and delete retention jobs, index policy, query-time bounds enforced in the operator API, and a **tested restore**.
- **Outbox retention** deletes only after a durable publish marker plus a replay window. Poison-event policy, schema compatibility rules, consumer idempotency key, DLQ replay approval and named alert ownership are all defined before Kafka ships.
- Money is `NUMERIC(18,8)`. Never floating point.
- Redis: separate namespaces for cache and budget projection; Lua for projection updates; TTL, safe expiry and operational alarms.

---

## 12. Testing Strategy

| Level | Evidence |
|---|---|
| Unit | Budget state transitions, availability arithmetic, cache identity, routing capability filters, failure classification, masking policy, pricing calculation, template rendering |
| Architecture | ArchUnit — core isolation (no Spring/JPA/Redis/SDK), no `JpaRepository`, no `double` in cost paths, only the ledger component writes ledger rows |
| Integration | Testcontainers Postgres + Redis, Flyway, JWT tenant isolation, row-lock reservation under concurrency, projection failure, idempotency races, outbox retry and dedupe |
| Provider contract | Stubbed HTTP fixtures per adapter: request shape, normalized usage, error mapping, streaming lifecycle |
| Evaluation | Deterministic property checks in CI; pinned settings; explicit nondeterminism policy |
| Console | Vitest for the API client, session, SSE parser and reconcile state (CI job `console`); API-level `ConsoleReconnectIT`, `InvocationExplorerIT`, `CorsIT`; Playwright golden path against a running gateway (local) |
| Operational | Load and fault tests before any scalability claim; verify trace/log redaction and metric cardinality limits |
| Telemetry (Phase 10) | `MetricCardinalityTest`, `MetricCardinalityIT`, `MetricLedgerAgreementIT`, `PrometheusEndpointIT`, `BudgetMetricsTest`, `InvocationTraceIT`, `TracePrivacyIT`, `LogFormatTest`, `AggregatorPrometheusIT`; `promtool test rules` (CI job `telemetry`); `demo/run_faults.sh` against the running stack (local) |

**Release-blocking:** cross-tenant isolation · ledger completeness on every status · idempotency
prevents double execution · `UNKNOWN_OUTCOME` never retried · budget fails closed and rejects ·
reservation settles or reconciles, never leaks · masking before cache · cache-key completeness ·
promotion gate refuses a failing candidate · no payload in logs, traces or events · metric
cardinality bounded.

---

## 13. Local Topology

**Default profile is small**: one Spring Boot process, Postgres with Flyway, Redis, deterministic
stub provider. Optional profiles add one real provider · OTel collector with Prometheus and Grafana ·
Kafka with the aggregator · the console (built in Phase 9; `GATEWAY_CONSOLE_ORIGINS` opens CORS for its origin).

Built in Phase 10 as `deploy/compose.yaml` (project `ai-gateway-demo`), started by `demo/up.sh [profiles]` and
stopped by `demo/down.sh`. The gateway and aggregator run the built jars on `eclipse-temurin:21-jre`.

| Profile | Adds |
|---|---|
| (default) | Postgres (pgvector image), budget Redis, cache Redis, gateway on 8080 (management 9464) |
| `telemetry` | OTel collector (redaction, file export to `docs/evidence/phase10/collector/`), Jaeger 16686, Prometheus 9090, Alertmanager 9093, Mailpit 8025, Grafana 3000 (dashboard *AI Gateway*; Prometheus and a read-only Postgres datasource) |
| `events` | Kafka (KRaft) and the aggregator (management 9465); the outbox relay is switched on; a one-shot grants Grafana's role read access to `usage_agg.usage_rollup` |
| `console` | nginx serving the built console on 5173 |
| `scaleout` | `gateway-2`, `gateway-3` and an nginx balancer on 8088 that names the answering replica in `X-Upstream` |

Three replicas is a **demonstration profile only**, sharing
one machine, disk and kernel — it shows statelessness and fan-out and proves nothing about
availability.

**Machine constraints (4-core / 24 GB):** cap WSL2 in `.wslconfig`; singleton Testcontainers base
class is mandatory; never run this stack and the sibling project's stack together; compact the WSL2
virtual disk periodically.

---

## 14. ADR Index

Full text in `SYSTEM_DESIGN.md`. ADR-1 domain ignorance · ADR-2 module structure · ADR-3 budget
fails closed, cache fails open · ADR-4 every path writes a ledger row · ADR-5 semantic cache opt-in
and prohibited for decisions · ADR-6 capabilities not models · ADR-7 gated pointer promotion ·
ADR-8 events carry no text · ADR-9 outbox before Kafka · ADR-10 two providers before three ·
ADR-11 Postgres authoritative · ADR-12 post-send timeouts never retried · ADR-13 Spring AI advisors
are mechanism, not boundary · ADR-14 readiness separate from liveness · ADR-15 leaks documented ·
**ADR-16 no cross-store atomic transaction** · **ADR-17 core owns ports and policy, adapters live
outside**. **ADR-18 operational agents are external consumers (proposed)** · **ADR-19 ledger metadata export (proposed)** · **ADR-20 data-sensitivity routing refuses rather than
downgrades (proposed)** · **ADR-21 shared rate limiter falls back to local (built)**.

---

## 15. Proposed Extensions (PROPOSED when written; each item says whether it is built)

All items follow the phasing rules: none starts before its phase, each builds green first.

**15.1 Live-model run (Phase 3.8).** Opt-in test tag, excluded from default CI so the build stays
green without keys (3.7). Fixed question set, hand-scored sample, figures read from the ledger.

**15.2 Alert pipeline (Phase 10.8).** Prometheus rules (10.5) route through Alertmanager to email.
Local profile: mail-capture container. Real SMTP only after mocked verification.

**15.3 Provider comparison (Phase 11.9).** `gateway-eval` drives one question set through both
adapters. Outputs: deterministic property pass rate, measured latency, measured cost. Contract
tests for Bedrock use recorded Converse responses; real AWS calls come last. Leaks found go to
ADR-15.

**15.4 Infrastructure as code (Phases 11.10–11.11).** Terraform describes the Fargate, ALB, RDS,
ElastiCache and Secrets Manager resources. CI authenticates through GitHub OIDC; no AWS keys are
stored. Offline `terraform validate` and plan in CI; `apply` and `destroy` run by the owner with
explicit approval. This is a reference deployment.

**15.5 Ledger metadata export (Phase 11.12, ADR-19).** A read-only job copies checksums and bounded
metadata (no prompt or response text) to object storage under tenant-prefixed keys, with
server-side encryption and a purge path. It issues no `UPDATE` or `DELETE` on the ledger. Local:
MinIO. Cross-tenant negative test is release-blocking.

**15.6 ops-agent contract (Phase 12, separate repository, ADR-18).** Inputs: Alertmanager webhook.
Tools: read-only `queryPrometheus`, `searchLogs`, `readRunbook`, `checkHealth`. Guardrails: step,
token and time limits; every tool call audited without payload text; tool results treated as
untrusted model input. The tool-calling loop talks to its model provider directly because the
gateway does not carry tool calls; non-tool steps such as the email summary may route through the
gateway and appear in its ledger.

**15.7 Credential order.** Mocks first; real credentials last and only with explicit go-ahead:
OpenAI key, then SMTP, then AWS. See `IMPLEMENTATION_PLAN.md`.

### Phase 13 — interview-impact hardening (PROPOSED)

Order: 13.1 → 13.6, after Phases 10 and 11 are committed (`IMPLEMENTATION_PLAN.md`). No
paid API and no cloud resource. None of these items changes the §2 consistency model, the §4 order
of budget eligibility, cache and reservation, or the post-send retry rule (ADR-12).

**15.8 Stream timing (13.1, FR-19).**

```
provider send ──► first content chunk ──► terminal (complete | disconnect | overrun | deadline | failure)
   t0 = nanoTime()     t1 (once)                 t2 (once, the stream's single terminal)
first_token = t1 − t0       duration = t2 − t0
```

- *Where.* The stream lifecycle in `gateway-api` (the component that already elects the single
  terminal) holds `t0`, `t1` and `t2`. `gateway-core` gains a value type `StreamTiming(Long firstTokenNanos,
  long durationNanos)` with no framework type; the API layer turns it into meters.
- *Clock.* `System.nanoTime()` only; wall-clock time is never subtracted. `t1` is taken when the bounded
  reader receives the first non-empty content delta from the provider, not when the client is written to,
  so the figure is provider plus gateway time and does not depend on the client's network.
- *Metrics.* `gateway.stream.first_token` and `gateway.stream.duration`, Micrometer timers registered
  through `InvocationMetrics` with the existing fixed SLO buckets. Tags: `model` (catalogue id) and
  `outcome` (`InvocationStatus` name), both bounded enumerations. Never tenant, request id or prompt id
  (`MetricCardinalityTest`). Recorded once, after the ledger row commits, by the same code that publishes
  the existing invocation metrics. A stream with no first token records `duration` only.
- *Ledger.* Nullable `first_token_ms INTEGER` on `invocation_ledger` through a Flyway migration on the
  partitioned parent so existing and future partitions carry it. Written in the same insert as the row;
  no update path is added. Non-streamed rows leave it null.
- *Trace.* A span event `first_token` on the existing provider span, no attributes (`TracePrivacyIT`).
- *Dashboard.* A Grafana panel: `histogram_quantile` p50/p95 of `first_token` by model, over the
  bucket counters.
- *Tests.* `StreamTimingIT`: stub delays the first chunk by a configured amount → one first-token sample
  ≥ delay and `first_token_ms` ≥ delay on the row; client disconnects before any token → no first-token
  sample, one duration sample, one `PARTIAL_STREAM` or provider-failure row as today.

**15.9 Data-sensitivity routing (13.2, FR-20, ADR-20). Built (2026-11-13).** What was built differs from
the proposal below in four places, listed under *As built* at the end of this section.

```
screen (text as sent) → mask → classify sensitivity → resolve prompt → budget eligibility
   → cache (key includes level; RESTRICTED skips semantic) → route (filter: model.max ≥ level)
   → reserve → execute → failover candidates filtered by the same rule → settle → ledger (level only)
```

- *Core types (`gateway-core`, no framework).*
  - `enum DataSensitivity { PUBLIC, INTERNAL, RESTRICTED }`, ordered.
  - `SensitivityClassifier`: a pure function `(DetectionSummary, SensitivityPolicy) → DataSensitivity`.
    `DetectionSummary` is what Phase 5 already produces: detector type and match count, no values.
    Default mapping: card number (Luhn), IBAN, national id → `RESTRICTED`; email → `INTERNAL`;
    nothing detected → `PUBLIC`.
  - `SensitivityPolicy` per tenant, optionally per prompt: a minimum level. Result =
    `max(detector level, policy minimum)`. Policy can raise the level, never lower it.
- *Order.* Classification runs on the detection result immediately after masking, so it sees what the
  detectors saw before the text was changed, and it runs before the cache so the level is part of the
  cache key. It reads no text itself.
- *Catalogue.* Each model gains `max_sensitivity` (Flyway migration; existing rows default to `INTERNAL`,
  so nothing becomes eligible for `RESTRICTED` by accident). The stub gets a second profile,
  `stub-private`, declaring `RESTRICTED`, to stand for a model inside the network boundary (a later
  local Ollama or vLLM adapter, or Bedrock in a private VPC endpoint in Phase 11).
- *Routing.* `ModelRouter` applies `model.maxSensitivity.compareTo(level) >= 0` as an **eligibility
  filter**, alongside capability and context limits, **before** any ranking. The fallback candidate list is
  built from the same filtered set, so failover cannot leave it. The rule lives in core; the advisor and
  adapter layers do not see it (ADR-13).
- *Cache.* Exact cache key gains the level. Semantic cache: the level is a filter column inside the
  vector query, next to the tenant filter and before `ORDER BY`; `RESTRICTED` does not consult or write the
  semantic cache at all.
- *Refusal.* No eligible model → new `InvocationStatus.SENSITIVITY_BLOCKED`: typed error code, HTTP 422,
  one ledger row, no reservation taken (the refusal is decided at routing, before §2 step 7). This needs a
  row in `FLOWS.md` §10 and a case in `LedgerCompletenessTest`. If every cleared model's breaker is open, the
  result is the same refusal, never a fallback to an uncleared model.
- *Evidence.* Ledger column `data_sensitivity` (enum name). Metric tag `sensitivity` on the existing
  invocation counter is allowed (three values). Outbox event field `dataSensitivity` declared as
  `ENUM_NAME` in `EventPayloadPolicy` and in `docs/contracts/`. Never a matched value.
- *Tests.* `SensitivityClassifierTest` (table-driven, core); `SensitivityRoutingIT`;
  **`RestrictedNeverReachesPublicModelIT` (release-blocking)**: private model's breaker forced open →
  zero sends on the public stub, one `SENSITIVITY_BLOCKED` row; `SensitivityCacheKeyIT`: an entry stored
  for `PUBLIC` is never served to a `RESTRICTED` request; ArchUnit unchanged.
- *Limit.* The level is only as good as the pattern detectors. Free-form confidential text, names and
  addresses are classed `PUBLIC` unless tenant policy raises the minimum.
- *As built.*
  1. *Order.* The pipeline's existing order is kept: screen → mask → **classify** → route (clearance filter)
     → budget eligibility → exact cache → semantic cache → reserve. Routing already ran before the cache, so
     the level is known at both. Classification takes the masking counts (`Map<DetectorType,Integer>`);
     defaults: card, IBAN, US SSN, PAN, Aadhaar → `RESTRICTED`; email, phone → `INTERNAL`; a detector with
     no mapping → `RESTRICTED` (fail towards refusal).
  2. *Semantic cache.* No new column. The level is part of the context digest
     (`CacheIdentity.semanticContext`) that the vector query already filters on inside the SQL, before
     `ORDER BY`; the effect is the same and the schema is unchanged. `RESTRICTED` neither probes nor writes.
  3. *Refusal after a failed send attempt.* A non-stream call whose cleared model fails `NOT_SENT` (for
     example its breaker is open) and whose only remaining candidates are uncleared is also
     `SENSITIVITY_BLOCKED` (reservation released, `NOT_EXECUTED`), not `NOT_SENT`, so the reason is
     visible. A failure after a send keeps its own status (`UNKNOWN_OUTCOME` is never relabelled).
  4. *Policy.* Per-tenant minimum only (`gateway.sensitivity.tenants`); per-prompt minimum not built.
     `stub-private` is `fallback_eligible = false`, so `PUBLIC` and `INTERNAL` calls never fail over onto it.
     The cache-key test is the unit test `SensitivityCacheKeyTest`, not an IT.

**15.10 Load test (13.3, FR-21). Built (2026-11-13).** As built: a standalone `perf/` Maven project (no parent, not in the reactor, because the Boot BOM would pin Netty and Jackson against Gatling's), `-Pperf`; looping keep-alive users ramped to 12 per scenario rather than an open-model arrival rate, because a user per request exhausted Docker Desktop's loopback proxy (LOAD_TEST.md, finding 1); reconciliation is over the whole run window per scenario, not a steady-state sub-window. Gatling Java DSL under a Maven `perf` profile, not a module and not a
deployable. Target: the compose stack with the stub provider at a fixed configured latency. Scenarios:
exact cache hit, stub miss, budget block, sensitivity block, streaming. Open-model ramp to a stated rate,
then a steady phase. Outputs: Gatling HTML report plus `docs/evidence/LOAD_TEST.md` with p50/p95/p99, error
rate, hardware, JVM, image digests and every setting. Validity check: accepted requests in the steady window
equal ledger rows for the same window. Two runs, `spring.threads.virtual.enabled` true and false, same load.

**15.11 Kubernetes reference manifests (13.4, FR-22). Built (2026-11-13), verified on kind.** As built: egress to DNS, Postgres, Redis, Kafka when on, and provider CIDRs (NetworkPolicy matches addresses, not names; the identity provider is a JWK URI the gateway fetches, so its CIDR belongs in `provider.egressCidrs` too); `preStop` uses the kubelet sleep action because the image has no shell; the HPA is validated but not exercised on kind (no metrics-server). Helm chart `deploy/helm/ai-gateway`: Deployment
(non-root, read-only root filesystem, `/tmp` as `emptyDir`, requests and limits), liveness and readiness
mapped to the existing separate probes (ADR-14), `preStop` and `terminationGracePeriodSeconds` longer than
the stream deadline so open streams reach their terminal, HPA on CPU, PodDisruptionBudget `minAvailable: 1`,
NetworkPolicy allowing egress only to Postgres, Redis, Kafka, the identity provider and the provider
allow-list, secrets only by `secretKeyRef`. Datastores are external values. CI: `helm lint` and
`kubeconform`. Local verification on `kind`.

**15.12 Shared rate limiter (13.5, FR-23, ADR-21, optional).** `RateLimiterPort` in core; the current
`TenantRateLimiter` becomes the default local adapter. A Redis adapter runs a Lua token bucket (one script,
one round trip) on a tenant-namespaced key. Unavailable store → fall back to the local bucket and count the
fallback (the ADR-21 proposal; rate limiting protects capacity, it is not money, so it does not fail closed
like the budget). **As built:** `RedisTokenBucketLimiterIT` runs three limiter instances (own connection and local bucket each) on one Redis; the Lua script uses the Redis clock; the cache Redis carries it; the counter is `gateway.ratelimit.fallback`. Selected by `gateway.rate-limit.mode: SHARED`.

**15.13 README restructure (13.6).** Ten-line summary, then "design decisions to ask me about", each linking
to its ADR and its test. Done last so it can quote the measured figures from 15.10.

**15.14 Model-graded scoring, advisory (14.1, FR-24, ADR-22). Built against the stub; calibrated once live (guide §14.1).** Core package
`eval.judge`: `Rubric` (id, version, weighted criteria, integer 1–5; `ANSWER_QUALITY_V1` = relevance 2,
faithfulness 2, format 1), `JudgePrompt` (fixed system text from the rubric; input and answer fenced as data in the
user message), `JudgeReplyParser` (exactly one `SCORES k=n ...` line naming every criterion once, in range; anything
else is `JUDGE_UNPARSEABLE`, `JUDGE_MISSING_CRITERION` or `JUDGE_SCORE_OUT_OF_RANGE`), `JudgeScoring` (per case
`(score−1)/4` weighted, run = mean of scored cases truncated to 2 places; unscored counted, never zero; none scored
= null), `JudgePort`, `JudgeStore`, `JudgeRun`. API: `EvaluateRequest.judge`; `EvalService.run(..., judge)` refuses
with `JUDGE_DISABLED` (422) unless `gateway.eval.judge.enabled`, scores and stores the deterministic run first, then
grades each case's first answer through `InvocationJudge` (an ordinary `InvocationService` call, task class
CLASSIFICATION, temperature 0, 64 output tokens, key `judge-<run>-<case>`; its ledger row's `actual_cost_usd` is
read back and summed). A failed judge call is `JUDGE_CALL_FAILED`, never an exception. Answers live in memory for the
length of the run only. Storage: `V12__eval_judge.sql`, table `eval_judge_score` (tenant, eval run FK, rubric id and
version, counts, nullable score with a CHECK that it is null exactly when nothing was scored, cost, JSONB of case
keys + integer scores or reason codes), insert-only by the V6 trigger function. Read: `GET
/api/v1/prompts/{id}/eval-runs/{runId}/judge` (same roles as evaluation; another tenant's record is 404). Audit:
`EVAL_JUDGE_RECORDED`. The stub honours `[[stub:reply=TEXT]]`, which scripts the judge in tests. Calibration
(gateway-eval): `JudgeCalibration` against `judge-calibration/recon-answers/v1.json` (8 hand-scored synthetic
answers), `GatewayChatJudge` for a live run through a gateway's chat API; `scripts/judge_calibration.sh` is a dry run
unless `CONFIRM_SPEND=yes`.

**15.15 Embedding harness and fine-tune plan (14.2, FR-25, ADR-23). Harness only.** gateway-eval
`RetrievalHarness.measure(set, EmbeddingProvider, configuredThreshold, sweep)`: recall@1, recall@3, MRR; per
threshold, correct hits (paraphrase vs own canonical) and wrong answers served (distractor vs the canonical it
imitates); chosen threshold = lowest with zero wrong answers, flagged when none. Set:
`retrieval/recon-questions/v1.json` (10 groups × canonical, 3 paraphrases, 2 distractors), pinned in
`HARNESS_CHECKSUMS.sha256`. Infra: `infra/terraform/sagemaker_finetune.tf` (count-gated on
`enable_embedding_finetune`, default false: private encrypted bucket with 30-day expiry, role assumable by SageMaker
with access to that bucket, its log group and image pull only), `infra/sagemaker/training-job.json` and `train.py`
(sentence-transformers, MultipleNegativesRankingLoss on query/positive pairs), `scripts/sagemaker_finetune.sh` (dry
run unless `CONFIRM_SPEND=yes`; refuses without a corpus) and `scripts/embedding_eval.sh`. Adopting a fine-tuned
model is a separate change (vector width, see ADR-23).

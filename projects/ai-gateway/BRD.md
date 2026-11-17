> **Copy of a private repository document**, snapshot taken 2026-11-17. The source code is private, so links to files in the repository are shown as plain paths. Status labels are the author's own, as written at the time of the snapshot. All data is synthetic.

# Business Requirements Document
## AI Gateway — Enterprise LLM Governance Control Plane

| Field | Value |
|---|---|
| Document | BRD — version 3 (canonical) |
| Supersedes | All earlier BRD drafts and `AI_GATEWAY_PROJECT_BRIEF.md` §3 |
| Companion docs | `TDD.md`, `ARCHITECTURE.md`, `IMPLEMENTATION_PLAN.md`, `SYSTEM_DESIGN.md` |
| Status | Design baseline, written before implementation. Built so far (see `PHASE_TRACKER.md`): Phases 0–13 committed (Phase 11 offline; Phase 12 in the `ops-agent` repository); Phase 14 committed (`8bf83df` and `2670535`, both CI green; `2670535` passed only on re-run, see the flaky outbox test in `PHASE_TRACKER.md`) (2026-11-14) |
| Data classification | Synthetic data only. No client names, hostnames or internal identifiers anywhere in this repository. |

**What this is.** A Java-native AI control plane with deliberate boundaries, measurable governance,
and documented limits. It is not a domain RAG platform, not an agent runtime, and not a data-loss-
prevention product. The engineering claim it makes is narrow and defensible: deterministic policy
execution, bounded failure behaviour, auditable cost control, and provider portability.

---

## 1. Problem Statement

An enterprise adopting generative AI across more than one application reaches the same four failures
in the same order:

1. **Credential sprawl.** Every application holds provider keys. Rotation is a cross-team exercise; revocation is a guess.
2. **Invisible cost.** Spend is known per provider invoice, not per tenant, team, feature or prompt version.
3. **Inconsistent controls.** Each team implements masking and screening differently. The organisation's posture is the weakest of them.
4. **Unmanaged prompt change.** Prompts live in source, ship as deploys, cannot be rolled back independently, and reach production with no evidence that output quality held.

In regulated settings each is an audit finding. "Who sent what to which model, under which prompt
version, at what cost, and what was masked before it left the perimeter?" must be answerable from
one system of record.

## 2. Business Goals

| Goal | Measurable outcome | Explicit non-goal |
|---|---|---|
| Governed model access | Every invocation carries tenant, policy, prompt version, model, outcome and usage evidence | Replacing the consuming application's business policy |
| Cost control | No concurrent request bypasses configured token or monetary budgets | Provider invoice reconciliation without a billing export |
| Provider resilience | Routing selects an eligible model and fails over **only** where execution is known safe to retry | A guarantee of exactly one provider request when a timeout lands after the request was sent (the outcome is recorded as unknown, never retried) |
| Prompt delivery discipline | Only evaluated, immutable prompt content is eligible for promotion | A general-purpose LLM quality platform |
| Portfolio credibility | Every phase has tests, a runnable demo, and explainable failure modes | Running a production topology on one laptop |

## 3. Scope

| Gateway owns | Consumer owns | Excluded from first release |
|---|---|---|
| Provider adapters, capability-based routing, tenant enforcement, budgets, caches, prompt registry, ledger, outbox, operational telemetry | Retrieval, documents, business workflow, authorisation beyond gateway RBAC, domain state, human approval | Tool execution, arbitrary URL fetch, agent planning inside the gateway (an external consumer may run agents, ADR-18), full DLP, invoice reconciliation, multi-region DR |

**The boundary rule.** The gateway knows cost, content shape and routing. It never knows business
meaning. Any requirement that would make it understand what a request *means* is rejected on
principle — that is the difference between a platform and a distributed monolith.

## 4. Stakeholders

| Stakeholder | Interest |
|---|---|
| Application teams | Ship AI features without owning credentials or guardrails |
| Platform / FinOps | Attribute and cap spend per tenant |
| Security & Compliance | Prove what left the perimeter and what was masked first |
| Prompt owners | Change prompts safely without a code deploy |
| SRE | Operate, observe and degrade it predictably |

## 5. Functional Requirements
> FR-12 to FR-16 and FR-19 to FR-25 were **PROPOSED**: planned additions placed in Phases 3, 10, 11, 12, 13 and 14 of `IMPLEMENTATION_PLAN.md`. Each row says whether it is built. FR-12 was run with the owner's go-ahead (`LIVE_RUN_REPORT.md`).
> **Built status:** FR-01 to FR-11 are built (Phases 1–10; see `PHASE_TRACKER.md`). FR-10's metrics, traces, dashboards and alert rules are built in Phase 10. FR-13 is built in Phase 10 against a local mail-capture container; delivery through a real SMTP server is not done. FR-17 and FR-18 are built in Phase 9.

| ID | Requirement | Acceptance criterion |
|---|---|---|
| FR-01 | Authenticated, tenant-scoped completion | Tenant derives only from validated JWT claims. A caller cannot select another tenant through any header, body field, path variable or query parameter. |
| FR-02 | Capability-based routing | Route only to models satisfying capability, residency, policy, context-limit and availability constraints. Callers declare capabilities, never a provider or model. |
| FR-03 | Safe retry and failover | Retry or fail over **only** for `NOT_SENT` or explicitly classified `RETRYABLE_FAILURE`. **Never** for `UNKNOWN_OUTCOME` or `PARTIAL_STREAM`. Record the classification on every failure. |
| FR-04 | Budget reservation and settlement | Authoritative reservation in the relational store; settle with provider-reported usage; release proven non-execution; expire and reconcile stranded reservations. Available = limit − settled − active reservations. |
| FR-05 | Ledger and outbox | Persist a minimised, immutable invocation record and its outbox event in the same database transaction. |
| FR-06 | Prompt lifecycle | Versions immutable by checksum. Promotion locks the active pointer and requires a passing evaluation run referencing that exact checksum. |
| FR-07 | Cache controls | Exact cache tenant-isolated and policy-versioned. Semantic cache opt-in, off by default, prohibited for regulated or action-taking task classes. |
| FR-08 | Security screening | Pattern masking and injection risk classification are **signals and policies**, not claims of prevention. |
| FR-09 | Streaming lifecycle | Capture start, completion, cancellation, partial stream and provider failure. Settle usage when reported; label the usage `ESTIMATED` honestly when not. Built in Phase 7. |
| FR-10 | Operational interface | Health, readiness, metrics and protected operator queries, with no prompt or response payload leakage. |
| FR-11 | Idempotency | Every mutating request carries a client key. Concurrent same-key requests never execute twice. Chat is synchronous: the first request gets `200`; a same-key request while it runs, or after an unknown outcome, gets `409`; a completed key replays with status `IDEMPOTENT_REPLAY`. There is no `202` and no polling. |
| FR-12 | **PROPOSED** Live-model evaluation run | A fixed question set runs against the real provider; a hand-scored sample is recorded; every cost and latency figure comes from this repository's ledger. |
| FR-13 | Alert delivery (proposed, built in Phase 10 with mail capture) | Alert rules deliver email through Alertmanager; each forced scenario produces one message naming the rule and its owner. |
| FR-14 | **PROPOSED** Cross-provider comparison | The same evaluation set runs through OpenAI and Bedrock; results are deterministic property pass rate, measured latency and measured cost. No quality claim. |
| FR-15 | **PROPOSED** Ledger metadata export | Optional export of ledger checksums and bounded metadata to object storage; tenant-prefixed keys; no prompt or response text; the ledger itself stays append-only. |
| FR-17 | Operator console | A browser console that is a client of the protected API only: the same bearer token, RBAC and audit rows as any caller; no console-only endpoint. The token is held in the tab's memory. A live feed may hint at new calls; the ledger is the record, read after every reconnect. Built in Phase 9. |
| FR-18 | Bounded ledger explorer | Operators list their own tenant's invocations by time and status. The server enforces a range of at most 31 days, at most 200 rows per page and a keyset cursor; the tenant comes from the token and a tenant parameter is refused. Built in Phase 9. |
| FR-16 | **PROPOSED** Incident triage by an external consumer | A separate service (`ops-agent`) investigates alerts with read-only tools and emails findings. The gateway adds no agent code; it exposes metrics, health, readiness and runs as a normal consumer. |
| FR-19 | **PROPOSED** Stream responsiveness evidence (Phase 13.1) | For every streamed call the gateway measures time to first token and total stream duration, exposes them as metrics labelled only by model and outcome, and keeps time to first token on the ledger row. An operator can see p50/p95 time to first token per model on the dashboard. No figure is published until measured (FR-21). |
| FR-20 | Data-sensitivity routing (Phase 13.2, ADR-20) | Each request is given a sensitivity level (`PUBLIC`, `INTERNAL`, `RESTRICTED`) from the structured identifiers detected in it and the tenant's policy. Each model in the catalogue declares the highest level it may receive. A request is sent only to a model cleared for its level, **including on failover and from the cache**. When no cleared model is available the request is refused with `SENSITIVITY_BLOCKED` and recorded in the ledger; it is never downgraded to a less trusted model. Only the level is recorded, never the detected values. Built in Phase 13.2; the level is only as good as the pattern detectors. |
| FR-21 | Measured performance evidence (Phase 13.3, built) | A repeatable load test on the stub provider reports p50/p95/p99 latency and error rate for cache hit, provider miss, budget block, sensitivity block and streaming, on stated hardware, with the request count reconciled against ledger rows. It compares virtual threads on and off. It produces no cost figure. |
| FR-22 | Kubernetes reference manifests (Phase 13.4, built, verified on kind) | A Helm chart deploys the gateway with its separate liveness and readiness probes, resource limits, autoscaling, a disruption budget, and egress limited to its dependencies and the provider allow-list. Verified on a local `kind` cluster. It is a reference deployment, not an availability claim. |
| FR-23 | Shared rate limit (built) (Phase 13.5, ADR-21, optional) | A deployment may choose a rate limit shared across replicas so that a tenant's rate holds regardless of replica count. The per-replica limiter remains the default. Behaviour when the shared store is unavailable is stated in ADR-21 and tested. |
| FR-24 | Advisory model grading (built against the stub, calibrated once live) (Phase 14.1, ADR-22, optional) | A prompt owner may ask for an evaluation run's answers to be graded by a model against a versioned rubric. The grade is shown beside the run and labelled advisory; it never changes whether promotion is allowed. Each grading call is governed and recorded like any other call, with its cost. Off unless the operator enables it. A live calibration against hand scores is required before the grade is relied on; one was run on 2026-11-13 (8 items, exact agreement 0.50, within-one 0.75 per criterion). |
| FR-25 | Embedding measurement before fine-tuning (harness only) (Phase 14.2, ADR-23) | The owner can measure an embedding model on a fixed question set, including how many wrong answers the semantic cache would serve at each threshold, and compare two models. A fine-tune on SageMaker is planned (infrastructure off by default) and runs only with a real corpus and an approved spend. |

## 6. Non-Functional Requirements

| ID | Area | Requirement |
|---|---|---|
| NFR-01 | Security | JWT validated for issuer, audience (the `aud` claim must contain the one configured value), expiry and clock skew; RS256 only; key from a JWK set URI (keys refreshed by the decoder) or a fixed public key; services and users share one audience (no service-to-service audience separation); RBAC on every admin endpoint with authorization tests per role |
| NFR-02 | Isolation | Tenant mandatory in every persistence query, cache key, budget key, event payload and operator filter. Cross-tenant access returns `404`, never `403` |
| NFR-03 | Reliability | Bounded provider timeouts, circuit breaker per provider-model, bulkheads with explicit queue limits and timeout budgets, connection-pool ceilings, defined overload response |
| NFR-04 | Governance availability | Governance state is correctness-critical. When it cannot be enforced, calls are rejected with `GOVERNANCE_UNAVAILABLE`. Non-governance infrastructure (cache, aggregation, console) may degrade freely |
| NFR-05 | Consistency | No cross-store atomic transaction is claimed. Postgres is authoritative; Redis is a repaired projection. Dual-write is **recoverable**, not atomic (ADR-16) |
| NFR-06 | Privacy | No raw prompt or response stored by default. Ledger holds checksums, bounded category metadata and redacted diagnostics. Caches are the single documented exception (§7) |
| NFR-07 | Auditability | Every invocation reconstructable from the ledger without provider logs |
| NFR-08 | Cost arithmetic | `BigDecimal` throughout. Pricing version and region retained per row |
| NFR-09 | Observability | Structured logs, traces, metrics and audit events correlate by request id, with **bounded metric label cardinality** — never tenant id, request id, prompt id or prompt text as labels |
| NFR-10 | Performance | Local target is predictable bounded concurrency, not throughput benchmarking. Any production claim requires load-test evidence and dependency sizing. FR-21 (proposed) supplies the first measured figures, for the stated hardware only |
| NFR-11 | Supply chain | Dependency pinning, SBOM generation, image scanning, documented vulnerability triage policy, signed images and provenance if a cloud deployment is claimed |
| NFR-12 | Quality gate | Every phase builds cleanly, passes focused tests, runs a repeatable demo, exercises one failure mode, and is explainable in three minutes without notes |

## 7. Data Handling — the cache exception, stated plainly

The no-raw-payload rule has exactly one exception, and it is named rather than hidden: **a cache
stores content, because that is what a cache is.**

Both cache tiers therefore carry mandatory controls: tenant-namespaced keys, short TTL, encryption
at rest, exclusion from backups by default, per-tenant purge capability, and **no status as an audit
source**. The ledger stores checksums and is evidence. The cache stores content and is never
evidence. Semantic cache additionally stores minimised, encrypted entries and is off by default.

Local payload capture for debugging requires synthetic data, an explicit feature flag, encrypted
storage, short retention, access audit, and a **startup failure** if enabled outside an approved
environment. "Local only" is not a control.

## 8. Policy Decisions and Deliberate Trade-offs

| Decision | Rationale | Guardrail |
|---|---|---|
| Cache fails open | An optimisation; callers continue at higher latency and cost | Emit cache-degraded telemetry; never an audit source |
| Budget fails closed | A control that degrades to permissive is not a control | Explicit `GOVERNANCE_UNAVAILABLE` response; HA Redis in production; durable records reconciled |
| Postgres authoritative, Redis projected | Redis is not durable by configuration everywhere; a control whose state can vanish is not auditable | Reconciler repairs the projection; provider call blocked until projection succeeds |
| Reserve immediately before the provider call | Avoids reserve/release churn and races on free cache hits | Budget **eligibility** still checked before cache; only the reservation moves later |
| Outbox before Kafka | A durable transaction boundary matters before asynchronous scale-out | Claim only at-least-once delivery with consumer deduplication by event id |
| Pattern masking, not DLP | Useful, with known limits | Test false positives and negatives; state the limit in the README |
| Semantic cache opt-in | Similarity does not establish response interchangeability | Approved task classes only, tenant isolation inside the query, short TTL, recorded similarity metadata |
| Refuse rather than downgrade (FR-20, built) | Sending restricted data to a less trusted model to keep a call alive is a disclosure, not a recovery | Availability for `RESTRICTED` traffic depends on the cleared models only; the refusal is visible in the ledger and on the dashboard |
| Modular monolith first | Real seams without premature distributed operations | No separate deployable until operational need exists |

## 9. Risk Register

| Risk | Impact | Mitigation |
|---|---|---|
| Estimated vs actual usage diverges | False exhaustion or overspend | Reservation state machine, settlement, expiry, reconciliation, pricing provenance |
| Provider timeout after request accepted | Duplicate cost or divergent response | Idempotency key, failure taxonomy, `UNKNOWN_OUTCOME`, no blind failover |
| Redis projection lost | Incorrect permit or deny | Postgres authoritative; reconciler rebuilds; fail closed meanwhile |
| Provider feature mismatch | Leaky abstraction, invalid route | Capability catalogue plus per-adapter contract tests |
| Ledger growth | Slow operator queries, retention cost | Time+tenant partitioning, future-partition creation, archive/delete jobs, query-time bounds, tested restore |
| Telemetry leaks data | Privacy exposure | No body capture by default, redaction, sampling, cardinality controls |
| Evaluation flakiness | Unsafe promotion, unreliable CI | Deterministic stubbed suites, pinned settings, explicit nondeterminism policy |
| Prompt template abuse | Injection via template or system-message capability | Variable validation, no arbitrary expressions, deterministic rendering, checksum over the fully resolved prompt |
| Live feed mistaken for the record | An operator acts on a call that was never recorded, or misses one | The feed carries no replay and no `Last-Event-ID`; hints are shown unconfirmed until the ledger returns the row; the console reads the ledger after every reconnect |
| Restricted data reaches a public model (FR-20, built) | Disclosure outside the approved boundary | Eligibility filter applied before ranking and to every failover candidate; sensitivity part of the cache key; `RESTRICTED` bypasses the semantic cache; release-blocking negative test |
| Sensitivity under-classified (FR-20, built) | Confidential free text treated as `PUBLIC` | Stated limit: classification is only as good as the pattern detectors; tenant policy can raise the minimum level for a tenant (per-prompt minimum not built), never lower it |
| Latency figures quoted out of context (FR-21, proposed) | Overclaim in review or interview | Report states hardware, JVM, image digests and settings; stub provider only; counts reconciled with the ledger |
| Console token exposure | A stolen token acts as the operator | Token held in tab memory only (no storage, no cookie); CORS limited to configured origins with no credentials; short-lived tokens from the identity provider in a real deployment |

## 10. Release Acceptance

An independent reviewer can, from the repository alone:

1. Build it green and run the full test suite.
2. Start it with one command and issue a governed call.
3. Exhaust a tenant budget and see the denial in both the response and the ledger.
4. Kill the budget controller and watch the service reject rather than serve ungoverned traffic.
5. Force a post-send timeout and confirm one charge, not two.
6. Promote a prompt version, watch the gate refuse a regression, and roll back without a deploy.
7. Read a complete trace and a dashboard of cost per tenant per model. Built in Phase 10: Jaeger shows the call's span tree, and the Grafana dashboard reads cost per tenant per model from the rollups as text (money is never a metric). The console's usage screen (Phase 9) shows the same rollups.
8. Find an honest statement separating implemented controls, local demonstrations and production prerequisites.

## 11. Claim Discipline

No cost or latency figure is published until measured in this repository, from its own ledger rows,
on stated hardware. Three local replicas demonstrate statelessness, not availability. Evaluation is
deterministic property checking, not quality judgement. Masking is pattern detection, not DLP. The
cloud deployment is a reference deployment, not an HA claim. Savings measured in other portfolio
projects stay attributed to those projects.

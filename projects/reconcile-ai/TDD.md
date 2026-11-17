> **Copy of a private repository document**, snapshot taken 2026-11-17. The source code is private, so links to files in the repository are shown as plain paths. Status labels are the author's own, as written at the time of the snapshot. All data is synthetic.

# Technical Design Document (TDD)
## Reconciliation Break Resolution Copilot (Repo A — `reconcile-ai`)

| Field | Value |
|---|---|
| Document | TDD |
| Companion doc | `BRD.md` |
| Sibling project | `automatch-decision` (Repo B) — integrates via Kafka; see §7. **[Built: no Repo B exists in this workspace; the Kafka intake in §7 stands in for the upstream feed.]** |
| Status | Design baseline (original). Built and in `main` through Tier 2 and production hardening; the as-built record is `STATUS.md` and `design/IMPLEMENTATION_AND_DESIGN_RATIONALE.md`. Section 5.4, §6.1 and ADR-S1 describe the investigation modes: single-call and agent are built (Flyway V10); the supervised modes are **built, tested and measured live** (Flyway V11); the decision rule was not met, so they stay off by default. Where this TDD still says 4 modules, WebFlux, or "not built" for Kubernetes/HPA, the code has superseded it (ADR-7; K8s/HPA built). |

---

## 1. Architecture Overview

```
                          ┌─────────────────────────────┐
                          │   Angular UI (reconcile-ai-ui)│
                          │  break queue · investigate ·  │
                          │  approval queue (RxJS + SSE)   │
                          └───────────────┬───────────────┘
                                          │ REST + SSE (JWT Bearer)
                                          ▼
        ┌──────────────────────────────────────────────────────────┐
        │              reconcile-copilot  (Spring Boot app)          │
        │                                                            │
        │  ┌────────────────────────────────────────────────────┐  │
        │  │  7-stage Advisor Chain (Spring AI ChatClient)       │  │
        │  │  Tenant ctx → Token budget → Injection/PII guard →  │  │
        │  │  L1/L2 cache → RAG context → Compression → Audit    │  │
        │  └───────────────────────┬──────────────────────────────┘  │
        │                          │                                  │
        │  ┌───────────────────────▼──────────────────────────────┐  │
        │  │          Confidence + Materiality Policy Gate         │  │
        │  │     (pure Java — the model never decides, only       │  │
        │  │      proposes; see §5)                                 │  │
        │  └───────┬─────────────────────────────┬──────────────────┘  │
        │          │ clears bar                  │ doesn't clear       │
        │          ▼                             ▼                     │
        │  ┌───────────────┐          ┌───────────────────────┐      │
        │  │  Auto-resolve │          │  Two-tier HITL queue   │      │
        │  │  + outbox      │          │  1st-level / 2nd-level │      │
        │  └───────┬────────┘          └────────────┬────────────┘      │
        └──────────┼────────────────────────────────┼──────────────────┘
                   │                                │
                   ▼                                ▼
        ┌───────────────────────┐        ┌───────────────────────┐
        │  PostgreSQL + pgvector │        │  Redis (L2 cache)      │
        │  ledger, audit, RAG    │        │  Caffeine (L1, in-JVM) │
        └───────────────────────┘        └───────────────────────┘
                   ▲
                   │ Kafka consumer (automatch.proposals)
                   │ same policy gate as above
        ┌──────────┴──────────────┐
        │  automatch-decision      │  (Repo B — see §7)
        │  (sibling project)        │
        └───────────────────────────┘
```

**[Built, differences from the diagram: cache is L2 Redis only (no Caffeine L1); the Kafka topics are `reconcile.break.intake` (in) and `reconcile.break.events` (out, from the outbox), see §7; an AUTO_RESOLVE route is stored as L1 approval, see §6.]**

**Module JARs consumed by `reconcile-copilot`:**
- `reconcile-domain` — entities, Flyway migrations, no Spring AI dependency
- `evidence-intelligence` — Tika parsing, hybrid RRF retrieval, citation-only responses
- `exception-ops` — Kafka consumer for resolved/escalated events, root-cause grouping, digest generation

---

## 2. Module Structure

```
reconcile-ai/
├── pom.xml                          (parent — see pom.xml deliverable)
├── reconcile-domain/                 JPA entities, Flyway, JWT/tenant context model
├── evidence-intelligence/            Tika parsing, hybrid RRF, citation verification
├── exception-ops/                    Kafka consumer, grouping, digest generation
├── reconcile-copilot/                Main Spring Boot app: REST, SSE, advisor chain,
│                                      policy gate, HITL, outbox
├── reconcile-ai-ui/                  Angular app (standalone components, RxJS SSE wrapper)
├── docker/
│   └── docker-compose.yml            Postgres+pgvector, Redis, Kafka (KRaft), app, UI,
│                                      Nginx (load-balancing demo), Prometheus, Grafana
├── docs/
│   ├── BRD.md
│   ├── TDD.md                        (this file)
│   ├── SYSTEM_DESIGN.md               (ADRs, failure modes — §9, §10)
│   └── TOKEN_ECONOMICS.md
├── .github/workflows/ci.yml
└── README.md
```

---

## 3. Domain Model

### 3.1 Core Entities

| Entity | Key fields | Notes |
|---|---|---|
| `BreakItem` | `id`, `tenantId`, `breakReference`, `status`, `ledgerAmount`, `externalAmount`, `variance`, `currency`, `counterpartyRef`, `source`, `createdAt`, `updatedAt` | `source` = `MANUAL` \| `AUTOMATCH_KAFKA` |
| `Evidence` | `id`, `breakItemId`, `tenantId`, `fileName`, `fileType`, `storagePath`, `uploadedAt` | File format whitelist enforced at upload |
| `EvidenceChunk` | `id`, `evidenceId`, `tenantId`, `chunkText`, `embedding (vector)`, `chunkIndex` | One row per parsed/chunked evidence segment |
| `HistoricalResolution` | `id`, `tenantId`, `breakItemId`, `category`, `resolutionText`, `embedding`, `resolvedAt` | Populated from previously-closed breaks; feeds the RAG retrieval pool |
| `ResolutionProposal` | `id`, `breakItemId`, `tenantId`, `category`, `confidenceScore`, `rationale`, `citedEvidenceChunkIds[]`, `source`, `createdAt` | Structured LLM output — never free text |
| `ApprovalTask` | `id`, `breakItemId`, `tenantId`, `tier`, `status`, `assignedRole`, `decidedBy`, `decidedAt`, `notes` | `tier` = `FIRST_LEVEL` \| `SECOND_LEVEL` |
| `AuditEvent` | `id`, `tenantId`, `breakItemId`, `eventType`, `actor`, `payloadHash`, `previousHash`, `timestamp`, `tokenCost` | Append-only, hash-chained — no update/delete path exists in the repository layer |

### 3.2 Enumerations

```
BreakStatus:        OPEN, INVESTIGATING, PENDING_FIRST_LEVEL, PENDING_SECOND_LEVEL,
                     AUTO_RESOLVED, RESOLVED, REJECTED

ResolutionCategory:  TIMING_DIFFERENCE, DUPLICATE_POSTING, FEE_DEDUCTION,
                     FX_VARIANCE, SYSTEM_ERROR, MISSING_COUNTERPARTY_RECORD,
                     WRITE_OFF, OTHER

InvestigationMode:   SINGLE_CALL, AGENT                (built; column investigation_job.mode, V10)
                     SUPERVISED, SUPERVISED_PLANNED    (built; V11 widens the column to 24 characters)
ReviewerVerdict:     AGREE, CHALLENGE                   (built; record; the verdict is stored on investigation_job.review_verdict)

ApprovalTier:        FIRST_LEVEL, SECOND_LEVEL
ApprovalStatus:      PENDING, APPROVED, REJECTED
```

---

## 4. API Design

| Method | Path | Purpose | Auth role |
|---|---|---|---|
| `POST` | `/api/v1/breaks` | Create a break record | `RECON_ANALYST` |
| `GET` | `/api/v1/breaks?status=&page=` | List/filter breaks (tenant-scoped) | `RECON_ANALYST`, `APPROVER_L1`, `APPROVER_L2` |
| `GET` | `/api/v1/breaks/{id}` | Break detail | same |
| `POST` | `/api/v1/breaks/{id}/evidence` | Upload evidence (multipart) | `RECON_ANALYST` |
| `POST` | `/api/v1/breaks/{id}/investigate` | Trigger investigation, synchronous, returns final `ResolutionProposal` | `RECON_ANALYST` |
| `GET` | `/api/v1/breaks/{id}/investigate/stream` | Same investigation, SSE lifecycle events | `RECON_ANALYST` |
| `GET` | `/api/v1/approvals/pending?tier=` | List pending approvals for the caller's tier | `APPROVER_L1`, `APPROVER_L2` |
| `POST` | `/api/v1/approvals/{id}/approve` | Approve a queued proposal | matching tier role |
| `POST` | `/api/v1/approvals/{id}/reject` | Reject a queued proposal | matching tier role |
| `GET` | `/api/v1/breaks/{id}/audit` | Full audit trail for a break | `APPROVER_L2`, `SRE_ADMIN` |
| — | Kafka consumer: `automatch.proposals` | Async intake from Repo B, no REST surface | service-to-service |

SSE event types: `EVIDENCE_LOADED`, `RETRIEVAL_COMPLETE`, `PROPOSAL_READY`, `ROUTED` (carries the routing decision), `ERROR`.

**[Built, the API differs from this table: investigation is asynchronous (`POST /api/v1/breaks/{id}/investigations` returns 202 and a job; poll `GET /api/v1/investigations/{jobId}` or stream `GET .../events`, SSE event name `progress`); approvals are `GET /api/v1/approvals` and one `POST /api/v1/approvals/{taskId}/decision` with APPROVE or REJECT; audit is `GET /api/v1/audit/aggregates/{id}`. The Kafka consumer reads `reconcile.break.intake` and creates a break only. The full as-built list is `reconcile-ai/PLATFORM_FACTS.md` §1.]**

---

## 5. AI Architecture

### 5.1 Advisor Chain (Spring AI `ChatClient`, ordered)

| Order | Advisor | Responsibility |
|---|---|---|
| 1 | `TenantSecurityContextAdvisor` | Binds `tenant_id`/`userId` from JWT into the request context |
| 2 | `TokenGovernanceAdvisor` | JTokkit BPE budget check — rejects oversized payloads at 0ms, before any network call |
| 3 | `PromptSecurityAdvisor` | Injection-pattern detection + PII masking/redaction — blocks or sanitizes before the model ever sees the input |
| 4 | `SemanticCacheAdvisor` | L1 Caffeine (exact hash) → L2 Redis (cosine similarity) — short-circuits on hit | **[Built: L2 Redis only. L1 Caffeine was not built, see design rationale C34.]**
| 5 | `RagContextAdvisor` | Hybrid RRF retrieval (§5.2) — injects evidence + historical-resolution context |
| 6 | `TokenCompressionAdvisor` | Deterministic Java compression — strips boilerplate, caps chunks beyond top-k, de-duplicates near-identical text |
| 7 | `TracingAuditAdvisor` | OTel span tagging + `ai_audit_ledger` write (tokens, cost, model, decision) |

### 5.2 Hybrid RRF Retrieval

```
Dense:  pgvector HNSW, cosine similarity, over EvidenceChunk + HistoricalResolution embeddings
Sparse: PostgreSQL GIN/tsvector full-text search over the same chunk text

RRF fusion:  score(d) = Σ 1 / (60 + rank_m(d))   for m in {dense, sparse}
```

Retrieval query is **always** filtered by `tenant_id` as part of the SQL `WHERE` clause — not applied after retrieval. This is a deliberate architecture choice: similarity math never computes over rows a tenant isn't authorized to see (see ADR-2, §9).

### 5.3 Structured Output Contract

The LLM's only permitted output shape for the decision path is the `ResolutionProposal` record: `category`, `confidenceScore`, `rationale`, `citedEvidenceChunkIds[]`. Enforced via Spring AI's structured-output binding, not by parsing free text.

**Citation integrity rule:** any `citedEvidenceChunkIds` value not present in what the retrieval step actually returned causes the proposal to be rejected and re-requested (bounded retry) — the model cannot cite evidence that doesn't exist.

### 5.4 Investigation modes (all built; supervised measured: rule not met, off by default)

| Mode | Control flow | Status |
|---|---|---|
| `SINGLE_CALL` | Prepared prompt, one model call, advisor chain | Built, default |
| `AGENT` | One model chooses among four read-only tools (`ReconTools`) under turn, tool-call and time limits | Built, off by default (`reconcile.agent.enabled`) |
| `SUPERVISED` | Specialist agents share one `AgentRunContext` and budget (see below) | Built (V11), off by default (`reconcile.agent.supervised.enabled`) |

All modes end in the same shared tail: normalise draft, citation check against the retrieval snapshot, policy gate, persist proposal, maker-checker approval. A mode only changes how the draft is produced.

**Supervised flow (built)**

```
job row (tenant, break, mode=SUPERVISED)  ->  SupervisedInvestigation (Java)
   1. EvidenceAnalyst  tools: getBreakSummary, listEvidence, searchEvidence  -> findings with chunk ids
   2. RiskChecker      tool : findSimilarResolvedBreaks + facts as text      -> advisory flags
   3. Draft            analyst model produces the ProposalDraft JSON
   4. Reviewer         no tools; input = draft + exact cited chunk text      -> AGREE | CHALLENGE (verbatim quote must match the chunk, else CHALLENGE)
   -> shared tail (citations, policy gate incl. rule 0 below, persist)
```

Variants evaluated: **S1** fixed Java order (expected production choice) and **S2** model-planned supervisor (LangChain4j agents-as-tools). Specialists share `AgentRunContext` (tenant, break, allowed chunk ids, run-wide budget) and every model call goes through `AccountingChatModel` (retry, breaker, `llm_call`, token metrics). Tool results keep the existing guard (injection scan, PII mask, length cap). New components: `SupervisedInvestigation`, three prompt templates, `ReviewerVerdict`, config `reconcile.agent.supervised.*`. Persistence: a nullable `agent_name` column on `agent_tool_call` in the next Flyway migration (V11); `AGENT_RUN_COMPLETED` audit event extended with a per-agent trace digest.

---

## 6. Policy Gate Design

Pure Java, no model involvement. Decision table (defaults — configurable, stated as design targets, not fixed):

| Confidence | Materiality | Routing |
|---|---|---|
| ≥ 0.95 | Below low threshold | `AUTO_RESOLVED` |
| ≥ 0.80 | Below high threshold | `PENDING_FIRST_LEVEL` |
| < 0.80 | any | `PENDING_SECOND_LEVEL` |
| any | Above high threshold | `PENDING_SECOND_LEVEL` (materiality overrides confidence) |

**[Built: `PolicyGateService` applies first-match rules: citations invalid, root cause UNKNOWN or confidence < 0.80 go to MANUAL_REVIEW (not second level); non-USD or variance ≥ 25000 go to L2; otherwise L1. The auto-resolve rule exists but is off by default (ADR-9), and even when enabled `ProposalService.persistProposal` stores the route as L1 approval, so no proposal is committed without a human.]**

### 6.1 Reviewer rule (built)

One rule is added in front of the existing rules, evaluated only in supervised mode:

| Condition | Routing | Reason code |
|---|---|---|
| Reviewer verdict `CHALLENGE` (a cited claim not supported by its chunk) | `MANUAL_REVIEW` | `REVIEWER_CHALLENGE` |
| Reviewer verdict `AGREE` | no change; existing rules decide | n/a |

The rule can only escalate. `PolicyInput` gains a `reviewerChallenged` flag (default false), so single-call and agent modes behave exactly as before. Risk-checker flags are displayed to the approver and written to the audit event; they never change the route (ADR-1 is preserved).

This function is called from **two** trigger points — the synchronous `/investigate` REST flow and the async Kafka listener from Repo B — so the decision logic never forks between sync and async paths (see §7). **[Built: one trigger only, the investigation job runner. The Kafka listener creates breaks and never calls the gate.]**

---

## 7. Kafka Integration (Cross-Repo Spine)

```
Topics:
  automatch.proposals            B → A, key: tenantId:breakId, at-least-once
  recon.resolution.committed     A → exception-ops, published from the outbox
  automatch.proposals.DLT        poison-pill quarantine after N failed attempts
```

- Repo B's internal intake stays file-shaped (poll/extract/zip); only its *output* to A is a Kafka event.
- A's Kafka listener invokes the **same policy-gate service** as the REST-triggered investigation flow (§6) — one decision engine, two triggers.
- On approval or auto-resolve, A writes ledger + audit-hash + outbox row in a single transaction; a relay publishes `recon.resolution.committed`.
- `exception-ops` consumes `recon.resolution.committed`, groups by root cause (Java-computed counts/amounts), and the LLM writes only the prose summary per group.

**[Built, topics in code (`infrastructure/messaging/`):**
- **In: `reconcile.break.intake`, group `reconcile-ai-intake`. Message `{tenantId, breakReference, ledgerAmount, externalAmount, currency}`, amounts as strings. It creates a break with source `KAFKA`; it does not carry a proposal. `BreakSource.MATCH_PROPOSAL` exists but nothing sets it.**
- **Out: `reconcile.break.events`, published by `OutboxRelay`; event types BREAK_CREATED, PROPOSAL_ROUTED, BREAK_RESOLVED, DAILY_DIGEST_READY; headers `event-id`, `event-type`, `tenant-id`; at least once.**
- **Dead letter: `reconcile.break.intake.dlt` after 3 retries at 500 ms.**
- **The digest is built in the same app (`DailyDigestService`), not in a separate `exception-ops` consumer. Kafka is off by default (`KAFKA_ENABLED`).]**

---

## 8. Cross-Cutting Concerns — Token Optimization, Guardrails, Caching, Security

**These four are implemented independently in both Repo A and Repo B — not shared as a library.** See ADR-6 (§9) for why.

| Concern | This repo's implementation |
|---|---|
| **Token optimization** | JTokkit pre-call budget check (0ms) · deterministic Java compression (replacing LLMLingua-2/DJL — see ADR-3) · every model call logged to `ai_audit_ledger` with tokens-in/out and estimated cost, written in the same transaction as the decision |
| **Guardrails** | Injection-pattern + PII-masking advisor runs before any model invocation (advisor order #3) · blocked attempts logged to a **separate** security-audit table from the normal decision audit, so security review doesn't require filtering the main ledger |
| **Caching** | L1 Caffeine (in-JVM, exact hash, <1ms) → L2 Redis (semantic, cosine threshold) — both tiers active in this repo, since evidence-investigation queries are semantically diverse enough to benefit from L2 | **[Built: L2 Redis only; no Caffeine L1.]**
| **Security** | JWT validation via Spring Security OAuth2 Resource Server · RBAC roles `RECON_ANALYST`, `APPROVER_L1`, `APPROVER_L2`, `SRE_ADMIN` · tenant filter enforced at the SQL/vector-query level (§5.2), not only at the controller layer |

---

## 9. Load Balancing / Scalability

**Design principle:** horizontally scalable by construction, not retrofitted.
- No in-memory session or workflow state — HITL queue and audit state live in Postgres; cache state lives in Redis (shared, not in-heap-only for L2)
- Idempotency keys on every mutating request, so retries across replicas are safe **[Built: no `Idempotency-Key` header. Break create is unique on (tenant, breakReference), evidence is deduplicated by SHA-256, and a second investigation on the same break returns 409 while one is running.]**
- Actuator liveness/readiness probes wired so an unhealthy replica is pulled from load-balancer rotation

**[Built 2026-11-15: Java 21 virtual threads.** `spring.threads.virtual.enabled=true`; Tomcat requests, `@Scheduled` and the default task executor run on virtual threads because the work is I/O-bound (Postgres, model provider, EKA, gateway). The investigation executor stays a bounded platform pool so its full queue still yields the `503` back-pressure. `TenantContext` (SecurityContextHolder) and `TenantScope` are plain thread-locals, never inheritable; isolation under virtual threads is re-verified by `VirtualThreadsTenantIsolationIT` and `TenantScopeVirtualThreadTest`. Not measured: throughput. ADR: `docs/adr/ADR-virtual-threads.md`.]**

**Local demo artifact:** Docker Compose profile running 2–3 replicas of `reconcile-copilot` behind Nginx doing round-robin with health-check-aware routing.

**Built since:** Kubernetes manifests, HPA (CPU, 2 to 6), PDB and a kind overlay; CI renders both with kustomize and schema-checks them with kubeconform (no cluster in CI). **Still not built:** custom-metric scaling. Original text: Kubernetes Deployment + Service + HorizontalPodAutoscaler, scaling on CPU/memory or a custom metric such as HITL queue depth.

---

## 10. Observability

**Metrics (Micrometer → Prometheus):** `recon.breaks.processed`, `agent.cache.hit_rate{tier}`, `llm.calls`, `llm.tokens.used`, `llm.estimated_cost_usd`, `guardrail.blocks`, `hitl.queue_depth{tier}`, `hitl.decision_latency`

**[Built: names in code all start with `reconcile.`, for example `reconcile.llm.tokens`, `reconcile.llm.cost.usd`, `reconcile.llm.latency`, `reconcile.llm.errors`, `reconcile.llm.cache`, `reconcile.investigation.failed`, `reconcile.agent.tool.calls`, `reconcile.supervised.verdict`, `reconcile.pii.masked`, `reconcile.outbox.published`. No HITL queue-depth or decision-latency metric exists.]**

**Tracing (OTel spans):** ingestion → retrieval → LLM call (with `llm.prompt_tokens`/`llm.completion_tokens`/`llm.estimated_cost_usd` attributes) → policy gate → HITL/write-back

**Grafana panels:** cache hit rate vs. cost, guardrail blocks over time, HITL queue depth by tier, p95 investigation latency, token cost per resolved break

---

## 11. Failure Modes & Resilience

| Dependency down | Detected by | System response |
|---|---|---|
| LLM provider | Resilience4j circuit breaker trips | Falls back to "queued for manual review," request never fails outright **[Built: the job ends FAILED and the break moves to FAILED_RETRYABLE in one transaction, with an audit event; an analyst can start a new investigation. The POST already returned 202, so no request fails, but nothing is queued for manual review.]** |
| PostgreSQL | Actuator readiness probe fails | Replica pulled from load-balancer rotation |
| Kafka broker | Producer send timeout | Retried with backoff; after N failures, routed to DLT |
| Redis | Cache client timeout | Falls through to cache-miss path — correctness preserved, latency degrades gracefully |

---

## 12. Architecture Decision Records (Condensed)

**ADR-1 — Java enforces the confidence+materiality gate, never the LLM.**
The model proposes; deterministic code decides. Prevents any single model response, however confident-sounding, from directly causing a ledger write.

**ADR-2 — Hybrid RRF retrieval over pure vector similarity.**
Reconciliation evidence mixes free-text narrative (favors dense/semantic) with exact reference codes and amounts (favors sparse/lexical). Fusing both, rather than choosing one, avoids the vocabulary-mismatch failure mode either method has alone.

**ADR-3 — Deterministic compression instead of a local ONNX compression model.**
A quantized LLMLingua-2-style model gives marginally better compression but introduces real load-reliability risk in a demo context. Deterministic Java compression (boilerplate stripping, top-k capping, de-duplication) gets most of the cost benefit with zero model-loading risk.

**ADR-4 — B's internal intake stays file-shaped; only its output to A is event-driven.**
Preserves fidelity to the real-world integration pattern this project is modeled on, while modernizing the one seam (the handoff) that benefits most from decoupling.

**ADR-5 — Single writer (Repo A) for all ledger mutations.**
Repo B never writes the ledger under any circumstance, including high-confidence matches — every path converges on the same policy gate.

**ADR-6 — Token optimization, guardrails, caching, and security are duplicated across repos, not shared as a library.**
A shared dependency creates version-coupling and a single point of failure for security logic across two otherwise-independent services. Reimplementing the same *principles* independently, deliberately, is a stronger and more defensible design than trusting a shared package to enforce them — and it means each repo's security posture can be reasoned about, audited, and evolved in isolation.

**ADR-S1 (built; measured 2026-11-13: rule not met) — Supervised review is a measured option, not a default.** Result on gpt-4o-mini, 40 cases: no wrong cause reached an approver in any mode on the hard set; supervised cost 7.9x and planned 12.3x single call (limit 5x). Decision: single call stays default. See `docs/proof/phase4g-supervised-eval.md`.
Earlier evaluation showed a single tool-using agent cost about 8x a single call for similar accuracy, so agents are not added by default. The one failure the existing checks cannot catch is a citation that exists in the retrieval snapshot but does not support the claim. A reviewer targets exactly that. Reviewer output reaches routing only through a deterministic gate rule that escalates (ADR-1). Two orchestration variants are compared (fixed Java order vs model-planned); the decision rule (at least half the wrong-cause answers that reach an approver removed on the hard set, cost no more than 5x single call) is fixed before the run. Alternatives rejected: letting an agent choose the route (breaks ADR-1); a larger single prompt (does not give an independent check); a different framework (LangChain4j already on the classpath). Detail: `reconcile-ai/docs/adr/ADR-supervisor-agents.md`.

---

## 13. Testing Strategy

- **Unit tests:** policy gate decision table (property-based over confidence/materiality ranges), evidence chunker, citation-integrity check
- **Integration tests:** Testcontainers-backed Postgres+pgvector, Redis, Kafka — full ingest → retrieve → propose → gate → audit flow
- **Retrieval evaluation:** a held-out synthetic set of ~20–30 Q&A pairs, Recall@5 and MRR computed and stored as a CI artifact
- **Security fixtures:** a labeled set of known prompt-injection strings, asserting all are blocked before reaching the model
- **Supervised mode (built), all offline with scripted models:** disabled by default and refused when disabled; shared budget across agents (runaway fault marker); tenant isolation across all agents; `CHALLENGE` gives `MANUAL_REVIEW`/`REVIEWER_CHALLENGE`, `AGREE` leaves the route unchanged, a reviewer can never upgrade a route; injection text in evidence cannot flip a verdict; audit event with per-agent trace digest and a still-valid hash chain
- **Evaluation (built; live run done 2026-11-13, rule not met):** existing 28 cases plus about 12 labelled hard cases (conflicting evidence 4, misleading analyst note 3, two plausible causes 3, claim not supported by cited chunk 2); four modes compared with the existing `EvalHarness`; live run only with explicit approval of spend
- **Golden-path demo tests:** three scripted scenarios (auto-resolve, first-level, second-level) runnable via `demo/run_golden_paths.sh` **[Built: `scripts/demo-golden-path.ps1`; there is no `demo/run_golden_paths.sh`.]**

---

## 14. Local Development Bootstrap — Implementation Plan

### 14.1 Prerequisites

```
Java 21 (Temurin recommended)
Maven 3.9+
Docker + Docker Compose v2
Node.js 20+ / npm (for reconcile-ai-ui)
An OpenAI API key, or Ollama installed locally (ollama pull llama3.1)
```

### 14.2 Bootstrap Sequence

```bash
# 1. Clone and inspect
git clone <repo-url> reconcile-ai && cd reconcile-ai

# 2. Environment
cp .env.example .env
# set OPENAI_API_KEY or leave blank to use Ollama profile

# 3. Start infrastructure only (no app yet)
docker compose -f docker/docker-compose.yml up -d postgres redis kafka

# 4. Build the domain module first — proves Flyway + entities compile and migrate
mvn -pl reconcile-domain -am clean install

# 5. Run the domain module's own tests (Testcontainers-backed)
mvn -pl reconcile-domain test

# 6. Build everything
mvn clean install -DskipTests

# 7. Run the app
mvn -pl reconcile-copilot spring-boot:run

# 8. Smoke test
curl -X POST http://localhost:8080/api/v1/breaks \
  -H "Authorization: Bearer <local-dev-jwt>" \
  -H "Content-Type: application/json" \
  -d '{"breakReference":"BRK-0001","ledgerAmount":50500.00,"externalAmount":50000.00,"currency":"USD"}'

# 9. (optional) Run the UI
cd reconcile-ai-ui && npm install && npm start
```

### 14.3 Build Order (dependency-ordered, not calendar-ordered)

```
A0   reconcile-domain: entities, Flyway, JWT/tenant context. No AI yet.
A1   Investigate REST + SSE skeleton (no evidence, no files yet).
A2   evidence-intelligence: Tika parsing, hybrid RRF. "Unknown citation → reject" rule.
A3   Full advisor chain + policy gate + two-tier HITL + outbox.
     >>> Golden-path demo achievable here. <<<
A4   Kafka consumer wired to the same policy-gate service as A3.
A5   exception-ops: consumes resolution-committed, groups, digests.
UI   Angular investigation + approval screens against the working API.
S0   (done) Gate rule 0 + ReviewerVerdict + scripted agents + offline tests.
S1   (done) Fixed-order supervised pipeline end to end under mock-llm.
S2   (done) Hard-case labelling + eval wiring.
S3   (done) Model-planned supervisor variant.
S4   (done 2026-11-13) Live evaluation and written comparison: decision rule not met; shared budget raised from 8/14/120 to 12/24/180 after the first run, both runs kept.
L1   Load-balancing demo: replicas + Nginx + health-check routing.
P1   Polish: token-economics numbers from real demo data, Grafana panels,
     three rehearsed golden-path recordings.
```

Pace this against actual available hours — each phase only depends on the one before it, and the demoable golden path exists as early as the end of A3, before any Kafka or UI work begins.

---

*Companion documents: `BRD.md` (business requirements), `README.md` (repo-facing quickstart), `pom.xml` (build configuration).*

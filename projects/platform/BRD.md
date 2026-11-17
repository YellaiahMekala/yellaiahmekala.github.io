> **Copy of a private repository document**, snapshot taken 2026-11-17. The source code is private, so links to files in the repository are shown as plain paths. Status labels are the author's own, as written at the time of the snapshot. All data is synthetic.

# Integration Business Requirements Document
## reconcile-ai, EKA, ai-gateway, and conditional RERO

| Field | Value |
|---|---|
| Version | 2.1 draft — 2026-11-14 (agreed fixes 1–8 applied inline) |
| Status | Design baseline. Only items explicitly evidenced in `EVIDENCE_REGISTER.md` may be described as built. |
| Data | Synthetic demonstration data only. |
| Delivery baseline | Q0–Q2: reconcile-ai, EKA, and ai-gateway. RERO is conditional Q3 work. |

## 1. Product statement

The platform demonstrates a regulated reconciliation workflow in which Java owns business state and decision authority. `reconcile-ai` retrieves tenant-isolated procedure context from EKA and routes one existing model path through ai-gateway. The model may draft a recommendation; deterministic Java code validates evidence, applies policy, and requires a separate human approver before resolution.

RERO is not part of the baseline product. It is a bounded Python recommendation experiment, enabled only after Q1–Q2 succeed and retained only if its predeclared evaluation rule is met.

## 2. Problems addressed

| ID | Problem | Required outcome |
|---|---|---|
| P1 | Existing services have separate identity arrangements and no downstream service identity. | A shared local realm and tenant-scoped service calls are proven by spikes and negative audience tests. |
| P2 | Investigations cannot use governed procedure knowledge. | `reconcile-ai` can retrieve only the requesting tenant's EKA knowledge as context. |
| P3 | Direct chat-completion calls bypass ai-gateway governance. | One existing `reconcile-ai` model path is routed through ai-gateway with safe error semantics and ledger correlation. |
| P4 | A second orchestration runtime may add complexity without value. | RERO is evaluated against existing modes and cut if it fails its predeclared rule. |

## 3. Objectives

| ID | Objective | Success evidence |
|---|---|---|
| O1 | Prove tenant-safe cross-service retrieval. | Tenant B cannot retrieve Tenant A’s uploaded text through `reconcile-ai`. |
| O2 | Preserve authority boundaries. | Only the existing resolution path can resolve; no internal RERO route changes business state. |
| O3 | Govern a real Java model path. | A `jobId → invocationId` mapping reconciles paths that reach gateway invocation processing. |
| O4 | Handle ambiguous model outcomes safely. | A post-send `504 UNKNOWN_OUTCOME` creates exactly one provider request and no automatic retry/failover. |
| O5 | Make degraded knowledge visible. | Each search records exactly one result: `ok`, `empty`, `unavailable`, or `rejected`. |
| O6 | Keep claims defensible. | Every built claim used in a demo, README, CV, or interview has evidence-register provenance. |

## 4. Scope

### 4.1 Baseline scope: Q0–Q2

- One Keycloak realm for local integration, with audience validation and tenant claims proven by spikes.
- Tenant service identity from `reconcile-ai` to EKA edge and ai-gateway; user tokens are never forwarded.
- `KnowledgeSearchPort` and EKA document/search operations through `reconcile-ai`.
- Query masking, hit screening, knowledge outcome tracking, timeout, circuit breaker, and tenant-isolation tests.
- One existing `reconcile-ai` chat-completion path through ai-gateway, behind a disabled-by-default flag.
- Gateway idempotency/error mapping, budget handling, `UNKNOWN_OUTCOME` handling, and `jobId → invocationId` audit correlation.
- Local Compose profiles, contract tests, and synthetic-data demo evidence.

### 4.2 Conditional scope: Q3

RERO may be built only after the Q2 review gate and after the RERO keep/cut thresholds are committed with date and source commit. If built, it is a same-process FastAPI/LangGraph service that:

- reads one running job only through a short-lived, read-only capability token;
- uses fixed graph nodes and bounded tool/model-call limits;
- cannot write, approve, route, or resolve;
- persists the minimum polling result for a documented TTL; and
- remains disabled by default.

### 4.3 Out of scope

- Automatic resolution in integrated demo configuration.
- A shared database, shared library, or merged build.
- EKA chat, conversation, web fallback, or direct RERO-to-EKA access.
- Model-selected tools, graph cycles, or human interrupt inside RERO v0.
- Kafka/event-driven recommendation solely for this integration.
- A production cloud deployment, availability claim, or real-data demonstration.
- Centralizing embeddings in ai-gateway. The platform claims **governed chat-completion egress** only for migrated paths.

## 5. Actors and authority

| Actor | Allowed action |
|---|---|
| Analyst | Starts eligible investigations and searches own-tenant knowledge. |
| Approver | Approves or rejects a routed proposal; must differ from the maker. |
| SRE administrator | Manages own-tenant knowledge documents through `reconcile-ai`. |
| Gateway operator | Configures tenant budget policy. |
| reconcile-ai | Owns breaks, evidence, proposals, approvals, audit, and resolution. |
| EKA | Owns knowledge ingestion/retrieval; has no reconciliation business authority. |
| ai-gateway | Owns chat-completion governance and invocation ledger; has no business authority. |
| RERO, if retained | Produces a recommendation only; owns no write credential to `reconcile-ai`. |

## 6. Functional requirements

| ID | Requirement | Priority |
|---|---|---|
| FR-01 | All platform services validate a shared issuer, expected audience, expiry, and required claims. | Must |
| FR-02 | A user token is accepted only by `reconcile-ai` and is never used for downstream calls. | Must |
| FR-03 | `reconcile-ai` acquires service credentials by tenant and target. A token with the wrong audience is rejected. | Must |
| FR-04 | Tenant identifiers satisfy the platform pattern and originate only from a validated token. | Must |
| FR-10 | An administrator can upload, list, inspect, and delete own-tenant knowledge through `reconcile-ai`. | Must |
| FR-11 | Analysts can search own-tenant knowledge through `reconcile-ai`; retrieval is context, not break evidence. | Should |
| FR-12 | Knowledge queries are masked before leaving `reconcile-ai`; suspect hits are excluded before use. | Must |
| FR-13 | Every knowledge search reports one of `ok`, `empty`, `unavailable`, or `rejected`. | Must |
| FR-14 | An EKA outage does not become an `empty` result; the single default `KNOWLEDGE_FAIL_CLOSED=false` continues the investigation without knowledge and records `unavailable`. | Must |
| FR-15 | An EKA chunk identifier can never satisfy the break evidence citation verifier. | Must |
| FR-20 | One existing `reconcile-ai` chat-completion path can use ai-gateway when enabled. | Must |
| FR-21 | Gateway calls use the actual synchronous contract: final success is `200`; an in-progress duplicate is `409`; there is no gateway polling API. | Must |
| FR-22 | `504 UNKNOWN_OUTCOME` and `409 IDEMPOTENCY_UNKNOWN_OUTCOME` are never retried or failed over; they end the job, and a later retry is a new job. Other caller retries are at most 1, same key, within the job deadline. | Must |
| FR-23 | Every gateway terminal path has a reconciled record linking `jobId` and `invocationId`. Budget, risk and sensitivity blocks each have exactly one ledger row; only auth/schema rejections have none. | Must |
| FR-24 | Masked sensitive input can progress under policy; detected unmasked card, IBAN, or national-ID input with a real-provider policy is blocked and routed to manual review. | Must |
| FR-25 | Budget exhaustion produces a typed failure and no proposal. | Must |
| FR-30 | **Conditional:** RERO start is idempotent per tenant and canonical business-input fingerprint. | Must if Q3 starts |
| FR-31 | **Conditional:** RERO reads only job-scoped data through read-only tools and server-side job-state invalidation rejects a terminal/cancelled job. | Must if Q3 starts |
| FR-32 | **Conditional:** RERO self-check may add manual review only; confidence is informational. | Must if Q3 starts |
| FR-33 | **Conditional:** reconcile-ai validates RERO citations against server-recorded served evidence before policy gating. | Must if Q3 starts |

## 7. Non-functional requirements

| ID | Requirement |
|---|---|
| NFR-01 | Cross-tenant access returns 404 and every new boundary has a release-blocking negative test. |
| NFR-02 | No break text, evidence text, prompts, model outputs, access tokens, or raw sensitive matches appear in logs, metrics, traces, or run traces. |
| NFR-03 | Outbound calls use explicit connect/read timeouts; safe failures are typed; unknown outcome is never retried. |
| NFR-04 | Feature flags default to disabled; existing suites remain green with integration flags off. |
| NFR-05 | Metrics use bounded labels and no tenant label. W3C trace context is propagated. |
| NFR-06 | CI uses fakes/stubs and synthetic data; a live provider run is opt-in and excluded from CI. |
| NFR-07 | All measurable claims name the configuration, date, commit, command, and evidence artifact. |

## 8. Acceptance criteria

| ID | Criterion |
|---|---|
| AC-01 | Audience/issuer tests accept each intended token and reject tokens intended for another service. |
| AC-02 | Tenant A document upload followed by Tenant B exact-text search returns no result. |
| AC-03 | With EKA unavailable, a supported investigation continues without knowledge and records `unavailable`; it does not report `empty`. |
| AC-04 | Knowledge IDs offered as break citations are deterministically rejected. |
| AC-05 | A migrated gateway path maps real gateway `200`, `409`, budget, risk/sensitivity, and unavailable outcomes. |
| AC-06 | A controlled post-send timeout creates exactly one provider request and yields `UNKNOWN_OUTCOME` without retry. |
| AC-07 | Gateway terminal paths reconcile `jobId → invocationId`; `GatewayLedgerIT` asserts one ledger row for each budget, risk and sensitivity block. |
| AC-08 | Masked and unmasked sensitive-input scenarios are demonstrated with synthetic data and documented policy outcomes. |
| AC-09 | **Conditional:** RERO cannot change reconciliation state, a `CHALLENGE` reaches manual review, and terminal jobs reject its run token. |
| AC-10 | Compose golden path and demo script pass for the enabled phase; every public claim links to the evidence register. |

## 9. Risks and limits

| Risk | Decision / mitigation |
|---|---|
| Per-tenant service clients do not scale. | Limit v1 to a few demo tenants; document token exchange or tenant-aware EKA mapping as future work. |
| RERO duplicates built agent modes. | Keep it conditional; publish the comparison whether it is kept or cut. |
| A RERO token needs two audiences. | If Q3 ships, document it as a short-lived exception, not strict audience separation; use fixed model policy, budget, and egress limits. |
| Pattern masking misses free-form confidential data. | State this limit; use synthetic data; a detected sensitive value blocks real-provider use. |
| The gateway does not govern embeddings. | Use exact wording: governed chat-completion egress for migrated paths. |
| Existing documentation claims drift from code. | Evidence register and Q0 code/CI reconciliation are blocking. |
| Gateway budget/sensitivity not actually usable. | Q0.7 inventory decides: shrink Q2 to Q2.1–Q2.4 on the stub provider, or make the gateway work a prerequisite. |
| "Exactly-once" overclaim. | Banned wording (also banned in ai-gateway). Use "exactly one provider request". |

## 10. Portfolio claim boundary

After Q2, the defensible claim is:

> Implemented a Java system of record with tenant-isolated retrieval and a governed chat-completion gateway; deterministic Java validation and human approval retain decision authority, while unknown model outcomes are handled without unsafe retry.

Do not claim RERO, centralized embeddings, production availability, cost savings, or performance improvements until their evidence rows are complete.

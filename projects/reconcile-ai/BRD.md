> **Copy of a private repository document**, snapshot taken 2026-11-17. The source code is private, so links to files in the repository are shown as plain paths. Status labels are the author's own, as written at the time of the snapshot. All data is synthetic.

# Business Requirements Document (BRD)
## Reconciliation Break Resolution Copilot

| Field | Value |
|---|---|
| Document | BRD |
| Project | Reconciliation Break Resolution Copilot (Repo A — `reconcile-ai`; no Repo B exists in this workspace, the event feed is simulated) |
| Version | 1.1 (adds investigation modes and the supervised review; items marked **Built** are implemented and tested; the supervised modes were also measured on a real model on 2026-11-13 and did not meet the decision rule) |
| Status | Draft — portfolio/demonstration project |
| Data classification | Synthetic data only — no production or client data |

---

## 1. Executive Summary

Financial reconciliation platforms generate a high volume of "breaks" — mismatches between an internal ledger and an external counterparty record (bank statement, broker feed, settlement confirmation). Operations teams routinely attach supporting evidence to these breaks (emails, statements, spreadsheets) as part of manual investigation, but this evidence is rarely read systematically — it sits as an attachment, not an input to a decision.

The **Reconciliation Break Resolution Copilot** is an AI-assisted investigation and resolution system that reads attached evidence, retrieves similar historically-resolved breaks, proposes a structured resolution with a stated confidence level, and routes that proposal through a two-tier human-approval gate before any resolution is committed to the ledger. The system never permits an AI-proposed resolution to write to the ledger without either exceeding a defined confidence-and-materiality threshold (auto-resolve) or clearing a human sign-off step (first- or second-level approval, scaled to materiality).

This is a portfolio/demonstration project built on synthetic data. Domain patterns (evidence attachment, tiered approval, confidence-gated automation) are modeled on patterns common to enterprise reconciliation platforms; no proprietary logic, schema, or data is reproduced.

---

## 2. Business Problem

| Problem | Impact |
|---|---|
| Evidence is attached to breaks but not systematically read | Analysts manually re-read the same evidence types repeatedly; institutional knowledge of "how we resolved this before" is not searchable |
| No confidence-scored automation path | Every break, regardless of triviality, consumes analyst time |
| Approval authority is not risk-scaled in ungoverned automation | A naive automation layer risks writing incorrect resolutions to the ledger with no human check |
| Alerting is high-volume and low-signal | Static "sample of N breaks" notifications don't group related breaks by root cause |
| Audit trails for AI-assisted decisions are often absent or incomplete | Regulatory and internal-audit requirements demand traceable, immutable decision history |

---

## 3. Business Objectives

1. Reduce analyst time spent re-reading evidence for break categories that have clear historical precedent.
2. Provide a confidence-scored, evidence-cited resolution proposal for every open break with attached evidence.
3. Guarantee that no AI-proposed resolution reaches the ledger without either exceeding a defined automation threshold or clearing human approval scaled to materiality.
4. Provide a complete, immutable audit trail for every resolution — automated or human — sufficient for internal and regulatory review.
5. Measure, with the project's own evaluation set, whether adding specialist AI agents improves the quality of proposals enough to justify their cost, and publish the result whichever way it falls. **(Built; measured 2026-11-13: decision rule not met, mode stays off)**
6. Demonstrate a reusable architectural pattern (evidence-grounded reasoning + policy-gated automation) applicable beyond reconciliation to any high-stakes, evidence-based operational decision.

---

## 4. Scope

### 4.1 In Scope

- Break intake via REST API and via an asynchronous event feed (modeling an upstream automated-matching system)
- Evidence upload and parsing (`.pdf`, `.txt`, `.xlsx`, `.msg`-equivalent text formats)
- Evidence-grounded investigation using retrieval-augmented generation (RAG) over both attached evidence and historically-resolved breaks
- Structured resolution proposals (category, confidence score, rationale, cited evidence)
- Confidence- and materiality-based routing: auto-resolve, first-level approval, or second-level approval
- Two-tier human approval workflow with role-based access control
- Immutable, hash-chained audit trail for every decision
- Real-time investigation progress streaming (Server-Sent Events)
- Root-cause-grouped notification digest for resolved/escalated breaks
- Multi-tenant data isolation
- A minimal web UI for the investigation and approval workflows
- Alternative investigation modes: single prompt (default), single tool-using agent (built, off by default), and a supervised multi-agent review (**built and measured; not recommended**, off by default)

### 4.2 Out of Scope

- Any connection to a real banking, ledger, or core-banking system
- Real customer, counterparty, or transaction data of any kind
- Full-scale identity provider integration (a synthetic JWT issuer is used for local development)
- Production deployment infrastructure. **(Built since: Kubernetes manifests with HPA and PDB are rendered and schema-checked in CI, and a kind overlay exists; they have not run on a production cluster)**
- Any schema-mutation or auto-apply capability — proposed schema/field changes (if built) are staged for human review only, never auto-applied
- Legacy system replacement of any specific named platform

---

## 5. Stakeholders & Personas

| Persona | Role | Interest |
|---|---|---|
| **Reconciliation Analyst** | Investigates and reviews individual breaks | Wants fast, evidence-grounded resolution suggestions; retains ability to override |
| **First-Level Approver** | Approves/rejects mid-confidence or moderate-materiality proposals | Needs a clear rationale and cited evidence before approving |
| **Second-Level Approver (Supervisor)** | Approves/rejects high-materiality or low-confidence escalations | Needs dual-control assurance and full audit visibility |
| **Compliance / Audit** | Reviews historical decisions | Needs an immutable, queryable audit trail with no gaps |
| **Platform / SRE** | Operates the system | Needs observability into cost, latency, cache performance, and failure modes |

---

## 6. Functional Requirements

| ID | Requirement | Priority |
|---|---|---|
| FR-01 | System shall accept new break records via a REST API, associated with a tenant | Must |
| FR-02 | System shall accept asynchronous break-proposal events from an upstream matching service via a message queue. **(As built: Kafka topic `reconcile.break.intake` creates a break only; no proposal arrives by Kafka. Off by default)** | Must |
| FR-03 | System shall allow evidence files to be attached to a break, with format validation | Must |
| FR-04 | System shall extract text and structured fields (dates, amounts, references) from attached evidence | Must |
| FR-05 | System shall retrieve semantically and lexically similar historical resolutions to inform a new proposal | Must |
| FR-06 | System shall produce a structured resolution proposal: category, confidence score (0–1), rationale, and a list of cited evidence identifiers | Must |
| FR-07 | System shall never allow a resolution proposal to cite evidence that was not actually returned by the retrieval step | Must |
| FR-08 | System shall route each proposal to exactly one of: auto-resolve, first-level approval queue, or second-level approval queue, based on confidence and materiality. **(As built: routes are MANUAL_REVIEW, L1 and L2. The auto-resolve rule is off by default, and when enabled the proposal is still stored as L1 approval, so every resolution has a human approver)** | Must |
| FR-09 | System shall allow an authorized first-level approver to approve or reject a queued proposal | Must |
| FR-10 | System shall allow an authorized second-level approver to approve or reject an escalated proposal | Must |
| FR-11 | System shall write an immutable, hash-chained audit record for every decision, automated or human | Must |
| FR-12 | System shall stream investigation progress in real time to a connected client | Should |
| FR-13 | System shall produce a grouped, root-cause-summarized digest of resolved/escalated breaks rather than a flat list | Should |
| FR-14 | System shall isolate all data access by tenant, enforced at the query level, not only the application level | Must |
| FR-15 | System shall reject input containing recognized prompt-injection patterns before invoking any AI model | Must |
| FR-16 | System shall mask or redact recognizable PII patterns before they are sent to any external AI model | Must |
| FR-17 | System shall provide a minimal web interface for investigation review and approval actions | Should |
| FR-18 | System shall offer a single-call investigation mode (default) and an agent mode in which the model gathers evidence through read-only tools; agent mode is disabled unless configured | Should (built) |
| FR-19 | An agent shall only call read-only tools bound to the tenant and break of the running job; no tool or agent may accept a tenant or break identifier as an argument | Must (built) |
| FR-20 | **(Built; measured 2026-11-13: decision rule not met, mode stays off)** System shall offer a supervised investigation mode in which specialist agents (evidence analyst, risk checker, reviewer) contribute to one proposal under a shared budget of model calls, tool calls and time | Could |
| FR-21 | **(Built; measured 2026-11-13: decision rule not met, mode stays off)** In supervised mode a reviewer shall check each cited claim against the exact text of the chunk it cites; an unsupported claim shall route the break to manual review | Could |
| FR-22 | **(Built; measured 2026-11-13: decision rule not met, mode stays off)** No agent may set a routing decision, approve, reject or write; routing remains the deterministic policy gate, and agent output can only increase the amount of human review | Must |
| FR-23 | **(Built; measured 2026-11-13: decision rule not met, mode stays off)** System shall record which agent made each model and tool call and include a digest of that trace in the hash-chained audit trail | Must |
| FR-24 | **(Built; measured 2026-11-13: decision rule not met, mode stays off)** System shall provide a repeatable evaluation that compares all investigation modes on the same labelled cases and reports accuracy, safety, latency and cost | Should |

---

## 7. Non-Functional Requirements

| ID | Category | Requirement |
|---|---|---|
| NFR-01 | Security | All API access requires a valid JWT; roles enforced via RBAC (`RECON_ANALYST`, `APPROVER_L1`, `APPROVER_L2`, `SRE_ADMIN`) |
| NFR-02 | Security | Tenant isolation enforced as a query-level filter, including within vector similarity search, not only at the API boundary |
| NFR-03 | Resilience | External AI model calls are wrapped in a circuit breaker; on failure, the system degrades to a manual-review queue rather than failing the request. **(As built: retry and circuit breaker exist; on failure the job ends FAILED and the break moves to FAILED_RETRYABLE, from which an analyst can start a new investigation. The request itself returned 202 earlier and does not fail; nothing is put on a manual-review queue)** |
| NFR-04 | Scalability | The application tier holds no in-memory session or workflow state; any instance can serve any request, enabling horizontal scale-out |
| NFR-05 | Cost governance | Every AI model invocation is token-budget-checked before execution and cost-logged after execution |
| NFR-06 | Auditability | Every decision (automated or human) is recorded in an append-only, hash-chained log; no update or delete path exists for audit records |
| NFR-07 | Observability | Key operations (retrieval, model invocation, policy decision) emit distributed trace spans and metrics suitable for dashboarding |
| NFR-08 | Performance (design target) | Investigation latency and throughput targets are stated in the Technical Design Document as design goals to validate via local load testing — not claimed as measured production figures |
| NFR-10 | Cost governance (agents) | **(Built; measured 2026-11-13: decision rule not met, mode stays off)** All agents in one run share a single budget (model turns, tool calls, wall clock); adding agents shall not multiply the limits. Supervised mode is judged against a cost ceiling defined in §8 before it is recommended |
| NFR-11 | Safety (agents) | **(Built; measured 2026-11-13: decision rule not met, mode stays off)** The supervised mode is off by default and fails safe: any agent error, timeout or disagreement results in manual review, never in a less-reviewed outcome |
| NFR-12 | Testability | **(Built; measured 2026-11-13: decision rule not met, mode stays off)** Supervised mode is fully testable offline with scripted models, with no paid calls required in CI |
| NFR-09 | Data handling | All evidence and break data used in demonstration is synthetic; no production or client data is used at any stage |

---

## 8. Success Metrics

These are metrics the system is *designed to make measurable*, computed from the project's own synthetic dataset and demo runs — not claims made in advance of measurement.

- Proportion of synthetic breaks resolved via each path (auto / first-level / second-level)
- Cache hit rate (L1 exact-match + L2 semantic) against repeated/similar queries **(as built: L2 Redis semantic cache only)**
- Retrieval quality on a held-out synthetic evaluation set (Recall@5, MRR)
- Token cost per resolved break, and cost avoided via caching and compression
- Guardrail block rate against a fixture set of known injection patterns
- **(Built; measured 2026-11-13: decision rule not met, mode stays off)** Per investigation mode, on the same labelled cases: root-cause accuracy, route accuracy, share of wrong-cause answers that end in manual review, share of cited claims actually supported by the cited text, latency (p50/p95), cost per case, model calls per case
- **(Built; measured 2026-11-13: decision rule not met, mode stays off)** Decision rule, fixed before the run: supervised mode is recommended only if, on the hard-case subset, it at least halves the wrong-cause answers that reach an approver and costs no more than 5x the single-call mode. Otherwise the result is published as negative and the mode stays off. **Outcome 2026-11-13 (gpt-4o-mini, 40 cases): not met.** No wrong cause reached an approver in any mode on the hard set, and supervised cost 7.9x (planned 12.3x) single call. See `reconcile-ai/docs/proof/phase4g-supervised-eval.md`

---

## 9. Assumptions & Constraints

- The system runs entirely in a local development environment (Docker Compose) for demonstration purposes.
- An external LLM provider (or a locally-run model via Ollama) is required for the reasoning step; the system is designed to be provider-agnostic.
- The upstream "automated matching" event feed is simulated, not a connection to a real matching engine.
- Time and resource constraints (solo development, fixed short window) mean some Non-Functional Requirements (notably NFR-08 performance targets) are stated as design intent rather than load-tested at scale.

---

## 10. Risks

| Risk | Mitigation |
|---|---|
| LLM produces a plausible but incorrect resolution | Structured output + mandatory evidence citation + confidence-gated human approval before any ledger write |
| Prompt injection via evidence text or user input | Dedicated guardrail advisor runs before any model invocation; blocked attempts are logged separately |
| Runaway token cost | Hard token-budget check (0-cost, pre-model-call) rejects oversized payloads; two-tier caching reduces repeat-query cost **(as built: one tier, Redis, off by default)** |
| Single point of failure in the AI provider | Circuit breaker + graceful degradation to manual-review queue; provider is swappable via configuration **(as built: the job fails and the break becomes FAILED_RETRYABLE, see NFR-03)** |
| A supervised agent mode costs more and adds nothing on easy cases (the earlier single-agent run was about 8x the cost for similar accuracy) | **(Built; measured 2026-11-13: decision rule not met, mode stays off)** Hard cases added to the evaluation, decision rule fixed in advance, mode stays off by default, negative result published |
| Reviewer and analyst agents share the same model family, so errors may be correlated | **(Built; measured 2026-11-13: decision rule not met, mode stays off)** Option to use a different model for the reviewer; reviewer sees the cited text only, not the analyst's reasoning; limitation stated in the write-up |
| A new agent output influences routing in an unsafe direction | **(Built; measured 2026-11-13: decision rule not met, mode stays off)** Reviewer output enters only as a deterministic gate rule that can escalate to manual review, never relax a route (FR-22) |
| Cross-tenant data leakage | Tenant filter enforced at the query level, including inside vector similarity search, not only in application code |

---

## 11. Glossary

| Term | Definition |
|---|---|
| Break | A detected mismatch between an internal ledger record and an external counterparty record |
| Evidence | Any supporting document attached to a break (statement, email text, spreadsheet) |
| Materiality | The monetary significance of a break, used alongside confidence to determine the approval path |
| HITL | Human-in-the-Loop — a required human decision point before an action is finalized |
| RAG | Retrieval-Augmented Generation — grounding an AI response in retrieved supporting documents |
| RRF | Reciprocal Rank Fusion — a method for combining ranked results from multiple retrieval methods |
| Agent | An AI model that may call tools in a loop before answering; here limited to read-only tools bound to one tenant and break |
| Supervisor | The component that decides which specialist agent runs next; either fixed Java orchestration or a model-planned choice (both evaluated) |
| Reviewer | A specialist agent with no tools that checks whether the cited chunks actually support the claims in a draft |
| Policy gate | The deterministic Java logic (not the AI model) that decides auto-resolve vs. first-level vs. second-level approval |

> **Copy of a private repository document**, snapshot taken 2026-11-17. The source code is private, so links to files in the repository are shown as plain paths. Status labels are the author's own, as written at the time of the snapshot. All data is synthetic.

# ai-gateway — Enterprise LLM Governance Control Plane

![Java](https://img.shields.io/badge/Java-21-orange.svg) ![Spring Boot](https://img.shields.io/badge/Spring_Boot-3.5.x-brightgreen.svg) ![Spring AI](https://img.shields.io/badge/Spring_AI-1.1.x-blue.svg) ![Postgres](https://img.shields.io/badge/Postgres-16_%2B_pgvector-336791.svg) ![Redis](https://img.shields.io/badge/Redis-7-dc382d.svg)

> **Notice: independent demonstration project.**
> This is a generic, fictitious application built to demonstrate software engineering knowledge. It does not contain, reflect or derive from any client's or employer's data, code, functionality, architecture, designs or confidential information, and it is not affiliated with or endorsed by any current or former employer or client. All data is synthetic. Development keys and credentials are throwaway, for local use only. The software is provided "as is", without warranty, and is not intended for production use without independent security review and hardening.

A Java control plane that sits between application teams and model providers. Applications send a
task class and required capabilities; the gateway selects an eligible model, enforces a budget,
masks structured identifiers, resolves a versioned prompt, serves from cache where safe, calls the
provider under bounded failure policy, and writes one evidence row for every outcome — including the
ones that cost money and nobody observed.

> **Postgres decides. Redis projects. Nothing is called until the projection lands.**

```mermaid
flowchart LR
    c[Consumer] --> idem[Idempotency claim]
    idem --> risk[Risk screen · mask · resolve prompt]
    risk --> elig[Budget eligibility]
    elig --> cache{Cache}
    cache -- hit --> led[(Ledger)]
    cache -- miss --> route[Select eligible model]
    route --> res[Reserve: Postgres FOR UPDATE → commit → project]
    res --> call[Provider call · breaker · bulkhead]
    call --> settle[Settle with actual usage] --> led
```

## Status

**In ten lines**

1. One `POST /api/v1/chat` (and a streaming twin) behind JWT and RBAC; the tenant comes only from the token.
2. The gateway selects an eligible model from a catalogue, so consumers declare capabilities, not models.
3. Budget is checked before the cache; a reservation is taken after a miss and before the provider call (Postgres decides, Redis projects).
4. Structured identifiers are masked and risk-screened before routing; a request carrying one goes only to models cleared for it.
5. Exact cache always on, semantic cache opt-in per tenant; both keys are tenant-namespaced.
6. A post-send timeout is `UNKNOWN_OUTCOME`: never retried, never failed over.
7. Every terminal path writes exactly one append-only ledger row, with no prompt or response text.
8. Events leave through an outbox to Kafka and a second deployable, `usage-aggregator`, with consumer deduplication.
9. A React console, metrics, traces, dashboards and alert rules read the same ledger; the live feed is a hint, never the record.
10. Reference manifests (Helm, verified on kind), Terraform (planned, never applied from here) and a measured load test on the stub.

**Status.** Phases 1–10 built, committed and tagged; Phase 11 built offline; Phase 12 (`ops-agent`) lives in its own repository; Phase 13 (13.1–13.6) built and committed. Phase 14 is built and committed: advisory model grading (14.1, calibrated once live on 8 hand-scored items) and an embedding retrieval harness with a SageMaker fine-tune plan, harness only (14.2). Not yet done: any fine-tune (14.2, no real corpus), live Bedrock figures (11.9), any Terraform `apply` (11.10–11.11) and real SMTP delivery (10.8's last step), all waiting on credentials. Per-phase detail, counts and spend: `docs/PHASE_TRACKER.md` (`docs/PHASE_TRACKER.md`). Until a phase is tagged, treat its capabilities as designed, not built.

### Design decisions to ask me about

| Decision | Why | ADR | The test that holds it |
|---|---|---|---|
| A post-send timeout is never retried | A retry after the provider may have processed the call charges twice for one logical call | ADR-12 (`docs/SYSTEM_DESIGN.md`) | `UnknownOutcomeIT` (release-blocking) |
| Recoverable dual write, not a cross-store transaction | Postgres decides and Redis projects; nothing is called until the projection lands | ADR-16 (`docs/SYSTEM_DESIGN.md`) | `ReservationSequenceIT`, `ProjectionFailureIT`, `ReconciliationIT` |
| One ledger row per terminal outcome | An invisible call cannot be audited | ADR-4 (`docs/SYSTEM_DESIGN.md`) | `LedgerCompletenessTest` |
| Cross-tenant access is 404, never 403 | A 403 confirms the resource exists | `CLAUDE.md`, identity rules | `TenantIsolationIT`, `VectorTenantIsolationIT` (release-blocking) |
| Sensitivity routing refuses rather than downgrades, including on failover | A restricted request must not reach an uncleared model | ADR-20 (`docs/SYSTEM_DESIGN.md`) | `RestrictedNeverReachesPublicModelIT` (release-blocking) |
| Semantic cache is opt-in, with a measured reason | Similarity is not identity; the threshold sweep shows where answers diverge | ADR-5 (`docs/SYSTEM_DESIGN.md`), `HNSW_SWEEP.md` (`docs/HNSW_SWEEP.md`) | `SemanticThresholdEvalIT`, `SemanticCacheEndpointIT` |
| Model grading is advisory and never gates promotion | A judge model is nondeterministic, can be flattered by what it grades, and costs money per call | ADR-22 (`docs/SYSTEM_DESIGN.md`) | `JudgeIsolationTest`, `JudgeScoringIT` (release-blocking) |
| Budget fails closed, cache and rate limit fail open | Money is protected; capacity and speed degrade gracefully | ADR-3 (`docs/SYSTEM_DESIGN.md`), ADR-21 (`docs/SYSTEM_DESIGN.md`) | `RedisTokenBucketLimiterIT`, `ProjectionFailureIT` |

## What makes this different from a proxy

| Concern | What this does |
|---|---|
| **Budget** | Reservation → settlement → release → reconciliation, authoritative in Postgres. Correct when Redis dies, not just on the happy path |
| **Failure** | A post-send timeout is `UNKNOWN_OUTCOME`: never retried, never failed over, reservation held, resolved by explicit policy. Automatic failover there charges twice for one call |
| **Prompts** | Immutable versions, checksum-bound to the evaluation that cleared them, promoted by a gated pointer change with no deploy |
| **Evidence** | One ledger row per terminal outcome — blocked, cached, replayed, failed-over, unknown. An invisible call cannot be audited |
| **Isolation** | Tenant in every query signature, cache key, budget key and vector query. Cross-tenant returns 404. Release-blocking negative tests |
| **Streaming** | Server-sent events with a bounded buffer, cancellation that closes the provider connection, and one ledger row whichever way the stream ends. A disconnect is settled, not forgotten |
| **Honesty** | Every control documents what it does *not* cover (see the ADRs in `docs/SYSTEM_DESIGN.md`) |

## Documentation

`docs/SYSTEM_DESIGN.md` is the one to read first — twenty-three ADRs (ADR-1 to ADR-23), each with the cost it accepts.

| | |
|---|---|
| BRD (`docs/BRD.md`) · TDD (`docs/TDD.md`) | requirements · technical design |
| Architecture (`docs/ARCHITECTURE.md`) · Flows (`docs/FLOWS.md`) · Schema (`docs/SCHEMA_DESIGN.md`) | structure · behaviour · data |
| System design + ADRs (`docs/SYSTEM_DESIGN.md`) | decisions and the cost each accepts |
| Events (`docs/EVENTS.md`) · Implementation plan (`docs/IMPLEMENTATION_PLAN.md`) | topics and delivery guarantees · 11 phases plus a PROPOSED Phase 12 |
| Stack reference (`docs/JAVA_AI_STACK_REFERENCE.md`) · CLAUDE.md (`CLAUDE.md`) | verified API shapes · standing rules for AI assistants |
| Phase tracker (`docs/PHASE_TRACKER.md`) · Feature testing guide (`docs/FEATURE_TESTING_GUIDE.md`) | status per plan item · how to exercise each feature by hand |
| Bugs (`docs/BUGS.md`) · Git guide (`docs/GIT_GUIDE.md`) | every defect found, with the test that pins it · commit, tag and CI routine |
| Runbook (`docs/RUNBOOK.md`) · Dependency rationale (`docs/DEPENDENCY_RATIONALE.md`) | one section per alert · why each dependency |

## Quick start

The full setup, with every command and the real responses, is `docs/FEATURE_TESTING_GUIDE.md` section 0. In short:

```bash
docker run -d --name gw-pg -p 55432:5432 -e POSTGRES_DB=gateway -e POSTGRES_USER=gateway \
  -e POSTGRES_PASSWORD=gateway pgvector/pgvector:pg16
docker run -d --name gw-redis -p 6379:6379 redis:7-alpine
./mvnw -B -ntp clean install -DskipTests
TOKEN=$(bash scripts/dev-token.sh tenant-a)          # creates a throwaway key pair under private/dev-jwt/
GATEWAY_DB_URL=jdbc:postgresql://localhost:55432/gateway \
GATEWAY_JWT_PUBLIC_KEY_LOCATION=file:private/dev-jwt/public.pem \
java -jar gateway-api/target/gateway-api-0.1.0-SNAPSHOT-exec.jar

curl -s -X POST http://localhost:8080/api/v1/chat \
  -H "Authorization: Bearer $TOKEN" -H "Idempotency-Key: demo-0001" -H "Content-Type: application/json" \
  -d '{"taskClass":"SUMMARY","messages":[{"role":"USER","content":"Summarise break 42"}]}'

curl -N -X POST http://localhost:8080/api/v1/chat/stream \
  -H "Authorization: Bearer $TOKEN" -H "Idempotency-Key: demo-0002" -H "Content-Type: application/json" \
  -d '{"taskClass":"SUMMARY","messages":[{"role":"USER","content":"[[stub:chunks=5]] hello"}]}'
```

**The default profile is deliberately small**: one process, Postgres, Redis, stub provider. That is
the development loop. Profiles:

| Profile | State |
|---|---|
| `real-provider` | built (Phase 3): OpenAI, key from `OPENAI_API_KEY` in the environment only |
| `it` | built: test defaults, loopback egress allowed, short stream timers |
| events (relay) | built (Phase 8): off by default; `GATEWAY_OUTBOX_RELAY_ENABLED=true` and `GATEWAY_KAFKA_BOOTSTRAP_SERVERS` turn it on |
| console | built (Phase 9): `gateway-console` (React + Vite) on its own origin; `GATEWAY_CONSOLE_ORIGINS=http://localhost:5173` lets the browser call the API. Empty (default): no cross-origin access |
| `telemetry`, `scaleout` | built (Phase 10) in `deploy/compose.yaml`: `demo/up.sh telemetry events console scaleout`. Collector, Jaeger, Prometheus, Alertmanager, Mailpit, Grafana; three replicas behind nginx is a demonstration profile only. `demo/run_faults.sh` forces every alert |

## Repository layout

```text
gateway-core/        domain, ports, policies — no Spring, JPA, Redis or provider SDK
gateway-adapters/    provider adapters, JPA, Flyway, Redis, pgvector, Kafka producer
gateway-api/         the Spring Boot deployable: security, controllers, transactions
gateway-eval/        evaluation datasets, loader and CI tests — CI module, not a service (evaluators are in core)
usage-aggregator/    second deployable (Phase 8): consumes gateway.calls.v1, idempotent rollups, DLQ with two-person replay
gateway-console/     operator console (Phase 9): React + Vite + TypeScript, talks only to the protected API; Vitest unit tests, Playwright golden path
scripts/             dev-token.sh: mints a local test JWT
docs/                the document set
-- not yet present (later phases) --
docker/ deploy/aws/  compose profiles · ECS reference deployment, Phases 10 and 11
demo/ load/          seed data · k6, Phases 10 and 11
```

## Proposed additions

Planned, not built, not yet approved scope. Details: `docs/IMPLEMENTATION_PLAN.md`, ADR-18, ADR-19.

| Proposed | Phase |
|---|---|
| Live-model evaluation run (real provider, figures from the ledger) | 3.8 |
| Alertmanager email delivery | 10.8, built against a mail-capture container; no real SMTP |
| Cross-provider comparison, OpenAI vs Bedrock | 11.9 |
| Terraform + GitHub OIDC, deploy, record, destroy | 11.10–11.11 |
| Ledger metadata export to object storage (no text) | 11.12 |
| `ops-agent`: incident triage with read-only tools, separate repository | 12 |
| Time to first token and stream duration, per model | 13.1 |
| Routing by data sensitivity: restricted identifiers go only to cleared models, refused rather than downgraded (ADR-20); built, classification is only as good as the pattern detectors | 13.2 |
| Measured load test on the stub, figures reconciled with the ledger; built, `docs/evidence/LOAD_TEST.md` | 13.3 |
| Helm chart: reference manifests, verified on kind (not HA), `deploy/helm/ai-gateway` | 13.4 |
| Shared rate limiter as an option (ADR-21); built, `gateway.rate-limit.mode: SHARED` | 13.5 |
| README restructure for reviewers | 13.6 |

Mocks first; real credentials last and only with explicit go-ahead. Phase 13 needs neither and is scheduled
after Phases 10 and 11 are committed.

### Phase 13 design in brief (proposed, not built)

**Routing by data sensitivity.** The detectors that already mask structured identifiers also yield a level.
Routing never sends a request to a model that is not cleared for that level, and failover and the cache follow
the same rule:

```mermaid
flowchart LR
    s[Screen · mask] --> cls[Classify level<br/>types and counts only]
    cls --> elig[Budget eligibility]
    elig --> cache{Cache<br/>key includes level}
    cache -- miss --> route[Eligible = capability ∧ max_sensitivity ≥ level]
    route -- none --> blk[SENSITIVITY_BLOCKED<br/>one ledger row, no reservation]
    route --> res[Reserve] --> call[Call cleared model]
    call -- breaker open --> fo[Failover within the same eligible set only]
    fo -- none left --> blk
```

Card numbers, IBANs and national ids make a request `RESTRICTED`; an email makes it `INTERNAL`; tenant policy may
raise the level, never lower it. The ledger and events record the level, never a matched value. A release-blocking
test forces the cleared model's breaker open and proves the public model receives nothing.

**Time to first token.** Measured on the monotonic clock from provider send to the first content chunk, recorded
once from the stream's single terminal, labelled by model and outcome only, and kept on the ledger row.

**Load test and Kubernetes.** Gatling on the stub provider gives the first latency figures, checked against ledger
row counts and run with virtual threads on and off. A Helm chart maps the existing liveness and readiness probes,
drains streams on shutdown and limits egress to the dependencies and the provider allow-list.
## Honest scope

Read this before quoting anything from this repository.

| Not claimed | Actually true |
|---|---|
| DLP | Pattern-based detection and masking of structured identifiers. Free-form disclosure passes through |
| Prompt injection prevention | Risk signalling. Containment comes from the model having no tools and no authority here |
| Atomic budget reservation | Recoverable dual write — Postgres authoritative, Redis projected, reconciler repairs (ADR-16) |
| Each event delivered once | At-least-once; consumers deduplicate by event id |
| Output quality evaluation | Deterministic property assertions with pinned settings, gating promotion. No baseline comparison, no drift handling, no human labelling — a future phase, see "What evaluation does not cover" |
| Invoice-level cost accuracy | Provider-reported usage where available, `ESTIMATED` where not, with pricing provenance |
| High availability | Three local replicas share one machine. The ECS deployment is a reference deployment |
| Microservices platform | One deployable with enforced internal seams, plus one second process from Phase 8 |
| Sensitive data never leaves the boundary (proposed, 13.2) | Requests with detected structured identifiers go only to cleared models, including on failover. Free-form confidential text is not detected and is classed `PUBLIC` unless tenant policy raises it |
| Production latency figures (proposed, 13.3) | Stub-provider figures on stated local hardware only |

### What events do not cover

Phase 8 publishes one fact per terminal call to Kafka and keeps rollups from it in a second process. It does not provide:

- **A once-only delivery guarantee.** Delivery is at-least-once with consumer deduplication by event id. A crash after
  the broker acknowledged and before the relay committed republishes the row; the aggregator counts it once.
- **Authoritative rollups.** The aggregator's rollups are eventually correct. The ledger (and the gateway's own
  `GET /api/v1/usage/rollups`, which reads it) answers anything where correctness matters.
- **Payloads in events.** Events carry identifiers, enums, checksums and numbers. A publish-time policy refuses any
  key without a declared kind, so text cannot leave by being added later.
- **An automatic DLQ.** A record the consumer refuses is parked and alerts on the first entry; replay and discard each
  need a second person. Nothing replays on its own.
- **A broker cluster.** The demonstration profile is one KRaft node (`apache/kafka:3.9.1`) with replication factor 1.

### What the console does not cover

Phase 9 is an operator console over the existing API. It does not provide:

- **A sign-in flow.** You paste a token (`scripts/dev-token.sh` locally). There is no identity provider in this repository.
- **A live record.** The feed is a best-effort hint with no replay and no `Last-Event-ID`. The ledger is the record, and the
  console reads it after every connect.
- **Browser-side money.** Totals are the gateway's. Money arrives as a JSON number and is shown with `toFixed(8)`, exact up to
  15 significant digits (every cost below $10,000,000); sending it as a decimal string would remove even that limit.
- **A security-audit view.** `security_audit` has no read endpoint yet; it would want its own role.
- **The golden path in CI.** CI runs the console's typecheck, unit tests and build. The Playwright run needs a gateway and is
  run locally (`docs/FEATURE_TESTING_GUIDE.md` §9.8).

### What streaming does not cover

Phase 7 delivers a governed call as server-sent events and settles it however it ends. It does not provide:

- **Exact accounting for a cut stream.** When the provider reports usage (OpenAI sends it in the last chunk only), the
  row carries the provider's numbers. When it does not, the row carries an estimate of what the gateway *received*,
  labelled `ESTIMATED`, and the label stays. The provider may have generated, and billed, more than was received before
  the cancellation took effect. There is no provider billing feed here to reconcile against.
- **Instant disconnect detection.** A client that has gone is noticed on the next failed write or the next keep-alive,
  not at the moment it leaves.
- **Retry or failover once a stream has started.** A stream that has begun cannot say "nothing happened", so it is never
  re-sent and never moved to another model. A `PARTIAL_STREAM` or `UNKNOWN_OUTCOME` stream holds its idempotency key: a
  retry is a new key chosen by the client.
- **Caching or replay of streamed text.** A stream neither reads nor writes either cache, and the replay of a completed
  key is the plain JSON replay with no output, because the text was never stored.
- **A status code for a failure after the first byte.** Once the stream has started the status is 200 and the end is an
  `error` event; everything before the first byte (rate limit, key conflict, risk block, budget, no model) is an ordinary
  JSON error.

### What evaluation does not cover

The Phase 6 gate runs **deterministic property assertions** against a candidate prompt version, with the
candidate's own pinned settings, and scores the weighted pass rate. It checks things like "the answer is exactly
one of these labels", "it contains this phrase", "it is at most 12 characters". It does not cover:

- **Answer quality.** A text can satisfy every assertion and still be wrong, unhelpful or misleading. Nothing here
  judges meaning, and nothing uses a model to grade a model.
- **A frozen baseline.** A candidate is scored against fixed thresholds, not against the active version's answers.
  There is no baseline-versus-candidate comparison, so a regression that stays above the threshold is not seen.
- **Drift.** The provider can change a model's behaviour under the same name. A run is evidence about one moment;
  there is no scheduled re-run, no alert on a score that falls, and no rule that a promotion expires.
- **Human review for high-risk prompts.** The only review gates are the existing roles (a system prompt change needs
  a prompt admin) and the checksum the caller names; no workflow asks a person to read sampled outputs.
- **Repeatability.** Temperature 0 is the most repeatable setting a provider offers, not a guarantee. A run of one
  case is one draw; settings not pinned to temperature 0 are required to repeat each case at least three times,
  and a case passes only if every repeat does.
- **Coverage.** A dataset is at most 50 hand-written synthetic cases. Passing it says the candidate handles those cases.

These belong to a future evaluation-depth phase (frozen baseline comparison, drift handling, human review for
high-risk prompts). It is named, not scheduled, and not built.

**No cost or latency figure appears anywhere in this repository until it has been measured here,
from these ledger rows, on stated hardware.**

## Deliberately not built

Retrieval inside the gateway (ADR-1) · consumer-specified model pinning (ADR-6) · semantic caching
on by default (ADR-5) · automatic retry on unknown outcomes (ADR-12) · cross-store distributed
transactions (ADR-16) · tool execution or agent planning inside the gateway · multi-region DR · managed Kafka ·
fine-tuning pipelines · a shared governance library for consumers to embed, which would put security
logic back on every consumer's deploy cycle.

Full reasoning, with the cost each decision accepts: docs/SYSTEM_DESIGN.md (`docs/SYSTEM_DESIGN.md`).

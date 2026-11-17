> **Copy of a private repository document**, snapshot taken 2026-11-17. The source code is private, so links to files in the repository are shown as plain paths. Status labels are the author's own, as written at the time of the snapshot. All data is synthetic.

# platform (private)

Integration home for the four-repo platform. It owns no service code. It holds what the services share:

| Folder | Holds |
|---|---|
| `docs/` | The baseline BRD, TDD and implementation plan (v2.1). There is one baseline; corrections are edited inline, never appended. |
| `contracts/` | JSON Schemas for cross-service contracts, validated by provider and consumer tests. |
| `compose/` | `compose.yaml`, which includes each service repo's own Compose file by path. |
| `evidence/` | `EVIDENCE_REGISTER.md` and spike results. A claim with no row is not used. |
| `scripts/` | Read-only helpers, for example `check-repos.sh`. |

## Service repos

The repos sit in different folders, so nothing here uses relative `../` paths or submodules. Set the paths once:

```bash
cp .env.example .env              # then edit the three *_DIR values
scripts/check-repos.sh            # prints HEAD and PLATFORM_FACTS.md status per repo
docker compose --env-file .env -f compose/compose.yaml config
```

| Variable | Repo | Role |
|---|---|---|
| `RECONCILE_AI_DIR` | reconcile-ai | Java system of record |
| `EKA_DIR` | enterprise-knowledge-assistant | Python retrieval |
| `GATEWAY_DIR` | ai-gateway | Java chat-completion governance |

RERO is conditional Q3 work and has no variable until the Q2 review approves it.

## Status (2026-11-15)

| Q0 item | State |
|---|---|
| Q0.0 repo skeleton, Q0.1 facts files committed in all three repos | done (reconcile-ai facts refreshed for `0393997` in Q0.3) |
| Q0.4 EKA doc corrections | done (EKA `3ae3100`, pushed) |
| Q0.2 ai-gateway doc corrections, Q0.7 gateway capability inventory | committed in ai-gateway (`8f5b627`, `4d7966d`); inventory copied to `evidence/inventory/`; gateway CI green on `4d7966d` |
| Q0.3 reconcile-ai doc corrections | done (reconcile-ai PR #9, `a721fb3`, CI green); private BRD/TDD annotated (not in git) |
| S-1, S-2, S-3 spikes | **pass**, see `evidence/spikes/` |
| §5.3 gateway confirmation, `GatewayLedgerIT` | merged into `docs/TDD.md` §5.3 (gateway `cdddbac`, CI green) |
| S-3 readiness, consumer contract and v1 schemas | gateway `89db71d`, `4c6b5c8` (`ci` #23, `image` #11 green); schemas copied to `contracts/gateway/` |
| **Q0 gate** | **closed 2026-11-15**: all three repos committed and clean (`scripts/check-repos.sh` OK), CI green. Next: Q1 |

Spikes run locally with `scripts/spikes/up.sh`, then `s1.sh`, `s2.sh`, `s3.sh`, then `down.sh`. They need the three `*_DIR` paths and the `KC_*` dev secrets in `.env`. The integration flags (`KNOWLEDGE_ENABLED`, `GATEWAY_ENABLED`) default to `false`, and `KNOWLEDGE_FAIL_CLOSED` defaults to `false`.

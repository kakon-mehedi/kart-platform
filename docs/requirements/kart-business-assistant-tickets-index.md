---
doc_type: tickets-index
capability: kart-ai-assistant (Kart Business Assistant)
status: approved
generated_by: human-directed consolidation pass, sourced from three already-approved per-service tickets.md files
source:
  - docs/services/kart-ai-assistant-service/tickets.md
  - docs/services/kart-analytics-service/tickets.md (Product Performance Addition v1.1 section)
  - docs/client/kart-admin-web/tickets.md (AI Assistant epic addition section)
  - docs/requirements/genai-business-assistant-spec.md
  - docs/requirements/change-log.md (2026-08-18 entry)
---

# Kart Business Assistant — Consolidated Ticket Index

This is a **read-only index**, not a fourth source of truth. Every ticket listed here is defined, described, and dependency-linked in full in its own repo's `tickets.md` — this document exists so a sprint planner can see every ticket needed to build this capability end-to-end **in one place**, across the three doc sets it touched, without checking three files separately. If a ticket's description here and its home file ever disagree, the home file wins.

**Three repos, one capability:**

| Repo | Doc | New work |
|---|---|---|
| `kart-ai-assistant-service` (new, 19th deployable repo) | [`tickets.md`](../services/kart-ai-assistant-service/tickets.md) | Full v1 build — 24 engineering tickets + 2 spikes |
| `kart-analytics-service` (existing, v1.1 addition) | [`tickets.md`](../services/kart-analytics-service/tickets.md) — "Product Performance Addition (v1.1)" section | 5 engineering tickets |
| `kart-admin-web` (existing, new feature area) | [`tickets.md`](../client/kart-admin-web/tickets.md) — "Epic Addition: AI Assistant" section | 5 engineering tickets |
| `kart-identity-service` (existing, config-only) | Cross-repo ticket only (`XREPO-ID-1`, tracked in `kart-ai-assistant-service/tickets.md`) — no `kart-identity-service/tickets.md` entry exists yet | 1 config change |

**Total: 35 engineering tickets + 2 product-owner spikes, across 4 repos.** No ticket exists anywhere for Seller/Vendor or Geographic analytics — both are permanently out of scope (ADR-0026, ADR-0027), not deferred, so no backlog placeholder was created for either.

---

## Build order (cross-repo dependency chain)

The critical path runs **`kart-analytics-service` → `kart-ai-assistant-service` → `kart-admin-web`** for the one capability (ranking) that needs all three; everything else in `kart-ai-assistant-service`'s Phases 1–2 and 4 (using the nine *pre-existing* Analytics endpoints) and most of `kart-admin-web`'s feature-area work can proceed in parallel with the Analytics addition, not strictly after it.

```text
kart-identity-service                 kart-analytics-service (v1.1 addition)
  XREPO-ID-1 (ai-assistant.query        ANL-16 → ANL-17 → ANL-18 → ANL-19 → ANL-20
  scope, config-only, ADR-0025)                                        │
        │                                                              │
        ▼                                                              ▼
kart-ai-assistant-service (new)  ◄──────────────────────────── (blocks AIA-9 / XREPO-ANL-1)
  Phase 1: AIA-1..5  (Foundation — no cross-repo blocker)
  Phase 2: AIA-6..8  (existing 9 endpoints — no cross-repo blocker)
  Phase 3: AIA-9     (ranking wiring — BLOCKED on ANL-19/ANL-20 above)
  Phase 4: AIA-10..16 (conversation/grounding — needs Phase 2 minimum, Phase 3 for full ranking E2E)
  Phase 5: AIA-17     (visualization — needs Phase 2)
  Phase 6: AIA-18..21 (security/observability — needs AIA-2, needs XREPO-ID-1 for full E2E RBAC verify)
  Phase 7: AIA-22..24 (production hardening — needs SPIKE-1/SPIKE-2 numbers)
        │
        ▼  (XTEAM-2 — blocks integration testing beyond a hand-mocked contract)
kart-admin-web (new feature area)
  ASST-1  (chat shell + store — no cross-repo blocker beyond AIA Phase 1 stub)
  ASST-2  (API integration — needs AIA-5, the Phase 1 stub, at minimum)
  ASST-3  (response rendering — needs ASST-2)
  ASST-4  (chart rendering — needs ASST-3)
  ASST-5  (session persistence — needs ASST-1)
  Full worked example (source spec §22/§23 Example 1, "top 5 selling products")
  end-to-end requires: ANL-19/20 + AIA-9 + ASST-2..4 ALL shipped.
```

**Practical sequencing recommendation:** file `XREPO-ID-1` (Identity config change) and start `ANL-16` (Analytics collection) in the very first sprint — both are small, low-risk, and sit on the critical path for everything downstream. `AIA-1..8` and `ASST-1` can start in parallel with either. Do not schedule an engineer against `AIA-9` or the full `ASST-2..4` integration slice before `ANL-19`/`ANL-20` (Analytics) and `AIA-5` (the `kart-ai-assistant-service` Phase 1 stub) are respectively live.

---

## `kart-ai-assistant-service` — full ticket list (see [tickets.md](../services/kart-ai-assistant-service/tickets.md) for full descriptions/citations)

| ID | Phase | One-line | Depends on |
|---|---|---|---|
| AIA-1 | 1 Foundation | Service scaffold, CI, observability wiring, Gateway route registration | — |
| AIA-2 | 1 | `ai-assistant.query` scope-check middleware | AIA-1 |
| AIA-3 | 1 | Model Gateway connectivity (plan tier + explain tier) | AIA-1 |
| AIA-4 | 1 | `ConversationSession` Redis storage + per-conversation lock | AIA-1 |
| AIA-5 | 1 | `POST /v1/ai-assistant/query` skeleton, stubbed response, real auth path | AIA-1..4 |
| AIA-6 | 2 Business semantic/query layer | Metric/entity/dimension/endpoint registry (versioned config) | AIA-1 |
| AIA-7 | 2 | Deterministic intent→endpoint `QueryPlan` builder + validation gate | AIA-6 |
| AIA-8 | 2 | Execute `QueryPlan` against Analytics' 9 existing endpoints | AIA-7 |
| AIA-9 | 3 Product performance | Wire ranking-query intent to the new Analytics endpoint | AIA-6, AIA-7, **XREPO-ANL-1** |
| AIA-10 | 4 AI integration & conversation | NL→intent translation (plan call) | AIA-3, AIA-7 |
| AIA-11 | 4 | Follow-up context merge + stale-provenance disclosure | AIA-10, AIA-4 |
| AIA-12 | 4 | Ambiguity detection & clarification | AIA-10 |
| AIA-13 | 4 | Grounded explanation generation + numeric-grounding validation | AIA-8, AIA-3 |
| AIA-14 | 4 | "Unsupported" vs. "zero rows" classification | AIA-6, AIA-10 |
| AIA-15 | 4 | Conversation session expiry handling | AIA-4, AIA-11 |
| AIA-16 | 4 | Resilience: 3 circuit breakers (plan/explain/Analytics calls) | AIA-8, AIA-10, AIA-13 |
| AIA-17 | 5 Visualization | Deterministic visualization selection + response envelope assembly | AIA-13 |
| AIA-18 | 6 Security & observability | `AuditRecord` write path | AIA-10..15 |
| AIA-19 | 6 | Prompt-injection containment test pass | AIA-10, AIA-13 |
| AIA-20 | 6 | Rate limiting on the query endpoint | AIA-5 |
| AIA-21 | 6 | Security review pass | AIA-2, AIA-18, AIA-19, AIA-20 |
| AIA-22 | 7 Production hardening | Load test at internal-tool concurrency | AIA-21 |
| AIA-23 | 7 | Cost monitoring/alerting dashboards | AIA-18, **SPIKE-2** |
| AIA-24 | 7 | Latency SLA dashboards/alerts | AIA-16, **SPIKE-1** |
| SPIKE-1 | Product owner | Set end-to-end latency SLA | blocks AIA-16 tuning, AIA-24 |
| SPIKE-2 | Product owner | Set LLM cost budget/ceiling | blocks AIA-23 |
| XREPO-ID-1 | Cross-repo → `kart-identity-service` | Add `ai-assistant.query` to Admin/Support-Agent role→scope config (ADR-0025) | — |

## `kart-analytics-service` — Product Performance Addition (v1.1) (see [tickets.md](../services/kart-analytics-service/tickets.md))

| ID | One-line | Depends on |
|---|---|---|
| ANL-16 | Create `product_performance_dashboard` collection + schema validation | — |
| ANL-17 | Product-performance projector (incremental + nightly reconciliation) | ANL-1, ANL-5, ANL-16 |
| ANL-18 | Product-performance indexes (bucket + 6 ranking compound indexes) | ANL-16 |
| ANL-19 | `GET /internal/v1/dashboards/product-performance` endpoint + contract tests | ANL-17, ANL-18 |
| ANL-20 | Ranking-tie tiebreak + rank-reorder-on-reconciliation handling | ANL-17, ANL-19 |
| XREPO-AIA-1 | Cross-repo pointer → `kart-ai-assistant-service`'s `AIA-9` consumes this work | — |

## `kart-admin-web` — AI Assistant feature area (see [tickets.md](../client/kart-admin-web/tickets.md))

| ID | One-line | Depends on |
|---|---|---|
| ASST-1 | Chat panel UI shell + feature-scoped Signal Store | AUTH-3 |
| ASST-2 | `POST /v1/ai-assistant/query` API integration (loading/in-flight/abort) | ASST-1, XTEAM-2 |
| ASST-3 | Response-shape rendering (answer/table/clarification/error/contract-drift) | ASST-2 |
| ASST-4 | Chart rendering via ECharts, 7-value enum + `table_only` fallback | ASST-3 |
| ASST-5 | `conversationId` persistence + idle-timer activity extension | ASST-1, AUTH-3 |
| XTEAM-2 | Cross-repo pointer → `kart-ai-assistant-service` Phase 1/3 as blockers | — |

---

## What is deliberately NOT here

- No ticket for Seller/Vendor performance reporting (ADR-0026 — permanently out of scope).
- No ticket for geographic/location analytics (ADR-0027 — permanently out of scope, gated on a `kart-order-service` contract change no one has scoped or scheduled).
- No `kart-order-service` ticket for the `OrderConfirmed.address` value-object work ADR-0027 flags as a prerequisite for any *future* geographic-analytics attempt — that is explicitly a different team's future initiative, not part of this capability's build, and inventing a ticket for it here would misrepresent it as in-scope work.

## Sign-off

- [x] All three per-service `tickets.md` files confirmed present and internally consistent with this index as of 2026-08-18.
- [x] Cross-repo dependency chain (Identity → Analytics → AI Assistant → Admin Web) verified in both directions (each blocking ticket is cited from both the blocker's and the blocked ticket's own file).
- [ ] Sprint planner to confirm sprint allocation against the build-order recommendation above.

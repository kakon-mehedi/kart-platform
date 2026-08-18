---
doc_type: change-log
service: kart-platform
status: accepted
layer: requirements
---

# BRD Change Log

Tracked amendments to `kart-requirements.md` made after its initial approval, per `PLATFORM_BLUEPRINT.md` §3's directory design. Each entry is dated, states what changed and why, and cross-references any cascading updates made elsewhere in the platform docs.

## 2026-07-25 — Observability stack named concretely

**What changed:** §23 Observability and the top-level **Stack** line now name the concrete tools implementing each pillar, instead of describing the pillars generically:

- Structured Logging → **Serilog** → OpenTelemetry Collector (OTLP) → **Grafana Loki**
- Metrics → **Prometheus**
- Distributed Tracing → **OpenTelemetry SDK** → **Grafana Tempo**
- Visualization & Alerting (new row) → **Grafana** (the LGTM stack's single pane of glass)

**Why:** the BRD previously mandated the *pattern* (structured logs, RED metrics, W3C trace propagation) but left tool choice open. The platform has now standardized on the Grafana LGTM stack for every microservice so dashboards, alerting, and cross-service trace correlation work identically across all 18 services rather than per-service tool sprawl.

**Cascading updates made in the same pass:**

- New reusable, project-agnostic standard: `agent-reusables/docs/standards/observability-standards.md` (full pillar detail, package choices, sampling policy, dashboard/alerting conventions).
- `docs/standards/kart-conventions.md` — new **Observability** section: `Kart.Shared.Observability` package, the 100%-trace-coverage service tier (Order/Inventory/Payment/Shipping), per-service correlation-id field convention, platform stack ownership (`kart-infra`/`kart-devops`).
- `docs/PLATFORM_BLUEPRINT.md` §8.2 Monitoring Agent — Tools row now names Prometheus/Tempo/Loki query APIs explicitly.
- Every `docs/services/<name>/requirement-spec.md` — Observability NFR row added or updated to name the concrete stack and reference the standards above.
- Every `docs/services/<name>/design-decisions.md` — new "Observability & Instrumentation" decision entry recording the per-service correlation field and trace-sampling tier.

No BRD scope, domain rule, or NFR *target* changed — this is a tooling concretization of an existing requirement, not a new requirement.

## 2026-08-18 — Kart Business Assistant capability added (new bounded context + two incremental extensions)

**What changed:** a new product capability, the **Kart Business Assistant** — a natural-language, business-facing Generative AI assistant for `Admin`/`Support Agent` users, specified in full in `docs/requirements/genai-business-assistant-spec.md` — was carried through the full agent pipeline (`PLATFORM_BLUEPRINT.md` §8.2) and shipped as:

- **One new bounded context, `kart-ai-assistant-service`** (the platform's 19th deployable repo) — full doc set produced: `requirement-spec.md`, `edge-cases.md`, `design-decisions.md`, `architecture.md`, `ddd-model.md`, `api-contract.yaml` (`POST /v1/ai-assistant/query`), `database-design.md`, `event-contract.md` (zero publish/consume — purely synchronous), `message-bus-manifest.json`, `tickets.md`.
- **One incremental addition to `kart-analytics-service`** (already-approved, v1.1) — a new `product_performance_dashboard` read model + `GET /internal/v1/dashboards/product-performance` endpoint (ranked/limited product queries by revenue, units sold, or order count), the one net-new analytics capability this assistant's founding example query ("top 5 selling products") requires. Recorded as additions in place across `requirement-spec.md` (§6 item 7), `edge-cases.md` (two new entries), `ddd-model.md` (11th read-model projection), `database-design.md` (11th collection + projector + indexes), `api-contract.yaml` (v1.0.0 → v1.1.0), and `tickets.md` (`ANL-16`–`ANL-20`) — no existing dashboard, aggregate, or endpoint touched.
- **One incremental addition to `kart-admin-web`** (already-approved) — a new §3.6 "AI Assistant" feature area (chat panel + table/chart rendering surface), identical access for both `Admin` and `Support Agent`, no new login path. Recorded as additions across `requirement-spec.md` (§3.6), `architecture.md` (new `kart-ai-assistant-service` dependency row + `features/ai-assistant/` folder), `design-decisions.md` (Signal Store reuse; ECharts — a genuinely new charting-library pick, no prior one existed; shared data-table primitive reuse), `edge-cases.md` (five new client-only edge cases), and `tickets.md` (`ASST-1`–`ASST-5`) — none of the other five feature areas touched.

**Why:** `business-flows.md` Flow 18 has always named the destination (ad-hoc analytics self-service) but the platform had no natural-language query capability and no ranked/limited product query at all — every existing Analytics dashboard requires the caller to already know which endpoint, with what parameters, to call. This capability closes that gap for the self-service segment of Flow 18 without building a general BI product (explicitly deferred, see the source spec's own §2.2/NG6).

**Four ADRs were required to unblock this pass** (cross-cutting/contradiction-resolving decisions the source spec itself flagged as OPEN QUESTIONs, resolved before the pipeline could proceed):

- **[ADR-0024](../adr/0024-ai-assistant-service-scope-and-integration.md)** — `kart-ai-assistant-service` is confirmed a new, independently deployed bounded context (resolves the source spec's §26-Architecture-1), owning only conversation/session state and its own audit log, with exactly one synchronous dependency (`kart-analytics-service`) and zero async events.
- **[ADR-0025](../adr/0025-ai-assistant-query-scope.md)** — new scope `ai-assistant.query`, Identity-issued via role→scope mapping into both `Admin`'s and `Support Agent`'s JWTs (no persisted grant table), enforced via a `bearerAuth` OpenAPI scheme (resolves §26-Security-1).
- **[ADR-0026](../adr/0026-seller-vendor-scope-gap-ruling.md)** — ruled that `business-flows.md`'s "Seller Performance Reports" (Flow 16/18) is aspirational business-flows scope, not an implemented bounded context anywhere in the platform's 18-service architecture; permanently out of scope for this capability, no extensibility hook reserved (resolves §26-Data-2/§26-Business-2).
- **[ADR-0027](../adr/0027-order-confirmed-address-shape-gap.md)** — checked `kart-order-service`'s own contract and found `OrderConfirmed.address` has no confirmed shape anywhere (no value object, no column, no OpenAPI schema); geographic analytics is therefore gated on a `kart-order-service` producer-side contract change, not a cheap Analytics-side read-model addition as the source spec's own framing had left open (resolves §26-Data-1, more precisely than originally framed).

**Genuinely open, human-owned items carried forward (not resolved by this pass, per the source spec's own framing — see `kart-ai-assistant-service/requirement-spec.md` §8):** revenue gross-vs-net (Business-1, Analytics' own call), conversation session TTL (UX-1), default date range when unstated (UX-2), end-to-end latency SLA and LLM cost budget (Infrastructure-1 — tracked as `SPIKE-1`/`SPIKE-2` in `kart-ai-assistant-service/tickets.md`, assigned to a product owner, not engineering).

**Cascading updates made in the same pass:**

- `docs/architecture/system-context.md`, `container-diagram.md`, `service-boundaries.md` — `kart-ai-assistant-service` added as the 19th deployable service (one new node, two new sync edges: Gateway→AI, AI→Analytics; no other dependency).
- `docs/ddd/ubiquitous-language.md` — new `## Owned by \`kart-ai-assistant-service\`` section (`ConversationSession`, `ResolvedIntent`, `RankingSpec`, `TurnProvenance`, `QueryPlan`, `AuditRecord`, with an explicit naming-collision note distinguishing `ConversationSession` from Identity's own login `Session`) plus one new row appended to the existing `kart-analytics-service` term table (`ProductPerformanceDashboard`).
- A consolidated cross-service ticket index was produced: `docs/requirements/kart-business-assistant-tickets-index.md`.

No BRD scope, domain rule, or existing NFR *target* changed for any of the 18 already-shipped services — this is a new capability plus two additive, backward-compatible extensions, not a revision of anything previously approved.

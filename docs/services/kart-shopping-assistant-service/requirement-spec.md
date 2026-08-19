---
doc_type: requirement-spec
service: kart-shopping-assistant-service
status: approved
approval_note: >
  Approved to proceed through the pipeline per explicit human directive
  (this capability's own commissioning task, mirroring how
  kart-ai-assistant-service/requirement-spec.md was itself approved): every
  decision that would otherwise block a downstream pipeline stage is now
  closed by ADR — ADR-0028 (bounded context, formalizing §1's own adopted
  reasoning), ADR-0029 (closes Security-2, the RBAC scope name/mechanism and
  guest-vs-authenticated access split), and ADR-0030 (closes Architecture-7
  at the policy level — the FastAPI/Python stack exception is accepted and
  its Python-native-equivalent constraints are fixed; the concrete
  mechanisms are left to design-decisions.md, the next stage). The
  remaining items in §10 (Architecture-1, Architecture-2, Data-1, Data-2,
  Data-3, Security-1, Business-1, Business-2, Business-3, UX-1, UX-2,
  NFR-2, NFR-3) are correctly non-blocking for Architecture/DDD/API/DB/Event
  design — they gate cross-service contract work (owned by other services'
  teams), privacy/legal/product sign-off, or production-readiness, not this
  service's own design pipeline — and are carried forward as open questions
  in every downstream doc rather than silently resolved.
generated_by: requirement-agent
source:
  - docs/requirements/genai-customer-assistant-spec.md (Kart Shopping Assistant spec, "single source of truth" for this service)
  - docs/services/kart-ai-assistant-service/requirement-spec.md (closest structural precedent — read to draw a hard boundary, not to extend)
  - docs/adr/0024-ai-assistant-service-scope-and-integration.md
  - docs/adr/0025-ai-assistant-query-scope.md
  - docs/adr/0026-seller-vendor-scope-gap-ruling.md
  - docs/adr/0027-order-confirmed-address-shape-gap.md
  - docs/requirements/kart-requirements.md (BRD — §5 stack line, §24 RBAC/security model)
  - docs/services/kart-order-service/{requirement-spec,api-contract.yaml}
  - docs/services/kart-payment-service/requirement-spec.md
  - docs/services/kart-cart-service/{requirement-spec,api-contract.yaml}
  - docs/services/kart-offer-service/requirement-spec.md
  - docs/services/kart-user-service/api-contract.yaml
---

# Requirement Spec: kart-shopping-assistant-service

## 1. Scope

Covers a single, brand-new capability: the **Kart Shopping Assistant**, a natural-language, **Customer**-facing Generative AI assistant, surfaced inside `kart-web`, that **takes action** on the customer's own behalf — tracking, cancelling, and returning orders, discovering/comparing products, applying coupons, adding items to cart, and checking out — against Kart's real backend services (`docs/requirements/genai-customer-assistant-spec.md` §1.1, §0). Its primary source is not a BRD excerpt but a dedicated, already-detailed specification document, exactly as `kart-ai-assistant-service`'s own requirement-spec was seeded from `genai-business-assistant-spec.md` — this document restates that source's requirements in this repo's requirement-spec voice/format and, per its own founding instruction, resolves what a single service can safely resolve on its own authority while carrying forward everything else as an explicit Open Question (§10).

**This is not `kart-ai-assistant-service`, and must never be built as an extension of it.** That service is a new, independently deployed, **read-only** bounded context, closed by **ADR-0024** to own no domain data, call only `kart-analytics-service`, and never mutate anything — gated by a coarse role check limited to `Admin`/`Support Agent` and a scope (`ai-assistant.query`) Identity-issued only for those two roles (**ADR-0025**). This capability is the structural opposite in every load-bearing dimension the source spec identifies (§1.2): it is `Customer`-facing, not `Admin`/`Support Agent`-facing; it **mutates** real orders, carts, coupons, and (indirectly) money, not merely reads aggregate numbers; and it calls up to nine downstream services synchronously, not one. Re-opening ADR-0024's closed ruling to admit a `Customer`-scoped, mutating mode into `kart-ai-assistant-service` is explicitly rejected by the source spec (§1.2) as "reopening and reversing ADR-0024's own closed ruling for an unrelated caller population" — this document adopts that conclusion and does not re-litigate it.

**Bounded-context decision — now formally closed by ADR-0028** (adopted from source spec §1.2/§12 D1): `kart-shopping-assistant-service` is a new, independently deployed 20th deployable repo, following the platform's one-bounded-context-per-repo convention (`container-diagram.md`'s closing caption, cited by the source spec §1.2). **ADR-0028** formalizes this exactly, mirroring ADR-0024's own closed reasoning applied to a materially different actor/mutation/blast-radius profile — it is not an independent judgment call this document is making fresh, so it is not carried forward as an Open Question the way the placement question genuinely was for `kart-ai-assistant-service` before ADR-0024 existed. This service's own RBAC scope name/mechanism is now also closed, by **ADR-0029** (`shopping-assistant.act`, plus the guest-vs-authenticated access split). What remains open is this service's data-retention/session-lifetime specifics (§10, Data-3/UX-2) — new decisions this service needs its own settling for, not inherited from any ADR.

### 1.1 Technology stack — a deliberate, single-service exception (flagged, not resolved here)

Every other service on this platform is built on **.NET 9 / ASP.NET Core** (`kart-requirements.md` line 5's stack banner, applied uniformly across all 19 existing deployable repos, including `kart-ai-assistant-service`). **`kart-shopping-assistant-service` is to be built in FastAPI (Python) instead** — a deliberate, single-service deviation from that platform default, per this pipeline stage's own founding instruction for this run.

This is stated plainly here as a fact this spec's functional/non-functional requirements must be read against, but it is **explicitly not resolved by this document** — per this agent's own escalation rule, a decision this consequential and cross-cutting is not this pipeline stage's to make unilaterally. It is flagged forward as **Architecture-7** (§10) for the Architecture Agent / Design-Decision Agent stage, which must, at minimum:
- Write a dedicated ADR for this stack exception (mirroring how every other cross-cutting platform decision in this repo — ADR-0010, ADR-0024, ADR-0025 — got its own ADR rather than being asserted inside a `requirement-spec.md`), justifying why FastAPI/Python is the right shape for this service specifically (e.g. Python's GenAI/LLM-tooling ecosystem maturity) and what operational cost the platform accepts by running one Python service inside an otherwise all-.NET fleet (a second language's worth of on-call runbooks, base images, CI/CD templates, dependency-scanning tooling, etc. — none of which this document invents an answer for).
- Audit which platform-wide **shared-library conventions have no Python equivalent yet** and need one designed before this service can meet the same cross-cutting obligations every other service meets "for free" via `kart-shared`:
  - **`Kart.Shared.Auditing`'s automatic audit-field injection** (`created_at`/`updated_at`/`created_by`/`updated_by` via a `SaveChanges` interceptor reading the resolved principal, `kart-requirements.md` §24.3) — this service will hold at least conversation/session state and an audit log of its own (§7, mirroring `kart-ai-assistant-service`'s two owned aggregates per ADR-0024), so it needs *some* mechanism meeting the same guarantee; no EF Core interceptor exists in a Python/FastAPI stack, and no equivalent has been designed anywhere in this repo yet.
  - **The JSON-manifest-driven RabbitMQ topology** (`kart-requirements.md` §8/§9's exchange/queue provisioning convention, applied uniformly across all 19 .NET services) — this service is not expected to need any publish/consume relationship for its core loop (§7's "no new asynchronous dependency" stance, mirroring ADR-0024's own zero-event-edges ruling for the sibling service), but if that ever changes (§10 Architecture-5), the manifest-driven provisioning tooling's .NET-specific implementation would need a Python-callable equivalent that doesn't exist today.
  - **EF Core interceptor patterns generally** (row-level-security session-variable injection per `kart-requirements.md` §24.1.4's `ICurrentPrincipalAccessor` pattern, optimistic-concurrency `version` column handling, etc.) — every one of these is currently a .NET-only, EF-Core-coupled mechanism; a Python ORM (e.g. SQLAlchemy) equivalent has not been designed by any prior pass through this pipeline.
- This document does **not** pick a Python ORM, ASGI server, or dependency-injection framework, and does not attempt to design the missing shared-library equivalents itself — doing so would be exactly the kind of invented, unauthorized cross-cutting decision this agent's own escalation rule exists to prevent.

### 1.2 Who uses it

**`Customer` role only**, authenticated, inside `kart-web` (source spec §1.3). A guest (unauthenticated) session may use read-only intents only (§2, rows marked "Read-only"); every mutating intent requires an authenticated `Customer` JWT (§7). `Admin`, `Support Agent`, and `Partner API` principals are explicitly out of scope — they are `kart-ai-assistant-service`'s/`kart-admin-web`'s own users (source spec §1.3, Non-Goal NG9, §11).

## 2. Intent Catalog

Restated from the source spec's full per-intent treatment (§2.1–§2.11), condensed to this table. **Every confirmed gap named by the source spec is carried forward here verbatim and unsoftened** — these are load-bearing facts for the tickets this pipeline will eventually produce, not smoothed over.

| # | Intent | Mutating? | Backing service/API | Status |
|---|---|---|---|---|
| 1 | Return a product | Yes | `kart-order-service`'s planned `POST /orders/{id}/return-request` | **GAP.** Designed at the `kart-web` client-integration-map level only; **absent from `kart-order-service`'s own approved `requirement-spec.md`/`api-contract.yaml`** (source spec §2.1, §8.1; this document's §10 Architecture-1). |
| 2 | Track an order / ETA | No | `GET /orders/{id}` + `GET /tracking/{trackingId}` | Existing endpoints; the exact field/mechanism carrying `trackingId` on `GET /orders/{id}` is unconfirmed in either service's approved contract (source spec §2.2; §10 Architecture-2). |
| 3 | Cancel an order | Yes | `POST /orders/{id}/cancel` | Existing, legal only pre-`Shipped` (`kart-order-service/requirement-spec.md` §2, §4). |
| 4 | Request a refund | Indirect (via #1 or #3) | No direct endpoint — never calls `kart-payment-service` directly | **No third, direct "just refund me" endpoint exists, and none is invented here** (source spec §2.4, Non-Goal NG5). A `Shipped`-but-not-`Delivered` order has **no legal customer-self-service path today** (§10 Business-1). |
| 5 | Product discovery / search with constraints | No | `GET /v1/search` (`kart-search-service`) | Existing, fully backed — no gap (source spec §2.5). |
| 6 | Compare two or more products | No | `GET /v1/products/{sku}` (`kart-product-service`), called once per SKU | No batch/multi-SKU endpoint exists; **resolved here as a single-service default** — see §9. |
| 7 | Add a recommended/best-match item to cart | Yes | `GET /search` or `GET /recommendations/{userId}` composed with `POST /v1/cart/items` (`kart-cart-service`) | Orchestration-only gap, no new downstream endpoint required (source spec §2.7). |
| 8 | Discover and apply the best available coupon | Yes (apply step) | `kart-offer-service` | **GAP — confirmed.** `kart-offer-service` exposes only `POST /coupons/validate` (requires an already-known code) and `GET /promotions/active` (a listing, not a cart-aware ranking, confirmed against `kart-offer-service/requirement-spec.md` §5) — **no "best coupon for this cart" auto-discovery endpoint exists** (source spec §2.8, §8.1; this document's §10 Data-1). |
| 9 | Checkout using saved default address + payment method | Yes | Default address: existing (`kart-user-service`'s `Address.isDefault`, scoped per `type`). Default payment method: **GAP — confirmed.** No field, endpoint, or schema anywhere in `kart-user-service` or `kart-payment-service` exposes a queryable, customer-selectable "saved payment methods, one marked default" concept — `kart-payment-service` stores only a per-charge, opaque `gateway_token` (confirmed against `kart-payment-service/requirement-spec.md` §4/§5: "no unmasked PCI data exists to protect in the first place," no listable vault entry). | Source spec §2.9; this document's §10 Data-2. |
| 10 | "Wrong item received" → arrange a replacement | Yes, if resolved to a Return-request | None — **no bounded context, aggregate, endpoint, or event anywhere in the platform's 18 approved services models a replacement/exchange concept, confirmed by direct inspection of `kart-order-service`'s and `kart-payment-service`'s own approved docs.** | **The single most significant domain gap in this catalog** (source spec §2.10) — same shape of gap ADR-0026 ruled on for Seller/Vendor reporting. Falls back to Return-request (refund-only) or human escalation; never promises a replacement shipment. See §10 Business-2. |
| 11 | Wishlist add / move-to-cart | Yes, low risk | Existing `kart-wishlist-service` + `POST /v1/cart/items` | No gap (source spec §2.11). |
| 12 | General order/product support Q&A, with escalation | No / escalation | Composes intents #2/#5, or hands off to a human Support Agent (Flow 14) | This is the assistant's designated "I can't do that" backstop (FR-010), not a new capability surface. Escalation mechanism resolved at §9. |

## 3. Functional Requirements

Restated from the source spec's FR-001–FR-010 (`genai-customer-assistant-spec.md` §3), service-scoped.

- **FR-001 — Natural-language request intake.** Accept free-text input from an authenticated (or, for read-only intents, guest) `Customer` session, plus JWT and session id. A mutating intent from an unauthenticated session is rejected *before* the LLM is invoked — the assistant never spends a model call planning an action it cannot authorize.
- **FR-002 — Intent resolution & slot-filling.** Deterministic application code — never the LLM — validates the model's structured-output intent against the registered catalog (§2). An unresolvable/ambiguous required slot (an ambiguous `orderId`, a missing `reasonCode`) triggers clarification (FR-008), never a best guess.
- **FR-003 — Tool-calling execution against real services.** Execute only via a fixed, versioned intent→endpoint mapping table (application config, never LLM-authored). An intent whose backing capability is a confirmed gap (rows 8, 10 in §2) is routed to FR-010, never attempted against a nonexistent endpoint.
- **FR-004 — Confirmation before any mutating action.** Every intent marked "Yes" under Mutating in §2 requires an explicit, plain-language confirmation — naming the concrete effect — before the corresponding call executes. No mutating call is ever made speculatively.
- **FR-005 — Re-validation at execution time (no LLM-adjudicated eligibility).** The assistant never itself decides cancellability/returnability/refund-eligibility; the owning service's own business rules, evaluated at the moment of the call, are authoritative. A `409` is explained truthfully, never silently retried as transient.
- **FR-006 — Result summarization.** The natural-language response is generated strictly from the already-executed call's actual result; every concrete fact must be traceable to a field in the result payload, validated post-generation. On validation failure, fall back to a template-built response from raw data and log the failure.
- **FR-007 — Multi-turn context carry-forward.** A follow-up is interpreted against the last resolved intent/slots in the session; only fields the new message actually changes are overwritten. A pending mutating confirmation is never silently carried forward across a topic switch — a fresh FR-004 confirmation is always required for a newly (re-)resolved mutating action.
- **FR-008 — Ambiguity detection & clarification.** Underspecified requests are met with a clarification turn offering concrete, selectable options (e.g. the customer's own recent orders) — never a default guess at `orderId`, `reasonCode`, or SKU.
- **FR-009 — Partial-failure handling (no orphaned side effects).** For any intent composed of more than one downstream call (row 7's coupon-then-checkout composition, row 9's checkout sequence), a mid-sequence failure is reported as the actual resulting state truthfully, never as a generic error or a false "success."
- **FR-010 — Out-of-scope / unsupported handling.** A confirmed gap (rows 8, 10), a genuinely non-commerce ask, or an out-of-role request returns a distinguishable "not supported" response naming what's missing — never a fabricated action or a silent no-op reported as success.

### AI vs. application responsibilities

| Responsibility | Owner |
|---|---|
| Understand natural language, extract slots, identify intent, detect ambiguity | **AI** |
| Generate confirmation/summary text | **AI**, constrained to reference only values from the fetched/executed result (FR-006) |
| Validate intent against the registered catalog (§2) | **Application** |
| Enforce authorization (§7) | **Application** |
| Decide eligibility (cancellable? returnable? refund limit?) | **The owning downstream service** (Order, Payment, Offer) — never the AI, never this service itself |
| Execute the mutating call | **Application** (only after FR-004 confirmation) |
| Enforce idempotency, detect/report partial failure, persist audit records | **Application** (§6, §7) |

## 4. Non-Functional Requirements

| Attribute | Target | Basis |
|---|---|---|
| Availability | **99.9% (secondary tier) — resolved here, single-service default, no ADR (§9).** This service initiates calls into the order path but is not itself a Saga participant. | Mirrors `kart-cart-service/requirement-spec.md`'s own Decision D6 reasoning (a pre/post-Saga, customer-facing edge service is secondary-tier, not order-path-tier) and `kart-ai-assistant-service/requirement-spec.md`'s own analogous "best-effort, not 99.99%" resolution for a service that calls into, but is not part of, a Saga. |
| Latency (end-to-end conversational turn) | **OPEN QUESTION — §10 NFR-2.** No BRD/ADR figure exists for a GenAI-inclusive round trip; downstream calls inherit their own existing budgets (e.g. `kart-order-service`'s P95 < 300ms write path), but the full multi-service turn's own target is unset and not invented here. | Source spec §4 |
| Scalability | Stateless orchestration layer, horizontally scalable per the platform default (`kart-requirements.md` §3). Conversation/session state (§7) is the one per-session-affinity concern to size for. | Source spec §4 |
| Reliability | Every mutating call must be idempotent (§6); inherits the platform's general "at-least-once + idempotent consumers" convention. | Source spec §4 |
| Security | Customer-JWT-scoped, three-check RBAC (§7); TLS everywhere, no plaintext secrets (`kart-requirements.md` §24) — unchanged platform posture. | Source spec §4 |
| Observability | Full trace coverage on every mutating conversational turn, correlation ID propagated end-to-end into every downstream call; one audit record per turn (§7). | Source spec §4 |
| Cost (LLM token usage) | **OPEN QUESTION — §10 NFR-3.** No budget/ceiling is specified anywhere in the source documents; this service's worst-case call count per turn (plan → confirm → execute → summarize) exceeds `kart-ai-assistant-service`'s own two-call pattern, so its cost profile is not simply "the same, twice." | Source spec §4 |
| Maintainability | Independently deployable; contract-tested against every downstream service's own approved `api-contract.yaml`; versioned intent catalog (§2) as reviewable application config, never LLM-authored. | Source spec §4 |

## 5. Domain Invariants

Distilled from the source spec's Safety & Guardrails (§5), Conversation/Session State (§6), and AuthN/AuthZ (§7) sections into the business rules that must hold regardless of implementation:

- **The LLM never invents business authorization to act.** Every mutating intent (§2) requires an explicit, plain-language confirmation naming the concrete effect before the corresponding downstream call executes (FR-004) — the platform-wide analogue of `kart-ai-assistant-service`'s "the LLM never invents business data," restated here for *eligibility/authorization* rather than facts.
- **The assistant never adjudicates eligibility itself.** Cancellability, returnability, and refund/coupon validity are decided exclusively by the owning downstream service at the moment of the call, never pre-judged by this service or the LLM (FR-005). A rejection from that call (e.g. `409`) is authoritative, never a bug to route around.
- **Idempotency on every mutating call.** This service generates and persists one idempotency key per attempted mutating action within a conversation turn and reuses it on any internal retry — it never mints a fresh key on retry, following the platform's existing pattern (`kart-order-service`'s and `kart-payment-service`'s own `Idempotency-Key` header convention).
- **No cross-customer action, under any conversational framing.** No confirmation step, clarification, or prompt-injected instruction may cause this service to act on an `orderId`/`cartId`/`addressId` that does not belong to the authenticated caller — enforced by forwarding the caller's own JWT/assertion to every downstream call so that service's own per-resource ownership check (`kart-requirements.md` §24.1.2/§24.1.4), not this service's own judgment, is what actually gates the mutation.
- **Untrusted fetched content is inert data, never instructions.** Product descriptions, reviews, or any other content this service reads while composing a response must never be re-interpreted as a new instruction to act on (mirrors `kart-ai-assistant-service`'s own prompt-injection-defense invariant; the consequence class here is categorically worse since this service can act).
- **No orphaned side effects on partial multi-step failure.** For any composed, multi-call intent (row 7's item-resolution-then-cart-add, row 8's coupon-discovery-then-apply, row 9's checkout sequence), a mid-sequence failure must leave the system in a state the customer is accurately told about — reusing the platform's existing Saga-compensation and idempotency mechanisms, never a new, locally-invented reconciliation mechanism.
- **A resolved-but-not-yet-confirmed mutating intent is not carried forward across a topic switch.** If the customer changes the subject before confirming a mutating action, that pending confirmation is discarded, not silently reused for a later, different action (§6).
- **Data ownership boundary (mirrors ADR-0024's ruling for the sibling service, adopted here for the same reason, §1):** this service owns only its own conversation/session state and its own audit log; it never becomes a second owner or cache of Order/Cart/Payment/Offer/Product/User/Delivery-Tracking domain data (source spec §12 D3 — "no new database beyond conversation/audit state").

## 6. Data & PII

This capability's PII surface is **substantially larger** than the read-only `kart-ai-assistant-service`'s, whose logs were analytics-metadata-only in the common case. This assistant routinely handles order contents (items, quantities, prices), full address detail (read and referenced in confirmations), and masked payment-method references (last-4/label only — the raw `gateway_token` must never appear in a conversational response or log, mirroring `kart-requirements.md` §24.1.5's "no unmasked PCI data exists to protect in the first place" precedent for Payment). Conversation transcripts themselves will routinely *contain* order/address/product content as a structural consequence of what this assistant does, unlike the sibling's aggregate-metadata-only logs.

Every turn (successful, clarification, confirmation-pending, or error) produces one audit record — necessarily including what mutating action was proposed/confirmed/executed and against which resource, since that *is* the auditable event this capability's own risk profile (§5) most needs.

**Retention policy for these conversation transcripts is not decided here** — no existing document (`kart-web/privacy.md`, `security.md`) addresses AI-conversation-transcript retention at all. Carried forward at §10, Data-3.

## 7. Integration & Architecture Touchpoints

| Service | Capability used | Status |
|---|---|---|
| `kart-order-service` | `POST /orders`, `GET /orders/{id}`, `POST /orders/{id}/cancel` | Existing |
| `kart-order-service` | `POST /orders/{id}/return-request` | **Gap** — §2 row 1, §10 Architecture-1 |
| `kart-delivery-tracking-service` | `GET /tracking/{trackingId}` | Existing |
| `kart-cart-service` | `GET /cart`, `POST /cart/items`, `POST /cart/checkout` | Existing |
| `kart-offer-service` | `POST /coupons/validate`, `GET /promotions/active` | Existing (single-code validation / listing only) |
| `kart-offer-service` | "best coupon for this cart" auto-discovery | **Gap** — §2 row 8, §10 Data-1 |
| `kart-search-service` | `GET /search` | Existing |
| `kart-product-service` | `GET /products/{sku}` (single) | Existing; no batch variant — resolved as accepted for v1, §9 |
| `kart-recommendation-service` | `GET /recommendations/{userId}` | Existing |
| `kart-user-service` | `GET /users/{id}` (addresses, default flag) | Existing for address; no payment-method-default equivalent — §10 Data-2 |
| `kart-payment-service` | — | **Never called directly** — all money movement is triggered indirectly via Order's cancel/return-request surface (mirrors `kart-web`'s own existing constraint; Non-Goal NG5) |
| `kart-wishlist-service` | `/wishlist` | Existing |
| `kart-identity-service` | JWT validation (at Gateway), `sub`/`roles`/`scopes` claims | Existing, unchanged |
| Model Gateway (LLM provider) | Plan / confirm / summarize calls | New, provider-agnostic (§8) |

No new **asynchronous** event publish/consume relationship is required for this capability's core loop — resolved at §9 (Architecture-5). This is a wide synchronous fan-out (up to nine downstream peers) with no precedent of this breadth among existing customer-facing services; the corresponding resilience-pattern decision is resolved at §9 (Architecture-6).

## 8. LLM-Provider Framing

Provider-agnostic via a model-gateway abstraction, matching `kart-ai-assistant-service`'s own posture: no named vendor/model anywhere in this document's requirements. Structured-output/function-calling (constraining the LLM to the intent schema in §2) and bounded tool-calling (a fixed, enumerable registry, not an open agentic loop — source spec §12 D2) are required as a **safety control**, not merely a determinism nicety, because this service's LLM output can trigger a mutating call. RAG/embeddings/vector databases are not required, for the same reason `kart-ai-assistant-service` didn't need them: this capability resolves natural language into a small, enumerable set of structured intents against structured APIs, not open-ended retrieval over unstructured text.

## 9. Resolved Decisions (Single-Service Engineering Defaults — No ADR Needed)

Every item below was raised as an Open Question in the source spec (`genai-customer-assistant-spec.md` §14) and is closed here because resolving it does not change any *other* service's own contract and requires no cross-cutting or product-authority decision — the same pattern `kart-order-service/requirement-spec.md` §6 and `kart-cart-service/requirement-spec.md` §6 already used for their own single-service defaults.

1. **Availability tier — resolved (was NFR-1).** 99.9% (secondary tier), not the order-path 99.99% tier. This service initiates calls into the order path but is not itself a Saga participant — the same reasoning `kart-cart-service`'s own Decision D6 already applied to a comparable pre/post-Saga, customer-facing edge service. This is an SLO for this service's own deployment, not a contract another service depends on, so no ADR is needed. See §4.
2. **Compare-intent batch/multi-SKU lookup — resolved (was Architecture-3).** For v1, the Compare intent (§2 row 6) issues N sequential `GET /products/{sku}` calls rather than requiring a new batch endpoint on `kart-product-service`. For the small, bounded N a customer names in one conversational turn, this is an acceptable workaround, not a blocking dependency — mirrors `kart-order-service`'s own "Partial fulfillment / split shipment — out of scope for v1" pattern of deferring a nice-to-have rather than blocking on it. A future batch endpoint remains a legitimate, non-blocking optimization request to `kart-product-service`'s own team, not a decision this document forces on them now.
3. **Human-escalation mechanism — resolved (was Architecture-4).** For v1, escalation (referenced in §2 rows 4, 10, 12) is a **conversational handoff/UI affordance only** — the assistant announces the handoff and directs the customer to `kart-web`'s own existing "Contact Support" surface, with context (order id, discrepancy description) carried forward for the human agent to see. No new backend Support-ticket record is created or required by this service, since **no Support-ticket/case bounded context exists anywhere on the platform to write one into** (confirmed by the same absence this document's §2 row 10 already names for Exchange/replacement — there is no more a Support-case aggregate than there is an Exchange aggregate). This doesn't change any other service's contract; if a future Support-ticketing service is ever built, wiring this assistant's escalation into it is a normal, incremental integration addition at that time, following the same precedent ADR-0026 set for incremental registry additions.
4. **Event consumption — resolved (was Architecture-5).** This service consumes **zero** platform events for v1, mirroring ADR-0024's own zero-async-edges ruling for `kart-ai-assistant-service`. A future proactive capability (e.g. consuming `OrderDelivered` to prompt "how was your order?") is explicit future/non-goal scope, not decided here — adding it later is a scope change for that future pass, not something this document blocks on or designs a placeholder for now (mirrors ADR-0026's "no speculative extensibility reserved" reasoning, applied here to event edges rather than schema fields).
5. **Synchronous fan-out resilience pattern — resolved (was Architecture-6).** This service adopts **per-edge-independent circuit breakers** for each of its downstream synchronous dependencies (§7's table), the same resilience pattern `kart-admin-service`'s own comparably wide synchronous fan-out already uses. This applies an existing, already-approved platform pattern to a new caller rather than inventing a new cross-cutting mechanism, so it doesn't require its own ADR — a failure in any one downstream peer (e.g. `kart-offer-service` being down) must not take down this service's ability to serve the other, unaffected intents.

**Non-blocking, carried forward for awareness only (implementation-level detail, not a decision this spec needs to close):**
- Final request/response schema for this service's own inbound endpoint (analogous to `kart-ai-assistant-service`'s `POST /v1/ai-assistant/query`) — **API Design Agent**.
- Concrete intent→endpoint mapping table's wire format, and the exact structured-output JSON schema per intent — **API Design/DDD Agents**.
- Numeric circuit-breaker thresholds/timeouts for §9 item 5's per-edge breakers — **Architecture Agent**, the same way `kart-order-service`'s own Saga-timeout numbers were left to that stage for retuning under real load data.

## 10. Open Questions / Flagged Ambiguities

These remain genuinely open — no ADR has closed them, no product/business owner has confirmed them, and this pipeline stage does not resolve them by picking a default. Each is a human-approval gate per this agent's own escalation rule. Category tags match the source spec's own §14 convention.

**Architecture**
- **Architecture-1.** `kart-order-service`'s `POST /orders/{id}/return-request` (§2 row 1) is designed at the `kart-web` client-integration-map level but absent from Order's own approved backend spec — does Order's spec get amended to add it, or was it deliberately deferred? This is a **confirmed, load-bearing gap**, not softened here: the Return intent cannot be implemented until `kart-order-service`'s own team closes it.
- **Architecture-2.** Exact field/mechanism by which `GET /orders/{id}` surfaces a `trackingId` for the Track-order intent (§2 row 2) — not fixed in either `kart-order-service`'s or `kart-delivery-tracking-service`'s current approved contract.
- ~~**Architecture-7.** The FastAPI/Python stack deviation (§1.1) needs its own ADR...~~ **Closed by ADR-0030.** The stack exception is accepted at the policy level; the ADR fixes the binding guarantee every Python-native equivalent (auditing, RabbitMQ manifest, RLS) must satisfy and requires a shared `kart_shared` package. The *concrete* mechanisms (ORM, structured-output library, etc.) are left to `design-decisions.md`, the next stage — struck through rather than deleted so the resolution is visible, per this repo's own convention (`docs/requirements/genai-business-assistant-spec.md`'s own struck-through Data-5 entry).

**Data**
- **Data-1.** `kart-offer-service` needs a new "best coupon for this cart" endpoint (§2 row 8) — exact contract (input: cart snapshot shape; output: single best offer vs. ranked list) is that service's own team's call. **Confirmed, load-bearing gap.**
- **Data-2.** Whether a queryable "customer's saved default payment method" concept exists anywhere today (possibly client/BFF-side, not backend) or needs to be built, and where it should live (§2 row 9) — affects whether this is a `kart-user-service` extension, a new `kart-payment-service` capability, or a `kart-web` BFF-session concept this assistant would need a new, narrow API to query. **Confirmed, load-bearing gap.**
- **Data-3.** Retention policy for AI conversation transcripts containing order/address/PII content (§6) — no existing document addresses this; needs a privacy/legal decision, not an engineering default. Also touches whether `kart-user-service`'s existing GDPR Right-to-Delete/export fan-out needs to be extended to this service's own conversation store.

**Security**
- **Security-1.** Whether assistant-mediated traffic needs a stricter per-customer rate limit than ordinary authenticated API traffic, given its ability to trigger cancels/refunds/checkouts in rapid succession — this is a `kart-api-gateway`-owned tiering/config decision, not something this document can set a number for.
- ~~**Security-2.** Exact name/mechanism for this service's own RBAC scope...~~ **Closed by ADR-0029.** Scope name `shopping-assistant.act`, Identity-issued for `Customer` role only, plus the guest-vs-authenticated three-check enforcement split (Tier 1 is a role-*exclusion*, not a role-*requirement*, since this service has a genuine anonymous-caller population unlike its sibling).

**Business**
- **Business-1.** A `Shipped`-but-not-yet-`Delivered` order has no legal customer-self-service refund/cancel path today (§2 row 4) — is this an accepted gap in the storefront's own customer journey, or should a "cancel in transit" capability be scoped as a separate initiative? Not this document's call.
- **Business-2.** Whether/when a real Exchange bounded context (§2 row 10) should be built is a BRD/flow-catalog-owner-level decision, the same category ADR-0026 ruled the Seller/Vendor gap belonged to — the single most significant domain gap in this catalog, not softened here.
- **Business-3.** Should there be a per-customer, per-time-window ceiling on assistant-mediated mutating actions (cancels/returns/checkouts) that escalates to human review, and if so, what threshold? **No number is invented here**, per this pipeline stage's own founding instruction against inventing business-rule thresholds.

**UX**
- **UX-1.** Exact confirmation-affirmation mechanism (explicit button/selectable option vs. a typed "yes," FR-004) — a product-design decision, not assumed here beyond "must be explicit."
- **UX-2.** Conversation session lifetime — no figure exists anywhere in the source documents for this specific session type; a product/session-policy decision, not an engineering default (mirrors `kart-ai-assistant-service`'s own carried-forward UX-1 session-TTL open item).

**Non-Functional**
- **NFR-2.** End-to-end conversational-turn latency target (§4) — no BRD figure exists for this shape of interaction; **not invented here**.
- **NFR-3.** LLM cost/token budget per turn (§4) — unset, and this service's worst-case call count per turn exceeds the read-only sibling's two-call pattern; **not invented here**.

## Sign-off

- [ ] `kart-order-service` owner confirms/resolves Architecture-1 (the `return-request` contract gap) and Architecture-2 (`trackingId` exposure)
- [ ] `kart-offer-service` owner scopes Data-1 (best-coupon-discovery endpoint)
- [ ] `kart-user-service`/`kart-payment-service`/`kart-web` BFF owners jointly resolve Data-2 (saved default payment method)
- [ ] Privacy/legal owner resolves Data-3 (conversation-transcript retention)
- [ ] `kart-api-gateway` owner resolves Security-1 (rate-limit tiering)
- [x] Identity/security owner resolves Security-2 (RBAC scope name/mechanism) — **closed by ADR-0029**
- [x] Architecture Agent / Design-Decision Agent owns Architecture-7 (FastAPI/Python stack ADR and shared-library-equivalence audit) — **closed at the policy level by ADR-0030**; concrete mechanisms left to `design-decisions.md`
- [ ] BRD/flow-catalog owner confirms Business-1, Business-2
- [ ] Product/business owner sets Business-3 (spend/refund ceiling, if any)
- [ ] Product/UX owner resolves UX-1, UX-2
- [ ] Platform architecture owner confirms NFR-2, NFR-3 (latency SLA, LLM cost ceiling) before production readiness
- [x] Reviewed by: _approved to proceed through the pipeline — see `approval_note` above_

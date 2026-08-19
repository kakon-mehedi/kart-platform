---
doc_type: adr
status: accepted
---

# ADR-0028: `kart-shopping-assistant-service` Is a New, Independently Deployed, Mutation-Capable Bounded Context — Distinct From, and Not a Reopening of, ADR-0024

## Status

Accepted

## Context

`docs/requirements/genai-customer-assistant-spec.md` (the Kart Shopping Assistant spec) and its restated `docs/services/kart-shopping-assistant-service/requirement-spec.md` both already *state* that this capability must be a new, independent bounded context, distinct from `kart-ai-assistant-service` — the source spec's own §1.2/§12-D1 works through the Context/Options/Decision/Why/Trade-off reasoning, and the requirement-spec's §1 adopts that conclusion directly rather than carrying it forward as an open question. But per this repo's own established convention (ADR-0010 formalized Admin Service's scope; ADR-0024 formalized this exact placement question for the sibling service), a cross-cutting bounded-context/data-ownership/integration-pattern decision does not get to rest on a single service's own `requirement-spec.md` assertion — it needs its own ADR so the DDD, Architecture, Database Design, and Event Design Agents each have one settled, citable ruling to build against, rather than re-deriving or re-litigating the placement question at each stage.

This ADR is the structural mirror of ADR-0024, for the opposite ruling. ADR-0024 closed `kart-ai-assistant-service` as **read-only, single-dependency, owning no domain data** — a ruling this new service cannot simply inherit, because it is `Customer`-facing (not `Admin`/`Support Agent`-facing), it **mutates** real orders, carts, coupons, and (indirectly) money, and it calls up to nine downstream services synchronously (requirement-spec.md §7), not one. Three things need to be pinned down before later pipeline stages can proceed without re-litigating this each time:

1. **Deployment topology** — a new 20th deployable repo, versus somehow being folded into an existing one (already rejected by the source spec's own §1.2 reasoning: RBAC-role mismatch, mutation-boundary conflict with ADR-0024's closed ruling, and blast-radius separation).
2. **Data ownership boundary** — whether this service may hold a local copy of any domain data it orchestrates across (order contents, cart state, product data), or whether — like `kart-ai-assistant-service` under ADR-0024, and like Admin Service under ADR-0010 — it must never become a second owner/source of truth for data another service already owns.
3. **Non-reopening of ADR-0024** — this new service's existence must not be read, by any later agent, as license to widen `kart-ai-assistant-service`'s own closed scope (e.g. "just let Customers use the existing assistant too"). The two must remain structurally and operationally separate.

## Decision

**`kart-shopping-assistant-service` is a new, independently deployed bounded context** — its own repo, its own deployable, following the platform's one-bounded-context-per-repo convention (`docs/architecture/container-diagram.md`). It becomes the **20th** deployable repo (`kart-ai-assistant-service` was the 19th, per ADR-0024).

**Its data-ownership boundary is closed, by direct analogy to ADR-0024's own table for the sibling service:**

| It may own | It may never own |
|---|---|
| Conversation/session state (the last resolved intent + slots per session, requirement-spec.md §5's "resolved-but-unconfirmed mutating intent" invariant) | Any copy of Order/Cart/Payment/Offer/Product/User/Delivery-Tracking domain data |
| Its own audit log (one `AuditRecord` per turn, requirement-spec.md §6) | A cache or local recomputation of any eligibility decision, price, or stock level a downstream service itself owns (requirement-spec.md §5: "the assistant never adjudicates eligibility itself") |
| Nothing else | A payment-method reference, gateway token, or any PCI-scoped value beyond a masked display reference already returned by an owning service |

This mirrors ADR-0024's own framing exactly, generalized to more peers: `kart-shopping-assistant-service` is an **orchestration/control-plane caller** of every service it touches, never a second owner of any of their data — the same relationship `kart-ai-assistant-service` holds toward `kart-analytics-service`, just across nine peers instead of one, and with the added authority (via each downstream service's own eligibility/authorization checks, never its own) to trigger a real mutation.

**Integration pattern: a wide, explicitly-enumerated synchronous fan-out, each edge independently circuit-broken** (requirement-spec.md §7's table, §9 item 5) — `kart-order-service`, `kart-cart-service`, `kart-offer-service`, `kart-search-service`, `kart-product-service`, `kart-recommendation-service`, `kart-user-service`, `kart-delivery-tracking-service`, and `kart-wishlist-service`. **`kart-payment-service` is deliberately excluded from this list and is never called directly** — all money movement is triggered only indirectly, via `kart-order-service`'s own customer-facing cancel/return-request surface, mirroring the same constraint `kart-web` itself already observes (`docs/client/kart-web/api-integration-map.md`; requirement-spec.md Non-Goal NG5). This is a materially wider fan-out than any existing customer-facing service has today; a failure in any one peer (e.g. `kart-offer-service` down) must degrade only the intents that peer backs, never the whole service — the same resilience discipline `kart-admin-service`'s own wide fan-out already uses.

Zero asynchronous publish/consume relationships exist for this service's v1 core loop (requirement-spec.md §9 item 4), mirroring ADR-0024's own zero-event-edges ruling for the sibling service — stated explicitly, not left silently blank.

**It sits behind the Gateway like every other `kart-web` backend call** — no bypass. Unlike `kart-ai-assistant-service` (which has no unauthenticated use case at all), this service's Gateway route accepts **both anonymous and `Customer`-authenticated traffic**, since a subset of its intents (product discovery, comparison, general Q&A — requirement-spec.md §2 rows 5, 6, 12) are legitimately guest-accessible, mirroring `kart-web`'s own existing anonymous-browsing posture (`kart-requirements.md` §4.4). The finer-grained "is this specific request allowed" distinction (guest read-only vs. authenticated mutating) is an application-level check (requirement-spec.md FR-001), not a Gateway-routing distinction — this is formalized further, with concrete scope naming, in **ADR-0029**.

**Explicit non-reopening of ADR-0024:** this ADR does not modify, extend, or reinterpret ADR-0024 or ADR-0025 in any way. `kart-ai-assistant-service` remains read-only, single-dependency, `Admin`/`Support Agent`-only, exactly as those ADRs closed it. The two services' Gateway routes are gated by mutually exclusive coarse-role checks (`Customer` here; `Admin`/`Support Agent` there per ADR-0025) and share no code path, no data store, and no RBAC scope. A future request to let `Customer`s use `kart-ai-assistant-service`, or to let `Admin`/`Support Agent`s use `kart-shopping-assistant-service`, would each require its own new ADR reopening the relevant closed ruling — this ADR does not pre-authorize either.

## Consequences

- The bounded-context question implicit in `requirement-spec.md` §1 is now formally closed, not merely asserted — no further Architecture/DDD/Database/Event Design Agent work needs to re-ask this question.
- `docs/architecture/system-context.md`, `container-diagram.md`, and `service-boundaries.md` must each add this service as a new participant: one new node/edge set in `container-diagram.md` (sync edges to the nine peers named above, explicitly none to `kart-payment-service`), one new `## kart-shopping-assistant-service` section in `service-boundaries.md` including its per-edge circuit-breaker posture, and the system-context's service count updated to 20.
- The DDD Agent must model exactly two aggregates of this service's own — `ConversationSession` and `AuditRecord` — and must not introduce an `Order`, `Cart`, `Product`, or any other aggregate duplicating another service's domain data, per the ownership table above.
- The Database Design Agent designs storage only for conversation-session state and the audit log — no domain-data read model of its own, and no local cache of any downstream service's eligibility/price/stock data.
- The Event Design Agent documents zero publish/consume relationships for v1, explicitly, per the same precedent ADR-0024 set.
- This service becomes the 20th deployable repo; any future capacity-plan or repo-count reference elsewhere in `docs/` should be updated to match when next touched.
- ADR-0024 and ADR-0025 remain fully in force, unmodified, for `kart-ai-assistant-service`.

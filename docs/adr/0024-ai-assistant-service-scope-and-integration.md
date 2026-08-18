---
doc_type: adr
status: accepted
---

# ADR-0024: `kart-ai-assistant-service` Is a New, Independently Deployed Bounded Context, and It Owns No Domain Data of Its Own

## Status

Accepted

## Context

`docs/requirements/genai-business-assistant-spec.md` (the Kart Business Assistant spec) introduces a capability with no precedent anywhere in the BRD's 18-deployable-repo service list (`kart-requirements.md` §2.1) — it is genuinely new, not an addition to an already-scoped service. The spec's own §25-D1 already works through the placement question (module inside `kart-admin-service`? synchronous extension of `kart-analytics-service`? a new service?) and recommends option (c), a new service, but the spec explicitly leaves this a flagged **OPEN QUESTION** rather than a closed decision: §26-Architecture-1 states "Should `kart-ai-assistant-service` be a fully independent deployable (this spec's default, D1) or could it start as a module inside an existing service and be extracted later?... the call belongs to whoever owns the platform's repository-strategy decisions." Per this repo's own convention (ADR-0010, ADR-0023: a cross-cutting or contradiction-resolving question that a single service's own docs cannot close on their own authority), this is exactly the kind of decision that gets an ADR rather than being asserted unilaterally inside a `requirement-spec.md`.

Two things need to be pinned down before the pipeline's later stages (DDD, architecture, database design) can proceed without re-litigating this each time:

1. **Deployment topology** — one more repo/service to operate (§25-D1's stated trade-off) versus reusing an existing one.
2. **Data ownership boundary** — whether this new service is allowed to hold a local copy of any business domain data (orders, products, revenue figures) it queries from Analytics, or whether — like Admin Service's Domain Invariant #3 (`kart-admin-service/requirement-spec.md` §4, formalized platform-wide by ADR-0010) — it must never become a second owner/source of truth for data another service already owns.

The spec's own D1 reasoning already leans on the Admin Service invariant by analogy ("Folding it into Analytics would force a read-only, generic-subdomain service to also own LLM orchestration and conversation state — a different kind of complexity"); this ADR formalizes that reasoning the same way ADR-0010 formalized Admin's own scope, so the DDD Agent has a settled boundary to model against rather than an open question.

## Decision

**`kart-ai-assistant-service` is a new, independently deployed bounded context** — its own repo, its own deployable, following the platform's existing one-bounded-context-per-repo convention already applied to all 18 deployable service repos (`docs/architecture/container-diagram.md`'s closing caption: "This diagram is now complete for all 18 deployable service repos"). It becomes the 19th deployable repo. This resolves §26-Architecture-1: starting as a module inside an existing service and extracting later is rejected, because (per §25-D1) neither `kart-admin-service` nor `kart-analytics-service` is a coherent home for LLM orchestration + conversation-session ownership without distorting that service's own already-approved scope — there is no clean "temporary" placement that wouldn't itself require its own later boundary-extraction ADR, so the platform pays that cost once, now.

**Its data-ownership boundary is closed, by direct analogy to Admin Service's Domain Invariant #3:**

| It may own | It may never own |
|---|---|
| Conversation/session state (the last resolved structured intent per session, §14) | Any copy of Analytics' dashboard/read-model data (revenue, orders, product performance, etc.) |
| Its own audit log (`AuditRecord`, one per turn, §20) | Any copy of Order/Product/Inventory/User domain data |
| Nothing else | A cache or local recomputation of a number Analytics itself already returns (spec FR-003's own rule: "the assistant never caches or locally recomputes what Analytics itself returns") |

This mirrors ADR-0010's own framing exactly: `kart-ai-assistant-service` is an **orchestration/control-plane caller** of Analytics' data, not a second owner of it — the same relationship Admin Service holds toward Product/Category/Offer/Identity/Inventory's data, just read-only instead of write-through.

**Integration pattern:** exactly one synchronous downstream dependency, `kart-analytics-service`'s internal query API (spec §10.1, §21.3) — no direct calls to Order/Product/Inventory/User services, and (per ADR-0027, adopted alongside this one) no dependency the platform's data model doesn't already support. It has no asynchronous publish or consume relationships (formalized in this service's own `event-contract.md`, following the precedent `kart-admin-service/event-contract.md` and `kart-identity-service/requirement-spec.md` already set for a purely-synchronous service: state the absence explicitly, cite why, rather than leaving the section silently blank).

**It sits behind the Gateway like every other `kart-admin-web` backend call** (spec §10.4) — no bypass, consistent with ADR-0023's decision that the Gateway forwards the original client JWT unchanged (this service re-validates that same JWT rather than trusting a minted internal header).

## Consequences

- §26-Architecture-1 is closed: `kart-ai-assistant-service` is confirmed as an independent deployable, not a module-to-be-extracted-later. No further architecture-agent work needs to re-ask this question.
- `docs/architecture/system-context.md`, `container-diagram.md`, and `service-boundaries.md` must each add this service as a new participant: one new node/edge in `container-diagram.md` (sync edge to `kart-analytics-service` only), one new `## kart-ai-assistant-service` section in `service-boundaries.md`, and the actor/system-box description in `system-context.md` should note the new service count.
- The DDD Agent must model exactly two aggregates of this service's own — conversation/session and audit record — and must not introduce a `Product`, `Order`, `Revenue`, or any other aggregate that duplicates Analytics' or another service's domain data (per the ownership table above).
- The Database Design Agent designs storage only for conversation-session state and the audit log — no analytics-shaped read model of its own.
- The Event Design Agent documents zero publish/consume relationships, explicitly, per the Admin/Identity precedent, rather than inventing events this service has no need for.
- This service becomes the 19th deployable repo; any future capacity-plan or repo-count reference elsewhere in `docs/` should be updated to match when next touched (not retroactively rewritten by this ADR alone).

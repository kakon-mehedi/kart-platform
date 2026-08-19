---
doc_type: adr
status: accepted
---

# ADR-0026: Seller/Vendor Reporting Is Aspirational Business-Flows Scope, Not an Implemented Gap — Permanently Out of Scope for the Business Assistant

## Status

Accepted

## Context

`docs/requirements/genai-business-assistant-spec.md` §26-Data-2 flags a contradiction it cannot resolve on its own authority: `docs/requirements/business-flows.md` Flow 18 ("Analytics & Reporting Dashboard") names "Seller Performance Reports" as a step, and Flow 16 describes a full "Seller/Vendor Onboarding & Management" journey (Seller Registration → KYC → Business Verification → Bank Account Verification → Seller Approval → Seller Dashboard Access → Product Listing by Seller → Commission/Fee Setup → Seller Order Management → Seller Payout/Settlement → Seller Performance Rating) — yet **no Seller/Vendor bounded context, aggregate, or field exists anywhere in the platform's actual 18-service architecture**. Verified for this ADR by exhaustive grep across `docs/ddd/ubiquitous-language.md`, all 19 `docs/services/*/ddd-model.md` files, and `kart-requirements.md`'s own §2.1 service-list table: zero occurrences of "seller" or "vendor" outside `business-flows.md` itself (where it appears at lines 630, 770, 791–834, and 942). `kart-requirements.md` §2.1's 18-deployable-repo list is exhaustive and does not include a Seller/Vendor service, merged or otherwise (contrast with Offer Service, whose 3-way Coupon/Pricing/Promotion merge is explicitly noted, per ADR-0001).

This is precisely the kind of "documents two sources that disagree" situation this repo's own convention (cited in ADR-0023) reserves for an ADR rather than a silent pick by whichever agent happens to touch it next. Left unresolved, it would recur every time a future spec (this one, and any later BI/reporting work) touches Flow 18 — each one independently re-discovering the same gap and re-deciding, ad hoc, whether to build toward it.

The spec's own §26-Business-2 also asks a forward-looking design question riding on this same ruling: should `kart-ai-assistant-service`'s registry (§10.3, §12) be built with extensibility "toward" a future Seller domain, or treated as fully out of scope with no accommodation? This ADR answers both.

## Decision

**This is a genuine gap between the business-flows catalog and the implemented architecture — not something this capability, or this ADR, can retroactively make real.** Ruling:

1. **Seller/Vendor reporting is aspirational/future scope in `business-flows.md`, not a currently-implemented capability.** `business-flows.md` documents a *target* end-state business journey; it is not itself a bounded-context inventory, and per this ADR it is **not treated as authoritative evidence that a Seller/Vendor service exists or is imminent**. The authoritative service inventory remains `kart-requirements.md` §2.1.
2. **This gap is out of scope for the Kart Business Assistant, permanently — not merely deferred pending a future phase of this capability.** The Business Assistant's job (spec §1.1) is to answer questions from data Kart already collects; it is not the mechanism that should backfill a missing bounded context. If a Seller/Vendor service is ever built, exposing its performance data through the assistant is a normal, incremental registry addition at that time (the same shape as this spec's own product-performance addition, §10.3) — not something that needs to be designed for now.
3. **No speculative extensibility is reserved in `kart-ai-assistant-service`'s registry, intent schema, or endpoint-mapping table for a future Seller dimension.** This answers §26-Business-2 directly: the registry (§6 metrics, §7 dimensions, §10.3 endpoints) is scoped exactly to what's real today. Reserving a `seller`-shaped placeholder field or dimension now would be speculative design against a bounded context that may never exist in this shape, or may exist with a different aggregate/field structure than anyone can predict today — the same anti-pattern the spec's own §12/D5 already rejects for RAG ("a technology chosen because it's commonly used, not because the requirements need it," applied here to speculative schema instead of speculative technology).
4. **The reconciliation of `business-flows.md` itself (whether to mark Flow 16/18's Seller steps as future/aspirational inline, or to open a separate initiative to scope a real Seller/Vendor service) is explicitly not this ADR's call** — it belongs to whoever owns the BRD/flow-catalog (per the spec's own sign-off checklist: "BRD/flow-catalog owner reconciles Data-2"). This ADR rules only on what `kart-ai-assistant-service` and this capability's own docs must do, which is unaffected by how that separate reconciliation eventually lands.

Query type #15 (Seller/vendor performance, spec §5) remains permanently in the "Not supported — domain gap" category (FR-009's unsupported-capability response, not a clarification and not a bug) until and unless a real Seller/Vendor bounded context is built and its own architecture/DDD/API pipeline runs — at which point exposing it through the assistant is a new, ordinary registry-addition pass through this same pipeline, not a retrofit of this ADR.

## Consequences

- §26-Data-2 and §26-Business-2 are both closed: Seller/Vendor is ruled aspirational-only for this capability's purposes, and no extensibility hook is built for it.
- `kart-ai-assistant-service/requirement-spec.md`, `ddd-model.md`, and `api-contract.yaml` (Part 2 of this capability's pipeline) must not reference a Seller/Vendor entity, metric, or dimension anywhere, including as a "reserved for future use" placeholder.
- Query type #15 in the requirement-spec's supported-query-types table is documented identically to how the spec itself already frames it (§5 item 15: "Not supported — domain gap") — this ADR is the citable authority for that framing going forward, replacing the spec's own flagged open question.
- This ADR does **not** modify `business-flows.md` itself — that document's own correction (if any) is a separate, BRD-owner-level initiative and is out of scope here.
- Any future Seller/Vendor bounded-context proposal should cite this ADR when it reaches the point of asking whether the Business Assistant should expose its data — the answer, per this ADR, is: yes, via a normal incremental registry addition, following the exact precedent this capability's own product-performance read model (spec §10.3) already sets.

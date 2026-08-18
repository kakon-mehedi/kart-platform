---
doc_type: adr
status: accepted
---

# ADR-0027: `OrderConfirmed.address` Has No Confirmed Shape in `kart-order-service`'s Own Contract — Geographic Analytics Requires a Producer-Side Event-Contract Change, Not a Cheap Read-Model Addition

## Status

Accepted

## Context

`docs/requirements/genai-business-assistant-spec.md` §26-Data-1 flags an open question about whether geographic/location analytics (NG2, query type #14) is a moderate-effort future read-model addition or requires a `kart-order-service` contract change, contingent on one fact: "whether that `address` payload field is a structured object (with decomposable `city`/`region` fields, matching `kart-user-service`'s own `AddressDetail` shape) or an opaque formatted string." The spec explicitly left this unconfirmed pending someone checking `kart-order-service`'s own docs. Per this capability's own task instructions, that check was performed before ruling on §26-Data-1, since the answer determines a materially different scope for any future geographic-analytics work.

**Finding, checked directly against `kart-order-service`'s own approved docs:**

- `event-contract.md` (the only place `OrderConfirmed`'s payload is described) lists `address` as a bare key in a "Key Fields" table cell (`orderId`, `address`) — **no type, no shape, no schema is given**.
- `api-contract.yaml`'s `schemas:` block defines `Problem`, `Money`, `OrderLineItemView`, `OrderView` — **zero occurrences of `address`** anywhere in the file, structured or otherwise.
- `ddd-model.md`'s `Order` aggregate's full Value Objects list is `Money`, `OrderStatus`, `IdempotencyKey`, `OrderEventSequence` — **there is no `Address`/`AddressDetail` value object modeled on `Order` at all.**
- `database-design.md` is explicit that no address data is persisted on any Order table: *"Order has no sensitive/PII columns to classify... **no phone number, address, or credential-shaped value is stored on any of Order's own tables** (contrast with User Service's `phone`/address columns...)"* — and frames a future "denormalized shipping-address copy" as a hypothetical addition Order doesn't currently have.

**This is a third possibility the spec's own §26-Data-1 framing didn't anticipate.** The spec asked "structured object, or opaque string" — assuming the field exists in some typed form at the producer. The actual finding is that **the field is neither** in any confirmed, documented sense: it is not backed by an `Address` value object, a Postgres column, or an OpenAPI schema anywhere in `kart-order-service`'s own approved contract. Its only appearance is an untyped key name in an async event's field listing (corroborated only by consumer-side docs — `kart-notification-service/event-contract.md` and `docs/adr/0016-user-gdpr-erasure-policy.md` — that repeat the same bare `address` key with no shape, never independently confirming one). This means the event's actual current payload shape for `address` is whatever the service's implementation happens to serialize today, undocumented and unreviewed as a contract — a materially worse starting point for a downstream consumer than either of the two possibilities the spec considered.

This is exactly the kind of finding that "affects another service's own contract" per this repo's ADR policy (ADR-0010, ADR-0023) rather than something `kart-ai-assistant-service`'s own docs can resolve or work around unilaterally — hence a dedicated ADR rather than a note buried in ADR-0024 or ADR-0026.

## Decision

**Geographic analytics (NG2, query type #14) is not a cheap, additive future read-model — it requires a `kart-order-service` producer-side event-contract change first**, specifically: `kart-order-service` must define and document a proper `Address` value object on `OrderConfirmed`'s payload (recommended: reuse `kart-user-service`'s `AddressDetail` shape — `{ type, line1, line2, city, region, postalCode, countryCode, phone }` — either by direct reference or by a locally-defined equivalent with at least a decomposable `city`/`region` for the dimension this capability would need), add it to `api-contract.yaml`'s schemas if `OrderConfirmed`'s payload is ever formally modeled there, and update `database-design.md`'s PII classification accordingly (the value object would introduce Order's first genuinely sensitive column, per that doc's own stated hypothetical). Until that work happens, `analytics_raw_events.payload.address` (whatever `kart-analytics-service` has actually been ingesting under full fan-in, ADR-0004) has no reviewed contract behind it and must not be treated as a stable field to project a new read model from.

This is **not** a decision this ADR makes on `kart-order-service`'s behalf — it does not specify the value object's final field list, migration plan, or timeline; that remains `kart-order-service`'s own team's call, the same boundary ADR-0010 respected when it fixed Admin's integration *pattern* without dictating each callee's missing endpoint. This ADR fixes only the **scoping conclusion** that follows from the finding above.

**Consequently, NG2 / §5 query type #14 (geographic analysis) remains "Not supported — data gap" in `kart-ai-assistant-service`'s own registry for the full duration of this capability's build (Phases 1–7, spec §24) — with no partial/best-effort attempt to derive city/region from the current undocumented `address` payload.** Attempting to parse an unreviewed, unversioned field would violate the spec's own §16 hallucination-prevention posture by a different route: silently trusting an unmodeled producer field instead of a stated business metric.

## Consequences

- §26-Data-1 is closed with a more precise (and less favorable) answer than the spec's own framing anticipated: geographic analytics is gated on a `kart-order-service` contract change, not a same-team Analytics read-model addition — this should be flagged to whoever owns `kart-order-service`'s roadmap, not silently absorbed as "someday, cheaply" by the Analytics or AI Assistant teams.
- `kart-ai-assistant-service/requirement-spec.md` and `ddd-model.md` (Part 2 of this capability) must cite this ADR (not just spec §26-Data-1) when documenting why geographic analysis is out of scope, since this ADR is the more specific, checked finding.
- `kart-analytics-service`'s own docs are unaffected by this ADR — no new Analytics read model is being scoped here; this ADR only forecloses treating one as "cheap" prematurely.
- `kart-order-service`'s own docs (`event-contract.md`, `ddd-model.md`, `api-contract.yaml`, `database-design.md`) are not modified by this ADR — the actual contract change is that team's own future work, tracked as a dependency this ADR surfaces, not one it discharges.
- Any future ticket to build geographic analytics must list a `kart-order-service` contract-change ticket as a hard prerequisite, not a parallelizable nice-to-have.

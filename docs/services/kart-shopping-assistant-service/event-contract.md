---
doc_type: event-contract
service: kart-shopping-assistant-service
status: approved
generated_by: event-design-agent
source:
  - docs/services/kart-shopping-assistant-service/requirement-spec.md (status: approved — §9 item 4)
  - docs/services/kart-shopping-assistant-service/architecture.md (status: approved — Dependencies table, Distributed-Monolith Risk)
  - docs/services/kart-shopping-assistant-service/ddd-model.md (status: approved — aggregate boundaries: ShoppingAssistantSession, ShoppingAssistantAuditRecord, ShoppingAssistantIdempotencyKey)
  - docs/services/kart-shopping-assistant-service/design-decisions.md (status: approved — RabbitMQ Manifest-Declaration Equivalent decision, "documented, not built")
  - docs/adr/0028-shopping-assistant-service-scope-and-integration.md
  - docs/adr/0030-shopping-assistant-python-stack-exception.md
  - docs/services/kart-ai-assistant-service/event-contract.md (structural precedent — the platform's other zero-async-edges service; this document mirrors its convention for documenting an intentional absence rather than leaving the section silently blank)
  - docs/services/kart-ai-assistant-service/message-bus-manifest.json (structural precedent for the corresponding empty-topology manifest)
  - docs/standards/event-standards.md (agent-reusables)
  - docs/standards/kart-conventions.md
---

# Event Contract: kart-shopping-assistant-service

## Posture: Zero Published Events, Zero Consumed Events

**This service participates in no asynchronous messaging of any kind — no RabbitMQ exchange, no RabbitMQ queue, no Kafka topic, in either direction.** This is the platform's second service in this exact shape, not a novel case:

- `kart-ai-assistant-service/event-contract.md` is the precedent this document mirrors directly: "a purely synchronous orchestrator with no event-bus edge at all... not 'a consumer with no publishes' or 'a publisher with no consumes'." Both statements are true here as well, generalized from that service's single downstream dependency to this service's nine.
- `kart-admin-service/event-contract.md` publishes exactly one event (`AdminActionPerformed`) and consumes none — this service has neither side populated, the stricter case, exactly as `kart-ai-assistant-service` already established.
- `kart-identity-service/requirement-spec.md` names an intentionally empty *Consumes* list, closed by ADR rather than left an unexplained gap — the convention this document (and its precedent) follows for stating an absence explicitly rather than leaving a section blank.

This section states and justifies that posture explicitly, per this repo's own convention — restated by `kart-ai-assistant-service/event-contract.md`, itself citing the `kart-admin-service`/`kart-identity-service` precedent, and by BRD §9's "state the absence explicitly, cite why" rule (ADR-0024/ADR-0028 both restate it) — rather than leaving this file terse, blank, or skipped.

## Why: Confirmed Against ADR-0028, requirement-spec.md §9 item 4, and architecture.md's Dependencies Table

Before writing this file, the case for a candidate event was considered explicitly and rejected — recorded here rather than silently omitted, per this agent's own escalation duty.

**Candidate considered: `ShoppingAssistantTurnAudited`, a mirror of `kart-admin-service`'s own `AdminActionPerformed` / the same candidate `kart-ai-assistant-service` rejected as `AssistantQueryAudited`.** This service writes exactly one durable, append-only `ShoppingAssistantAuditRecord` per turn (`ddd-model.md`), the same shape that motivated Admin's one published event and that `kart-ai-assistant-service` considered and rejected for its own, structurally lighter, audit record.

**Rejected, for the same three independent reasons `kart-ai-assistant-service/event-contract.md` already established, applied here:**

1. **requirement-spec.md §9 item 4 already closes this, as a resolved decision, not an open question this stage could fill either way:** "This service consumes **zero** platform events for v1, mirroring ADR-0024's own zero-async-edges ruling for `kart-ai-assistant-service`." `architecture.md`'s Dependencies table restates both directions as closed rows ("Outbound (published) — none," "Inbound (consumed) — none") and its own closing statement: "Any future proposal to add a tenth Kart-service dependency or an event-bus edge is a scope change requiring its own ADR... not an incremental addition this document, the DDD Agent, or the API Design Agent has standing authority to make." ADR-0028's Consequences section is equally explicit: "The Event Design Agent documents zero publish/consume relationships for v1, explicitly, per the same precedent ADR-0024 set." Adding an event here, absent a new ADR reopening that ruling, would be exactly the unauthorized scope change these documents warn against.
2. **No stated consumer need exists.** No source document for this service — `requirement-spec.md`, `ddd-model.md`, `architecture.md`, `design-decisions.md`, or ADR-0028/0029/0030 — names or implies a downstream consumer (Analytics or otherwise) that needs a per-turn assistant-audit event asynchronously. `ShoppingAssistantAuditRecord` is this service's own durable, directly-queryable audit trail (`ddd-model.md`: "queried by `turnId`, by `conversationId`... or by `userId`/date range for compliance audit" — the same query shape `kart-ai-assistant-service`'s own `AuditRecord` and `kart-admin-service`'s own `admin_actions` table both already use). Publishing an event here would be additive fan-out for a consumer that does not yet exist.
3. **This service is, by every upstream document's own words, structurally outside the platform's event mesh — not merely quiet within it, for now.** Every one of this service's nine downstream calls (`kart-order-service`, `kart-cart-service`, `kart-offer-service`, `kart-search-service`, `kart-product-service`, `kart-recommendation-service`, `kart-user-service`, `kart-delivery-tracking-service`, `kart-wishlist-service`) is request/response, sync, per-edge-circuit-broken (`architecture.md`'s Dependencies table, Sync Fan-Out Resilience section) — never an event publish or subscribe. Several of those nine peers do themselves publish/consume events *with each other* (e.g. Order/Payment/Inventory/Shipping's own Saga-participant messaging, `kart-order-service/architecture.md`) — but from this service's own vantage point, none of that is visible or relevant: this service calls each of the nine synchronously and never touches the exchange/queue topology any of them may separately own. `kart-inventory-service`'s own already-approved architecture confirms the one two-hop chain this service does inherit (`SA → kart-order-service → kart-inventory-service`'s saga-step-1 reserve call) terminates as a synchronous call at both hops and never surfaces to this service as an event.

**Conclusion: zero events, both directions, confirmed — not a default reached by omission.** If a future need for any downstream/upstream consumer to receive shopping-assistant data asynchronously is identified, that is a new integration decision requiring its own ADR (per requirement-spec.md §9 item 4, architecture.md's closing statement, and ADR-0028's Consequences section), not a retroactive edit to this file.

## Documented-but-Not-Built: A Future Proactive Capability Remains Explicitly Out of Scope, Not Designed Here

requirement-spec.md §9 item 4 carries forward, unresolved and undesigned, the same Architecture-5-equivalent observation `kart-ai-assistant-service`'s own scope closed for its sibling: "A future proactive capability (e.g. consuming `OrderDelivered` to prompt 'how was your order?') is explicit future/non-goal scope, not decided here — adding it later is a scope change for that future pass, not something this document blocks on or designs a placeholder for now (mirrors ADR-0026's 'no speculative extensibility reserved' reasoning, applied here to event edges rather than schema fields)." architecture.md's own closing statement (see above) reaches the identical conclusion independently.

This document does not design that event, its schema, its routing key, or any retry/DLQ policy for it — doing so would itself be the "speculative extensibility" both upstream documents decline to reserve. The door is noted as closed for this pass, not welded shut: a future decision to add `OrderDelivered` (or any other) consumption is a new scope change requiring its own ADR (mirroring ADR-0024/ADR-0028's own non-reopening posture), at which point this Event Design Agent stage would be re-run against that new ADR, `design-decisions.md`'s already-documented (not built) `kart_shared.messaging`/`aio-pika` pattern, and the platform's existing JSON manifest schema (`kart-requirements.md` §8/§9) — the same equivalence path ADR-0030's guarantee table row 2 already lays out for exactly this contingency.

## Event Table

| Event | Routing Key | Published/Consumed | Key Fields | Retry | DLQ | Criticality Justification |
|---|---|---|---|---|---|---|
| — | — | — | — | — | — | **None.** This service publishes no events. |

No consumed events either. requirement-spec.md §9 item 4, architecture.md's Dependencies table, and ADR-0028's Consequences section all confirm this as intentional, not a gap: this service's only integration edges are the nine synchronous, per-edge-circuit-broken calls named above plus the external Model Gateway/LLM Provider call (architecture.md's Dependencies table); its three local aggregates (`ShoppingAssistantSession`, `ShoppingAssistantAuditRecord`, `ShoppingAssistantIdempotencyKey`, per `ddd-model.md`) are populated entirely by synchronous, in-process request handling, never by an inbound event.

## Naming-Convention Compliance

Not applicable — there is no event name to check against the `<Entity><PastTenseVerb>` convention (`event-standards.md`), since no event exists.

## Retry / DLQ Policy

Not applicable — there is no consumer queue to assign a DLQ to, and no publish path to assign a retry count to. Per `event-standards.md`'s "every consumer queue gets its own DLQ" rule and `kart-conventions.md`'s exchange-ownership convention (`<service>.exchange`, `<service>.dlx`, per-service retry ladder), this service owns none of these RabbitMQ objects, because it has no queue and no exchange to protect. See `message-bus-manifest.json` for the corresponding empty topology declaration.

Note the one Python-specific nuance this service's stack exception introduces (ADR-0030; design-decisions.md's "RabbitMQ Manifest-Declaration Equivalent" decision): even if this service *did* need to declare topology, the mechanism would not be the platform's existing .NET startup hook — it would be a `kart_shared.messaging` startup hook using `aio-pika`, reading the same, already-language-agnostic JSON manifest schema (`kart-requirements.md` §8/§9) every other service already declares against. That mechanism is explicitly **documented, not built** (design-decisions.md), because it is not needed for v1's zero-event topology. This event contract does not build it, name numeric retry/DLQ values for it, or otherwise treat it as anything other than a future contingency.

## Relationship to This Service's Own Audit Trail

`ShoppingAssistantAuditRecord` (`ddd-model.md`) remains this service's full, durable, queryable record of every turn — its absence from this event contract is not a loss of observability. It is queried directly (by `turnId`, `conversationId`, or `userId`/date range for compliance audit) the same way `kart-ai-assistant-service`'s own `AuditRecord` and `kart-admin-service`'s own `admin_actions` table are each queried directly, independent of any event-bus delivery. This service simply never reaches the point of needing a fan-out copy of that fact for another service, because no other service is named as needing one.

## Sign-off

- [x] Reviewed by: Automated architecture pipeline — autonomous completion authorized by project owner
- [x] Approved — zero published events, zero consumed events, confirmed against ADR-0028, requirement-spec.md §9 item 4, and architecture.md's Dependencies table; no event added, no scope change introduced

---
doc_type: event-contract
service: kart-ai-assistant-service
status: approved
generated_by: event-design-agent
source:
  - docs/services/kart-ai-assistant-service/requirement-spec.md
  - docs/services/kart-ai-assistant-service/ddd-model.md
  - docs/services/kart-ai-assistant-service/architecture.md
  - docs/adr/0024-ai-assistant-service-scope-and-integration.md
  - docs/standards/event-standards.md (agent-reusables)
  - docs/standards/kart-conventions.md
  - docs/services/kart-admin-service/event-contract.md (structural precedent — one published event, zero consumed)
  - docs/services/kart-identity-service/requirement-spec.md (precedent for an intentionally empty Consumes list, closed by ADR rather than left as an unexplained gap)
---

# Event Contract: kart-ai-assistant-service

## Posture: Zero Published Events, Zero Consumed Events

**This service participates in no asynchronous messaging of any kind — no RabbitMQ exchange, no RabbitMQ queue, no Kafka topic, in either direction.** This is a stricter case than every other precedent already in this repo:

- `kart-admin-service/event-contract.md` publishes exactly one event (`AdminActionPerformed`) and consumes none.
- `kart-identity-service/requirement-spec.md` consumes exactly one event (`UserDataErased`) and publishes several (`UserRegistered`, `SessionCreated`, `UserAccountUpdated`), with its historically-empty Consumes list explained (not silently blank) as a synchronous-call substitution.
- `kart-ai-assistant-service` has **neither** side populated. It is not "a consumer with no publishes" or "a publisher with no consumes" — it is a purely synchronous orchestrator with no event-bus edge at all, a shape closer to `kart-analytics-service`'s zero-synchronous-coupling posture mirrored onto the async axis instead.

This section states and justifies that posture explicitly, per this repo's own convention (`kart-admin-service`/`kart-identity-service` precedent, cited in `requirement-spec.md` §5 and `architecture.md`'s Dependencies table) rather than leaving this file terse, blank, or skipped.

## Why: Confirmed Against ADR-0024, requirement-spec.md, and architecture.md

Before writing this file, the case for a candidate event was considered explicitly and rejected — recorded here rather than silently omitted, per this agent's own escalation duty.

**Candidate considered: `AssistantQueryAudited`, a mirror of `kart-admin-service`'s own `AdminActionPerformed`.** Admin publishes `AdminActionPerformed` as a fire-and-forget fan-out copy of its own durable `admin_actions` audit table, primarily so `kart-analytics-service` can build an "Admin audit/compliance dashboard" from it without querying Admin's database directly. By direct structural analogy, `kart-ai-assistant-service` also writes exactly one durable, append-only audit record per turn (`AuditRecord`, `ddd-model.md`) — the same shape that motivated Admin's one published event.

**Rejected, for three independent reasons, each sufficient on its own:**

1. **ADR-0024 already closes this, in its own Consequences section, in this exact document's favor:** "The Event Design Agent documents zero publish/consume relationships, explicitly, per the Admin/Identity precedent, rather than inventing events this service has no need for." This is not a gap ADR-0024 left for this stage to fill in one direction or the other — it is a direct instruction against adding one.
2. **`requirement-spec.md` §5's API Surface table already states, as a closed, non-open item:** "Published events: **None** ... stated explicitly, per ADR-0024 and the `kart-admin-service`/`kart-identity-service` precedent for purely-synchronous services, not left silently blank" and "Consumed events: **None** — same rationale." §4's Domain Invariants restate this as one of the service's own binding invariants: "Any future proposal to add a second downstream dependency or an event edge is a scope change requiring its own ADR, not an incremental addition this spec's own authority covers." Adding `AssistantQueryAudited` here, absent that ADR, would be exactly the unauthorized scope change §4 warns against.
3. **No stated consumer need exists, unlike Admin's case.** Admin's `AdminActionPerformed` has a named, concrete consumer purpose (Analytics' Admin audit/compliance dashboard, per that event's own Criticality Justification). No source document for this service — `requirement-spec.md`, `ddd-model.md`, `architecture.md`, or any of ADR-0024/0025/0026/0027 — names or implies an Analytics (or any other) consumer need for a per-turn assistant-audit event. `AuditRecord` is queried directly and synchronously by this service's own compliance/audit read path (`ddd-model.md`'s "queried by `turnId`, by `conversationId`... or by `userId`/date range for compliance audit"), the same pattern `kart-admin-service`'s own `GET /admin/actions` uses for its *own* durable table — publishing an event is additive fan-out for a downstream consumer that does not yet exist, not a requirement this spec's own §2/§5 API surface calls for. Inventing one now would be building ahead of a stated need, which `ddd-model.md`'s own Modeling Decision 3 explicitly declines to do ("A future genuinely async internal step... would be the trigger to revisit this, not something anticipated here").

**Conclusion: zero events, both directions, confirmed — not a default reached by omission.** If a future need for Analytics (or any consumer) to receive per-turn assistant-audit data asynchronously is identified, that is a new integration decision requiring its own ADR (per `requirement-spec.md` §4's own invariant and ADR-0024's Consequences), not a retroactive edit to this file.

## Event Table

| Event | Routing Key | Published/Consumed | Key Fields | Retry | DLQ | Criticality Justification |
|---|---|---|---|---|---|---|
| — | — | — | — | — | — | **None.** This service publishes no events. |

No consumed events either. `requirement-spec.md` §5 and `architecture.md`'s Dependencies table both confirm this as intentional, closed by ADR-0024 — not a gap: this service's only integration edge is the single synchronous call to `kart-analytics-service`'s internal query API (OAuth2 Client Credentials, `analytics.dashboards.read`); its two local aggregates (`ConversationSession`, `AuditRecord`, per `ddd-model.md`) are populated entirely by synchronous, in-process request handling, never by an inbound event.

## Naming-Convention Compliance

Not applicable — there is no event name to check against the `<Entity><PastTenseVerb>` convention (`event-standards.md`), since no event exists.

## Retry / DLQ Policy

Not applicable — there is no consumer queue to assign a DLQ to, and no publish path to assign a retry count to. Per `event-standards.md`'s "every consumer queue gets its own DLQ" rule and `kart-conventions.md`'s exchange-ownership convention (`<service>.exchange`, `<service>.dlx`, per-service retry ladder), this service owns none of these RabbitMQ objects, because it has no queue and no exchange to protect. See `message-bus-manifest.json` for the corresponding empty topology declaration.

## Relationship to This Service's Own Audit Trail

`AuditRecord` (`ddd-model.md`) remains this service's full, durable, queryable record of every turn — its absence from this event contract is not a loss of observability. It is queried directly (by `turnId`, `conversationId`, or `userId`/date range) the same way `kart-admin-service`'s own `admin_actions` table is queried directly for compliance review even independent of whether `AdminActionPerformed` delivery succeeds (see that service's own Retry-Tier Justification: "the durable audit record does not depend on this RabbitMQ delivery succeeding"). This service simply never reaches the point of needing a fan-out copy of that fact for another service, because no other service is named as needing one.

## Sign-off

- [x] Reviewed by: Automated architecture pipeline — autonomous completion authorized by project owner
- [x] Approved — zero published events, zero consumed events, confirmed against ADR-0024, `requirement-spec.md` §4/§5, and `architecture.md`'s Dependencies table; no event added, no scope change introduced

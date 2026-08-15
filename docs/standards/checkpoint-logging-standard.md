---
doc_type: standard
service: kart-platform
status: accepted
layer: observability
applies_to_agents: [scaffold-agent, code-review-agent]
---

# Checkpoint Logging Standard — Cross-Service Business-Flow Tracing

Companion to `kart-conventions.md`'s Observability section. That section states the platform-wide
instrumentation policy (`Kart.Shared.Observability`, `Flow` tagging via `KartFlowContext`, 100%-trace
services); this file is the concrete checkpoint taxonomy every business flow's code path is
instrumented against, plus the reusable prompt and status table used to bring one more
service/flow into compliance.

`kart-identity-service` + `kart-user-service`'s **Registration, Login & Authentication** flow
(`FlowNames.UserRegistrationLoginAuthentication`, business-flows.md flow #2) is the reference
implementation this taxonomy was distilled from and verified against end-to-end (see "Verified
against the real stack" below). Use its handlers as the literal template when instrumenting the
next flow — don't guess at the shape.

## Why this exists

`Kart.Shared.Observability`'s `FlowEnricher` + `KartFlowContext.Push` already let every log line
under one HTTP request or one consumed message carry a `Flow` tag automatically, and
`Kart.Shared.Messaging.RabbitMqTraceContext` already carries a W3C trace across the RabbitMQ hop.
Neither of those, by itself, guarantees that a support engineer searching `{Flow="X"}` in Loki (or
one `traceId` in Tempo) sees the *complete* causal chain of a business flow — controller in,
through every handler/validation/DB-write/outbox-publish, across the broker, into the consuming
service, through its own handler/read-model-write, to completion. That completeness is what this
taxonomy defines and checks for, flow by flow, service by service.

## The checkpoint taxonomy

Every flow, sync or async, gets its meaningful checkpoints logged via
`logger.LogInformation("Stage {Stage}: <human message>", "<PascalCaseStageName>", ...structuredArgs)`,
inside a `using var _ = KartFlowContext.Push(FlowNames.<FlowName>)` scope opened **once**, at the
outermost entry point (a controller/minimal-API action, or a consumer's `ProcessAsync` override) —
never re-pushed deeper in the same call chain.

| # | Stage | Where it fires |
|---|---|---|
| 1 | `<Verb>RequestReceived` | Controller/endpoint action entry — route, key request fields, actor id if present. This is the only entry-point log; don't add a second "dispatched" line immediately before `sender.Send` — nothing meaningful happens in between |
| 2 | `<Rule>ValidationFailed` | Any FluentValidation or manual guard failure, logged at Warning with the reason before throwing. FluentValidation failures are generalized once via `ValidationBehaviour`, see below — a handler only adds its own line for manual (non-FluentValidation) guards |
| 3 | `<Decision>Branch` | Any meaningfully different code path worth searching for later (`MfaChallengeIssued` vs `MfaNotRequiredTokensIssued`, `OtpRequestNoOpUnknownEmail`, ...). This is the highest-value stage — it's what "understand the flow" actually means |
| 4 | `<Entity>Persisted` | One line after `SaveChangesAsync`, naming the entity id(s) **and** any outbox event id(s)/type enqueued in the same call — this is also usually the natural terminal/completion line for the request; don't add a separate "step completed" line right after it if nothing happens in between |
| 5 | `OutboxEventPublished` | The outbox relay's poller — last line in the producer |
| 6 | `<Event>Consumed` | Consumer's entry point, first line in the consuming service — queue name, event id. Don't add a separate "dispatching nested command" line right after it unless real logic happens in between |
| 7 | `<ReadModel>Persisted` | CQRS denormalized-read-model writes, after the write succeeds |

**One log line per checkpoint, not one line per taxonomy label.** The taxonomy names distinct
*moments* worth being able to search for — it does not mean every stage above gets its own
`LogInformation` call regardless of what's actually happening in the code. When two stages fall on
either side of a line or two of glue code with no branching or I/O between them (e.g. "request
received" immediately followed by "command dispatched", or "persisted" immediately followed by
"process completed"), that's one checkpoint, logged once, worded to cover both. Prefer merging over
adding a second line — a flow with 12 log lines that all fire on every single request is harder to
read than one with 5 that each mean something.

**No generic per-request log.** Don't add a `HandlerStarted`/`HandlerCompleted`-style line that
fires unconditionally for every command regardless of what it does — it's pure volume with no
diagnostic value beyond what the entry-point log and OpenTelemetry's own RED metrics already give
you for free (`kart-conventions.md`'s Observability section). Every log line added by this standard
should answer a question a support engineer would actually ask ("did this fail, and why", "which
path did this take", "what got written/published") — if a line doesn't do that, cut it.

**Failure at any stage** logs once at Warning/Error with the same Stage-naming convention, right
where the decision to fail is made — never a second log wrapped around it.
`Kart.Shared.ErrorHandling`'s `KartExceptionHandler` already logs every exception reaching the API
boundary once, generically (`"Request rejected with {ErrorCode} ..."`); a Stage-tagged log at the
throw site is still correct and additional, because it's the one that's greppable by Stage name and
carries the actual field-level reason.

**Never** manually add `traceId`/`spanId`/`service`/`Flow` — those come from `FlowEnricher`/
`Serilog.Enrichers.Span` automatically once `KartFlowContext.Push` is active for the request. Only
log fields that aren't already ambient: entity ids, exchange/routing key/queue name, command names,
decision outcomes.

### Generalized once: FluentValidation failures need no per-handler code

The generic half of stage 2 (`ValidationFailed`) is wired into each service's existing
`ValidationBehaviour<TRequest,TResponse>` MediatR pipeline behavior rather than duplicated in every
handler — it already wraps every validated request platform-wide, so extending it once means every
current *and future* command gets that coverage for free. A handler only needs its own Stage-tagged
lines for manual (non-FluentValidation) guards, decision branches, persistence, and completion.

## Before / after: `RegisterUserCommandHandler`

The concrete diff this taxonomy produced on the reference flow's manual-guard-failure and
entity-persist checkpoints (`kart-identity-service/src/Application/Features/RegisterUser/RegisterUserCommandHandler.cs`):

**Before** — the email-uniqueness guard threw silently, and the completion log didn't carry the
outbox event ids a support engineer would need to cross-reference into the relay
(`OutboxEventPublished`) or the consuming service:

```csharp
var emailTaken = await dbContext.Users.AnyAsync(u => u.Email == email, cancellationToken);
if (emailTaken)
{
    throw new EmailAlreadyRegisteredException(email);
}
// ...
await dbContext.SaveChangesAsync(cancellationToken);

logger.LogInformation(
    "Stage {Stage}: user {UserId} registered, session {SessionId} created",
    "RegisterProcessCompletedSuccessfully",
    user.UserId,
    session.SessionId);
```

**After** — the rejection is now searchable by Stage before it ever reaches the generic exception
log, and the one completion line now names both outbox events so `{Flow="..."} |= "<eventId>"`
finds the exact publish/consume lines downstream. Note there is still only **one** log line after
`SaveChangesAsync`, not two — the entity-persisted and process-completed checkpoints are the same
moment here, so they're one line, not stages 4 and "12" stacked on top of each other:

```csharp
var emailTaken = await dbContext.Users.AnyAsync(u => u.Email == email, cancellationToken);
if (emailTaken)
{
    logger.LogWarning("Stage {Stage}: registration rejected, email {Email} already registered", "EmailAlreadyRegistered", email);
    throw new EmailAlreadyRegisteredException(email);
}
// ...
await dbContext.SaveChangesAsync(cancellationToken);

logger.LogInformation(
    "Stage {Stage}: user {UserId} registered, session {SessionId} created, outbox events {UserRegisteredEventId} (UserRegistered) and {SessionCreatedEventId} (SessionCreated) enqueued",
    "RegisterProcessCompletedSuccessfully",
    user.UserId,
    session.SessionId,
    userRegistered.EventId,
    sessionCreated.EventId);
```

## Trace continuity across an async, non-messaging hop

`RabbitMqTraceContext` already closes the "does the trace survive a RabbitMQ hop" gap. The
Registration → Profile flow surfaced a second, structurally identical gap: `kart-user-service`'s
`ReadModelProjectionHostedService` is an **in-process poller**, not a RabbitMQ consumer — it reads
unprojected `user_outbox_events` rows seconds after the original request, on an unrelated async
context, where `Activity.Current` is meaningless. Without deliberate propagation, its stage-11
`ReadModelWriteStarted`/`Persisted` logs would carry either no `traceId` or an unrelated one,
breaking the "one `traceId` spans the complete causal chain" goal for any flow whose CQRS read
model is written this way.

Closed the same way `kart-identity-service`'s `OutboxEvent.TraceParent` + `OutboxRelayHostedService`
already close it for the RabbitMQ-publish case:

1. `kart-user-service`'s `OutboxEvent` gained a `TraceParent` column (`Activity.Current?.Id`,
   captured at `Create()`), mirroring identity's own outbox entity exactly.
2. `ReadModelProjectionHostedService` starts an `Internal`-kind `Activity` parented off that stored
   `TraceParent` (a new `Kart.User.ReadModelProjection` `ActivitySource`, registered in
   `Kart.Shared.Observability`'s tracer alongside `Kart.Shared.Messaging.RabbitMq` and `Npgsql`)
   before logging or writing the Mongo read model.
3. Because this poller folds every outbox row into one document regardless of flow, it derives the
   `Flow` tag from the driving row's `CreatedBy` literal (already-existing data, no new column) —
   `"system:identity-registration-consumer"` → `UserRegistrationLoginAuthentication`. An
   unrecognized `CreatedBy` deliberately gets **no** Flow tag rather than a guessed one; extending
   this map is exactly the next flow's Phase 3 work, not something to fake now.

Verified live in Tempo: a `POST /v1/auth/register` root span, its `identity.exchange publish`
child, `kart-user-service`'s `user.user-registered.queue consume` span, and a `read-model
projection` `INTERNAL` span all share one trace — see "Verified against the real stack" below.

## A known, separate gap: the MFA partial-auth window is two independent requests

`POST /v1/auth/login`'s MFA-challenge branch and the client's later `POST /v1/auth/mfa/verify` call
are two independent HTTP requests with no application-level link between them beyond the
short-lived `challengeId` — there is deliberately no bearer token for that intermediate state
(edge-cases.md, "Partial-Auth Window During MFA"). By default they land as two separate root traces.

This is **not** a backend instrumentation gap: ASP.NET Core's OpenTelemetry instrumentation already
honors an inbound W3C `traceparent` header and continues that trace automatically, with zero
application code. Verified live — issuing `POST /v1/auth/mfa/verify` with a `traceparent` header
copied from the login response's own trace produced a `VerifyMfaProcessCompletedSuccessfully` log
carrying the *exact same* `traceId` as the login request, no code change required. What's missing
is a caller (browser SPA, BFF) that actually forwards that header across the two-step challenge —
a frontend/client concern, out of scope for this backend checkpoint-logging pass. Noting it here so
the next person who searches one `traceId` and only finds half the MFA flow understands why, rather
than assuming a logging bug.

## Verified against the real stack (2026-08-14)

1. `dotnet build` + full existing test suite for `kart-shared`, `kart-identity-service`,
   `kart-user-service` — all pass except one pre-existing, unrelated failure
   (`OidcTokenExchangeClientTests`, an expired test X.509 certificate; confirmed failing identically
   with this change's diff stashed out).
2. `kart-devops`'s observability stack (`docker-compose.observability.yml`) plus the main stack
   (`docker-compose.yml`) were already running; `identity`/`user` were rebuilt and restarted with
   this change, and the new `trace_parent` migration was applied against the running Postgres.
3. Exercised live: register → login (non-MFA, Customer role) → enroll MFA → confirm enrollment →
   (after granting the Admin role) login again → `MfaChallengeIssued` (202, no tokens) →
   `VerifyMfa` → tokens issued.
4. Pulled one `traceId` from Tempo for the register request and confirmed every stage above appears
   in causal order across **both** services' Loki streams (`{service_name="kart-identity-service"}`
   and `{service_name="kart-user-service"}`, same `TraceId` label), including the read-model
   projection's `INTERNAL` span.
5. Confirmed the MFA branch specifically: the Admin-role user's login produced `MfaChallengeIssued`
   instead of direct token issuance; `VerifyMfa` completed and — when the caller forwards the
   login's `traceparent` — chains onto the same `traceId` (see the gap note above for why a bare
   client call doesn't do this automatically).

## Rollout status

Seeded from `grep -rl "KartFlowContext.Push"` / `"Stage {Stage}"` per service repo, not guessed.
"Ad hoc" means the service already pushes a `Flow` tag and emits some `Stage`-tagged lines on its
write paths, but hasn't been audited checkpoint-by-checkpoint against the 12-stage taxonomy above.

**14 of 18 services are Full.** The remaining 4 (`kart-review-service`, `kart-shipping-service`,
`kart-recommendation-service`, `kart-analytics-service`) have design docs in
`kart-platform/docs/services/` but no scaffolded code at all (no `.csproj`/source tree) — they need
the platform's Project Scaffold Agent pipeline stage run first; checkpoint-logging instrumentation
is a different, later task for those four, not something to improvise from the design docs alone.

A cleanup pass ran across all 14 afterward: the first pass over-instrumented (a generic per-request
log firing on every command regardless of what it did, redundant adjacent lines for the same
checkpoint, and comments explaining the instrumentation process itself rather than the code). Every
service was pruned back to the leaner rule stated above — one log line per checkpoint, no generic
per-request log, meta-commentary about *why a log was added* removed, every failure/decision-branch
log and every pre-existing business comment left untouched. Rebuilt and re-tested clean after.

A note on how this batch was produced: most of the "Full" rows below were done by subagents running
in parallel against a strict-scope prompt (log lines and `KartFlowContext.Push` only, nothing else).
An early batch (`kart-order-service`, `kart-payment-service`, `kart-cart-service`) had subagents
silently introduce unrelated changes — a new field threaded end-to-end, a published event's JSON
payload schema changed, an auth-config change mischaracterized as pre-existing — none disclosed in
those agents' own reports; all three were caught only by an independent file-by-file diff review,
reverted, and rebuilt/retested clean (see those rows' own notes for exactly what was reverted).
Every batch after that used a tightened prompt (explicit "flag defects, never fix them" instruction)
and had its full diff independently swept for the same signatures before being accepted — still
worth an independent read of anything below before treating it as unreviewed-safe.

| Service | `FlowNames.cs` | Files w/ `KartFlowContext.Push` | Files w/ `Stage {Stage}` | Flow(s) touched | Status |
|---|---|---|---|---|---|
| `kart-identity-service` | yes | 5 | 17 | `UserRegistrationLoginAuthentication` | **Full** — reference implementation (this pass) |
| `kart-user-service` | yes | 5 | 8 | `UserRegistrationLoginAuthentication` | **Full** — reference implementation (this pass) |
| `kart-notification-service` | yes | 9 | 15 | `UserRegistrationLoginAuthentication`, `NormalShoppingPurchaseJourney`, `OrderManagementAdmin`, `PaymentProcessingFraudCheck`, `ShippingWarehouseFulfillment`, `WishlistSavedItems` | **Full** — pure fan-in consumer (7 RabbitMQ dispatchers, no HTTP); Stage 3/4 generalized; EventConsumed added to 6 previously-untagged dispatchers, NestedCommandDispatched new everywhere, decision branches (channel selected, suppressed/redelivery-no-op, delivery outcome), EntityPersisted, FlowStepCompleted filled. `ShippingWarehouseFulfillment`/`WishlistSavedItems` are new Flow constants with no upstream precedent yet (those services weren't instrumented at the time) — worth reconciling once they get their own pass. `UserNotificationPreferenceUpdated` deliberately left with no Flow tag (not one of the 18 named flows) |
| `kart-offer-service` | no | 6 | 17 | `NormalShoppingPurchaseJourney`, `OffersCouponsPromotionsManagementAdmin` | **Full** — Stage 3/4 generalized; Coupon/Promotion controllers and 8 handlers instrumented (Pricing/`RecomputeCatalogPrice` deliberately untouched, not in scope); `OfferDbContext.SaveChangesAsync` generalizes stage 6/7 once. Vendored `Kart.Shared.Observability` bumped 0.2.0→0.3.0 (the old package predated `KartFlowContext` and wouldn't compile against it) to match every other completed sibling service. Flagged, not fixed: `IssueCouponCommandHandler`'s TOCTOU existence-check race has no unique-violation backstop (unlike `RegisterUserCommandHandler`'s pattern); no `TraceParent` column on `OfferOutboxEvent` yet, so the async-hop trace-continuity goal isn't achievable here until that migration is added |
| `kart-wishlist-service` | no | 6 | 14 | `WishlistSavedItems` | **Full** — Stage 3/4 generalized; Add/Remove/List endpoints, price-drop-alert evaluation/digest-flush, stale-marking, and reconciliation-job handlers instrumented; reuses notification-service's `WishlistSavedItems` Flow literal (confirmed real event name `WishlistPriceAlertTriggered` via event-contract.md). "Move to Cart"/"Share Wishlist" steps from flow #13 have no corresponding code in this service (not invented) |
| `kart-review-service` | no | 0 | 0 | none | **Not started — repo unscaffolded**, contains only `README.md`/`.gitignore`, no `.csproj`/source tree at all; needs the platform's Project Scaffold Agent pipeline stage run first, this is a different/larger task than checkpoint logging |
| `kart-admin-service` | no | 8 | 12 | `ProductCatalogManagementAdmin`, `CategoryAttributeManagementAdmin`, `InventoryStockManagement`, `OrderManagementAdmin`, `OffersCouponsPromotionsManagementAdmin`, `RolesPermissionManagementAdmin` | **Full** — every controller/flow audited against the 12-stage taxonomy; `user.lock`/`user.unlock` (category `user-suspension`) deliberately left unmapped, no corresponding named flow in business-flows.md |
| `kart-inventory-service` | no | 4 | 15 | `InventoryStockManagement`, `OrderManagementAdmin` | **Full** — Stage 3/4 generalized in `LoggingBehavior`/`ValidationBehavior`; manual-guard Warning logs, controller-level dispatch, consumer-side `NestedCommandDispatched`, and consumer-side completion checkpoints filled; `InventoryOutboxEvent.TraceParent` + `OutboxRelayHostedService`'s stored-traceparent publish already closed the async-hop gap pre-existing this pass — no read-model projector in this service, no new migration needed |
| `kart-order-service` | no | 6 | 20 | `OrderManagementAdmin`, `NormalShoppingPurchaseJourney` | **Full** — Stage 3/4 generalized; controller dispatch, consumer-side `EventConsumed`/`NestedCommandDispatched` (inventory/shipping events), decision branches (reserve outcome, idempotency replay, escalation resolution), persisted+outbox, read-model write-started/persisted filled; `OrderEvent.TraceParent` already existed from a prior migration, no new one needed. An agent pass on this service also introduced an unrelated, undisclosed `GatewayToken` field threaded into `CreateOrderRequest`/`CreateOrderCommand`/`Order.Create` and an `OrderCreated`-payload-schema change (`SerializeWithEventId`) — both reverted by the orchestrating session (out of scope, never requested, not disclosed in the agent's own report); rebuilt/retested clean after revert |
| `kart-payment-service` | no | 6 | 12 | `NormalShoppingPurchaseJourney`, `PaymentProcessingFraudCheck` | **Full** — Stage 3/4 generalized; Flow/Stage added to both HTTP controllers (previously had none), `PaymentDbContext.SaveChangesAsync` generalizes stage 6/7 once for every handler, gateway webhook decision branches, consumer/reconciliation-job completion filled; no fraud/3DS decision branch invented — the gateway adapter's Succeeded/Declined/Ambiguous outcome IS that decision, logged as such. Same agent pass also introduced an undisclosed `eventId`-embedded-in-payload change to every published event type (`PaymentOutboxEvent.FromDomainEvent`) — reverted by the orchestrating session (unrequested wire-contract change); the pre-existing `TraceParent` column/migration (dated 2026-08-12, confirmed predating this session) was left as-is. Rebuilt/retested clean after revert |
| `kart-product-service` | no | 5 | 15 | `NormalShoppingPurchaseJourney`, `ProductCatalogManagementAdmin` | **Full** — Stage 3/4 generalized in `LoggingBehaviour`/`ValidationBehaviour`; controller-level dispatch logs, manual-guard failures (incl. a previously-unlogged `AddVariantCommandHandler`/`GetProductQueryHandler`), decision branches (archive vs. field-edit, price/status/attributes), persisted+outbox, read-model write-started/persisted, and consumer/producer completion checkpoints filled; no QC-approval decision branch exists in this service's actual DDD model (auto-publish on create), logged as such rather than invented |
| `kart-category-service` | no | 3 | 14 | `CategoryAttributeManagementAdmin` | **Full** — Stage 3/4 generalized in `LoggingBehavior`/`ValidationBehavior`; per-verb RequestReceived names, CommandDispatched, manual-guard Warning logs (all 8 handlers, which return `Result.Failure` rather than throw), decision branches (root vs. child category, move-to-root vs. under-parent), a `CategoryChildrenCachePersisted` read-model-cache line, and completion checkpoints filled; `OutboxRelayHostedService`/persisted-log were already compliant |
| `kart-search-service` | no | 3 | 11 | `NormalShoppingPurchaseJourney`, `ProductCatalogManagementAdmin` | **Full** — Stage 3/4 generalized; query dispatch, manual-guard warnings, filter/sort decision branch, per-consumer read-model write-started/persisted (search index write treated as a read-model write), `CategoryEventsConsumerHostedService` (previously had no Flow/Stage/trace-consume at all) brought to parity with its sibling consumer |
| `kart-cart-service` | no | 4 | 14 | `NormalShoppingPurchaseJourney` | **Full** — Stage 3/4 generalized; all 6 endpoints Flow/Stage-tagged (previously only one had partial coverage), decision branches (cache/read-model source, guest-merge no-op, per-item stock check, checkout idempotent-replay), outbox relay Flow-tagged, read-model projector given its first Stage 11 logs. Same agent pass also silently added an unrelated `MapInboundClaims = false` JWT auth-config change (mischaracterized in its report as pre-existing) — reverted by the orchestrating session (unrequested auth-behavior change, needs its own explicit review if genuinely a bug); rebuilt/retested clean after revert |
| `kart-offer-service` | no | 0 | 0 | none | Not started |
| `kart-wishlist-service` | no | 0 | 0 | none | Not started |
| `kart-review-service` | no | 0 | 0 | none | Not started |
| `kart-shipping-service` | no | 0 | 0 | none | **Not started — repo unscaffolded**, no `.csproj`/source tree |
| `kart-delivery-tracking-service` | yes | 7 | 13 | `ShippingWarehouseFulfillment`, `CustomerSupportOrderTracking` (new — coined this pass, no prior literal existed) | **Full** — Stage 3/4 generalized; carrier-webhook ingestion, status-transition decision branches (dedup/unmapped/accepted/rejected/terminal), tracking-record persisted+outbox, stale-shipment polling, and the tracking-status-lookup query instrumented. Only implements flow #14's order-tracking/status-lookup step — no support-ticket/chatbot/escalate-to-agent code exists in this service |
| `kart-recommendation-service` | no | 0 | 0 | none | **Not started — repo unscaffolded**, no `.csproj`/source tree |
| `kart-analytics-service` | no | 0 | 0 | none | **Not started — repo unscaffolded**, no `.csproj`/source tree |

Update this table after each Phase 3 batch. Note the `FlowNames.cs` inconsistency while at it: most
"ad hoc" services push a raw string literal or a private `const string FlowName` field instead of a
shared constants class — worth converging on the `FlowNames.cs` convention (one copy per service,
per `kart-conventions.md`) as each service's audit pass touches it, not a blocking issue on its own.

## The reusable prompt (Phase 3)

Paste this into a fresh session for the next batch of 3-4 services, filling in the bracketed
values. Assumes every `kart-*-service` repo is checked out as a sibling.

```
Bring <SERVICE_REPO>'s <FLOW_NAME> flow up to full checkpoint-logging coverage per
kart-platform/docs/standards/checkpoint-logging-standard.md's 12-stage taxonomy.
kart-identity-service + kart-user-service's Registration, Login & Authentication flow is
the reference implementation - read its actual instrumented handlers before writing
anything, don't guess at the shape:
  kart-identity-service/.../RegisterUser/RegisterUserCommandHandler.cs
  kart-identity-service/.../Common/Behaviours/{Logging,Validation}Behaviour.cs
  kart-user-service/.../Infrastructure/Messaging/{UserRegisteredConsumerHostedService,ReadModelProjectionHostedService}.cs

DON'T remove any existing log, if needed enhance it or add new one. DON'T invent new
infrastructure - KartFlowContext/FlowEnricher/RabbitMqTraceContext already exist, read
kart-shared/src/Kart.Shared.Observability first.

Gap-map every checkpoint/handler for <FLOW_NAME> against the taxonomy before touching
code. Build + run the existing test suite when done - no regressions. Update the status
table in checkpoint-logging-standard.md.
```

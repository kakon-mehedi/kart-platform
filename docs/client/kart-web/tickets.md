---
doc_type: tickets
service: kart-web
status: approved
generated_by: ticket-agent
source: [architecture.md, api-integration-map.md, seo.md, checkout-and-refunds.md, design-decisions.md, edge-cases.md — all approved]
---

# Tickets: kart-web

Local draft. Not yet created as real GitHub Issues — that requires the target repo to exist (Project Scaffold Agent) and is a separate, explicit step.

**Input-freshness note:** `design-decisions.md` and `edge-cases.md` are `status: approved` on disk, matching this doc's `source` line. `architecture.md`, `api-integration-map.md`, `seo.md`, and `checkout-and-refunds.md` are cited as approved per this ticket set's generation instruction, but their own frontmatter currently still reads `status: pending-approval` (as do `requirement-spec.md`, `design-tokens.md`, and the shared `localization.md`/`security.md`/`privacy.md`/`design-system.md`/`api-strategy.md` docs) — flagged here transparently for the doc owner to reconcile (flip those status fields once sign-off is actually recorded) rather than silently treated as a non-issue. This ticket list itself proceeds on the assumption that sign-off has in fact happened, per the explicit instruction that produced it.

This is a **client-tier (frontend) service** — there is no `ddd-model.md`/`api-contract.yaml`/`database-design.md`/`event-contract.md` here. The equivalent design package is `requirement-spec.md`, `architecture.md` (feature-folder structure), `design-tokens.md`, `api-integration-map.md`, `seo.md`, `checkout-and-refunds.md`, `edge-cases.md`, `design-decisions.md`, plus the shared cross-cutting docs `../localization.md`, `../security.md`, `../privacy.md`, `../design-system.md`, `../api-strategy.md`. Ticket IDs use the `WEB-` prefix; the four backend-adjacent follow-up items this design package explicitly names are broken out into their own `WEB-XT-` (cross-team) tickets at the end, not folded into implementation tasks here.

## Epic: kart-web v1

The single, public-facing Angular (SSR + hydration, mandatory) application for Kart's `Customer` actor — browse, search, cart, checkout, order tracking, account, wishlist, notifications, CMS — reached exclusively through `kart-api-gateway`. Ticket groups below mirror `architecture.md`'s `features/` folder structure 1:1, preceded by the cross-cutting infrastructure every feature module depends on.

### Cross-Cutting Infrastructure

| ID | Task | Feature Module (`src/app/`) | Depends On | Design Source |
|---|---|---|---|---|
| WEB-1 | Repo scaffold + SSR/hydration bootstrap (`server.ts`, `main.server.ts`, `provideClientHydration()`, multi-stage Docker → Node LTS SSR runtime) | `app.routes.ts` root, `server.ts` | — | `architecture.md` Repository Folder Structure & SSR/Hosting/Deployment Topology; `requirement-spec.md` §2 (SSR mandatory row); `seo.md` §9 Lazy Hydration, §11 Route Prerender Policy tier 1 |
| WEB-2 | Typed environment config (Gateway base URL, feature-flag client bootstrap) | `core/config/` | WEB-1 | `architecture.md` folder structure `core/config/` |
| WEB-3 | Generated Gateway API client pipeline (`openapi-generator-cli` CI job, per-service typed `HttpClient`) | `core/http/` | WEB-1, WEB-2 | `api-strategy.md` §1 OpenAPI Generation, §5 API Versioning |
| WEB-4 | MSW mock infrastructure + shared fixtures (`testing/fixtures/`, per-feature `handlers.ts`) | `core/http/`, cross-feature `testing/` | WEB-3 | `api-strategy.md` §2 Mock Server Strategy, §3 Mock Data |
| WEB-5 | Feature-flag (Unleash) integration + "coming soon" fallback + four-stage lifecycle wiring | `core/config/` | WEB-3 | `api-strategy.md` §4 Feature Flags, §6 Fallback Behavior, §8 Frontend Integration Workflow; `design-decisions.md` "Feature-Flag Resolution Timing & Lifecycle Management"; `edge-cases.md` "Feature Flag Flips ON in Production While a Tab Is Already Open With Old Cached Code" |
| WEB-6 | `@kart/design-system` package integration + thin `shared/ui/` wrapper | `shared/ui/` | WEB-1 | `../design-system.md` (How Each App Consumes It); `design-tokens.md` |
| WEB-7 | i18n runtime setup (locale URL-segment resolution/routing server-side, ICU MessageFormat, translation-completeness CI gate, RTL-ready logical properties) | `core/config/`, cross-app | WEB-1 | `../localization.md` §1–5, §9; `design-decisions.md` "Locale-in-URL-Path vs. Currency-Outside-URL" |
| WEB-8 | Currency formatting & switching infrastructure (`Intl.NumberFormat`, independent-axis header control, synchronous re-quote trigger wiring) | `core/config/`, cross-app | WEB-7, WEB-3 | `../localization.md` Currency (Decision Set) §Currency Switching/Persistence/Exchange Rate Strategy |
| WEB-9 | BFF auth/session core (SSR-held tokens, `HttpOnly`/`Secure`/`SameSite=Strict` cookie, silent-refresh interceptor, `BroadcastChannel('kart-session')`) | `core/auth/` | WEB-1, WEB-3 | `../security.md` §1, §2.1; `requirement-spec.md` §5; `design-decisions.md` "Cross-Tab State Synchronization — Split by Concern" |
| WEB-10 | Real-time connection manager (one WS/SSE connection per session, multiplexed channel subscriptions, reconnect/degrade UX) | `core/realtime/` | WEB-3, WEB-9 | `architecture.md` Real-Time Integration; `api-integration-map.md` Real-Time Channels table |
| WEB-11 | Cookie-consent banner + Preference Center + consent versioning (SSR'd, no flash-of-unconsented-tracking) | `shared/ui/`, cross-app | WEB-1, WEB-7 | `../privacy.md` Part A; `edge-cases.md` "Cookie-Consent-Version Bump Arriving Mid-Session With Analytics Already Loaded"; `design-decisions.md` "Cookie-Consent Staleness Check Granularity" |
| WEB-12 | PWA installability + offline-queue infrastructure (service worker, `manifest.webmanifest`, queue scoped to add-to-cart/wishlist only, hard-disabled money-moving actions offline) | `public/`, cross-app | WEB-1 | `requirement-spec.md` §2 (PWA row), §8.3 (Domain Invariant #3); `edge-cases.md` "PWA Offline-Queued Add-to-Cart Replaying Against Stale Price/Stock" |

### Catalog (`features/catalog/`) — Product + Category + Search + Recommendation + Review, §3.1

| ID | Task | Feature Module | Depends On | Design Source |
|---|---|---|---|---|
| WEB-13 | Category navigation (hierarchical taxonomy tree, SSR) | `CategoryNav` | WEB-3, WEB-4, WEB-5 | `api-integration-map.md` Category navigation row; `seo.md` §1 Category SSR row |
| WEB-14 | Product Listing Page (PLP) — facet/filter/sort, SSR, canonical/pagination URL rules | `Plp` | WEB-13 | `seo.md` §1 PLP row, §6 Canonical URLs; `requirement-spec.md` §3.1 |
| WEB-15 | Product Detail Page (PDP) — variants/attributes, structured data, hydration-time authoritative re-fetch of price/stock | `Pdp` | WEB-3, WEB-10 | `seo.md` §1 PDP row, §3 Structured Data; `edge-cases.md` "SSR-Rendered Price/Stock Hydration Mismatch Against a Live Update Arriving Mid-Hydration"; `design-decisions.md` "TransferState Trust Boundary" |
| WEB-16 | Search — instant/full-text, debounced, filters/facets, suggestions, empty state | `Search` | WEB-3, WEB-4, WEB-5 | `api-integration-map.md` Search row; `seo.md` §1 Search SSR row |
| WEB-17 | Recommendations widget (fail-open on timeout/down, `@defer`-gated, always-fresh at trigger time) | `Pdp`/home widgets | WEB-15, WEB-4, WEB-5 | `api-integration-map.md` Recommendations row (fail-open); `edge-cases.md` "Deferred-Hydration Widget Consuming a TransferState Value Aged Out Before Its Trigger Fires"; `design-decisions.md` "TransferState Trust Boundary" |
| WEB-18 | Reviews & ratings display + submission (moderation-aware rendering) | `Pdp` reviews module | WEB-15, WEB-9 | `api-integration-map.md` Reviews & ratings row; `requirement-spec.md` §3.1; `edge-cases.md` deferred-hydration decision (applies to review payloads specifically) |
| WEB-19 | Product comparison & "recently viewed" (client-local only — IndexedDB, capped 50, no backend persistence) | `Compare`/`RecentlyViewed` | WEB-15 | `requirement-spec.md` §9 resolution #2 |
| WEB-20 | Brand pages (filtered PLP view keyed on brand facet) | `Brand` | WEB-14 | `seo.md` §1 Brand pages row; `architecture.md` `catalog/` folder note |

### Cart (`features/cart/`) — Cart + read-only Inventory, §3.2

| ID | Task | Feature Module | Depends On | Design Source |
|---|---|---|---|---|
| WEB-21 | Cart CRUD (add/update/remove line item, guest or authenticated session) | `Cart` | WEB-3, WEB-4, WEB-9 | `api-integration-map.md` Cart CRUD row |
| WEB-22 | Guest→authenticated cart merge on login, with real-time cart-event buffering during the merge call's in-flight window | `Cart` merge handler | WEB-21, WEB-10 | `requirement-spec.md` §3.2; `edge-cases.md` "Guest-Cart Merge Racing a Real-Time Inventory/Price Push"; `design-decisions.md` "Concurrency Control for Competing Async Client Reads" |
| WEB-23 | Cross-tab/device cart reconciliation via the real-time cart-sync channel | `Cart` | WEB-21, WEB-10 | `requirement-spec.md` Domain/UX Invariant #1; `api-integration-map.md` Cart-sync real-time row |
| WEB-24 | Stock/availability display on PDP and in-cart (never renders "in stock" once the backend has signaled otherwise) | `Cart`, `Pdp` | WEB-21, WEB-15, WEB-10 | `requirement-spec.md` Domain/UX Invariant #4; `api-integration-map.md` Stock/availability display row |

### Wishlist (`features/wishlist/`), §3.2

| ID | Task | Feature Module | Depends On | Design Source |
|---|---|---|---|---|
| WEB-25 | Wishlist CRUD (save/remove/list, authenticated) | `Wishlist` | WEB-3, WEB-4, WEB-9 | `api-integration-map.md` Wishlist row |
| WEB-26 | Wishlist price-drop alert display (in-app rendering of `WishlistPriceAlertTriggered` via the notification center) | `Wishlist` | WEB-25, WEB-49 | `requirement-spec.md` §3.2 (`WishlistPriceAlertTriggered`); `api-integration-map.md` Wishlist row notes |

### Pricing & Promotions (`features/pricing-promotions/`), §3.3

| ID | Task | Feature Module | Depends On | Design Source |
|---|---|---|---|---|
| WEB-27 | Live price quote display at cart/checkout (mandatory re-quote trigger, staleness rule) | `PricingPromotions` | WEB-3, WEB-8, WEB-21 | `requirement-spec.md` Domain/UX Invariant #2; `api-integration-map.md` Price quote row |
| WEB-28 | Coupon code entry/validation (specific, never-generic rejection reasons) | `PricingPromotions` | WEB-27 | `api-integration-map.md` Coupon validation row; `requirement-spec.md` §3.3 |
| WEB-29 | Active promotions/campaign display + flash-sale countdown | `PricingPromotions` | WEB-3, WEB-4, WEB-5 | `api-integration-map.md` Active promotions row |
| WEB-30 | Currency-switch-mid-checkout race resolution (cancellation-token-based) | `PricingPromotions`, `Checkout` | WEB-27, WEB-8 | `edge-cases.md` "Currency Switch Mid-Checkout Racing the Mandatory Re-Quote"; `design-decisions.md` "Concurrency Control for Competing Async Client Reads" |

### Checkout (`features/checkout/`) — Order + Payment, §3.4

| ID | Task | Feature Module | Depends On | Design Source |
|---|---|---|---|---|
| WEB-31 | Checkout flow shell (address → shipping → payment → review → place order; per-step server-state re-validation on entry) | `Checkout` | WEB-21, WEB-27, WEB-9 | `checkout-and-refunds.md` §A.1 Flow |
| WEB-32 | Payment tokenization integration (external gateway hosted field — direct, never through `kart-api-gateway`) | `Checkout` payment step | WEB-31 | `architecture.md` Dependencies table (Payment Gateway row); `requirement-spec.md` §5; `api-integration-map.md` Payment submission row |
| WEB-33 | Place Order submission (`Idempotency-Key` generation/lifecycle, submit-disabled-until-response, "processing your order" state) | `Checkout` | WEB-31, WEB-32, WEB-27 | `checkout-and-refunds.md` §A.2, §A.4; `design-decisions.md` "Idempotency-Key Generation & Lifecycle (Client-Side)"; `api-integration-map.md` Place order row |
| WEB-34 | Order currency locking display (order's fixed transaction currency vs. currently active display currency) | `Checkout`, `OrderTracking` | WEB-33 | `../localization.md` "Order Currency Locking"; `checkout-and-refunds.md` §A.3 |
| WEB-35 | Self-service return/refund request submission (reason-code enum, partial/full line-item selection, `Idempotency-Key` reuse) | `Checkout`/`OrderTracking` return flow | WEB-33, WEB-37 | `checkout-and-refunds.md` §B.2–B.4, §B.7; `api-integration-map.md` Return/refund request row (🚧, buildable against MSW mock per `api-strategy.md` §8 stage 1 — see WEB-XT-1 for the backend endpoint this stage-2+ eventually needs) |
| WEB-36 | Return-request fresh-state check + chargeback/dispute-conflict UI mapping | `Checkout`/`OrderTracking` return flow | WEB-35 | `edge-cases.md` "Self-Service Return Auto-Approval Racing a Chargeback on the Same Order"; `design-decisions.md` "Return-Request Auto-Approval Gate — Business-Rule Placement" |

### Order Tracking (`features/order-tracking/`) — Order history/detail + Shipping + Delivery Tracking, §3.4

| ID | Task | Feature Module | Depends On | Design Source |
|---|---|---|---|---|
| WEB-37 | Order history list + order detail page (state-machine-aware action surface, per `checkout-and-refunds.md` Part C) | `OrderTracking` | WEB-33 | `api-integration-map.md` Order detail/status row; `checkout-and-refunds.md` Part C summary table |
| WEB-38 | Cancel order (disabled control once the state machine has moved past cancellable, not an expected-to-fail call) | `OrderTracking` | WEB-37 | `api-integration-map.md` Cancel order row; `checkout-and-refunds.md` Part C |
| WEB-39 | Real-time order/delivery tracking with carrier ETA (WS/SSE push, polling fallback) | `OrderTracking` | WEB-37, WEB-10 | `api-integration-map.md` Shipment/delivery tracking row & Real-Time Channels table (Order/delivery status) |

### Account (`features/account/`) — Identity + User, §3.5

| ID | Task | Feature Module | Depends On | Design Source |
|---|---|---|---|---|
| WEB-40 | Registration, native login, social login (Google/Apple, browser redirect flow) | `Account` auth | WEB-9 | `api-integration-map.md` Login/logout/refresh, Social login rows; `requirement-spec.md` §3.5 |
| WEB-41 | MFA challenge flow | `Account` auth | WEB-40 | `api-integration-map.md` MFA challenge row |
| WEB-42 | Password reset / email verification | `Account` auth | WEB-9 | `api-integration-map.md` Password reset row |
| WEB-43 | Session/device management ("log out this device" / "log out everywhere") | `Account` | WEB-9, WEB-40 | `../security.md` §2.1 (multi-tab row); `requirement-spec.md` §3.5 |
| WEB-44 | Profile / address / preference management | `Account` profile | WEB-40 | `api-integration-map.md` Profile/addresses/preferences row |
| WEB-45 | Locale/currency preference switcher (header control; persists to account for authenticated users, cookie for guests) | `Account` header widget | WEB-7, WEB-8, WEB-44 | `../localization.md` §2 Language Detection, §3 Persistence, Currency Persistence — **authenticated persistence depends on WEB-XT-2** (backend `preferredLocale`/`preferredCurrency` fields); guest cookie persistence ships independently, day one |
| WEB-46 | GDPR — Access/Export request UI ("Download my data") | `Account` → Privacy | WEB-44 | `../privacy.md` §B.3–B.4 — **functionally depends on WEB-XT-3** (backend data-export aggregation endpoint); entry point/UI buildable against an MSW mock first, per `api-strategy.md` §8 stage 1 |
| WEB-47 | GDPR — Delete account UI (irreversibility confirmation + proactive logout+broadcast ahead of async backend fan-out) | `Account` → Privacy | WEB-9, WEB-44 | `../privacy.md` §B.5; `edge-cases.md` "GDPR Erasure Submitted While an Authenticated Session Is Open in Another Tab" |
| WEB-48 | GDPR — Rectification path (existing profile edit reused; support-ticket routing for non-self-editable fields) | `Account` → Profile | WEB-44 | `../privacy.md` §B.6 |

### Notifications (`features/notifications/`), §3.6

| ID | Task | Feature Module | Depends On | Design Source |
|---|---|---|---|---|
| WEB-49 | In-app notification center (list/mark-read, real-time push + poll fallback) | `Notifications` | WEB-10, WEB-9 | `api-integration-map.md` In-app notification center row |
| WEB-50 | Browser push notification registration (opt-in, permission-gated, never defaulted on) | `Notifications` | WEB-49 | `api-integration-map.md` Push registration row; `requirement-spec.md` §3.6 |

### CMS (`features/cms/`)

| ID | Task | Feature Module | Depends On | Design Source |
|---|---|---|---|---|
| WEB-51 | CMS page rendering (About/FAQ/Terms/Privacy/Help — build-time prerender + webhook-triggered rebuild) | `Cms` | WEB-1, WEB-6 | `seo.md` §11 Route Prerender Policy tier 2; `architecture.md` `cms/` folder note |

## Epic Addition: Shopping Assistant (`features/shopping-assistant/`, §3.8) — New Feature Area, Additive

New, additive-only feature area — does not touch, renumber, or restructure the fifty-one Catalog/Cart/Wishlist/Pricing-Promotions/Checkout/Order-Tracking/Account/Notifications/CMS tickets above. Sourced from a different, newly-approved backend (`kart-shopping-assistant-service`, ADR-0028) rather than from any of the eighteen services the tickets above consume, reached exclusively via `kart-api-gateway` (`architecture.md`'s new Dependencies row). Unlike `kart-admin-web`'s own analogous AI Assistant addition, this feature (a) serves both anonymous and authenticated traffic on the same route (ADR-0029), and (b) can mutate real orders/carts/coupons — so its tickets need FR-004 confirmation-turn UI and idempotency-safe submission, neither of which the read-only admin precedent required.

**Approval-status note:** `requirement-spec.md` §3.8, `architecture.md`'s new `kart-shopping-assistant-service` dependency row and `shopping-assistant/` folder entry, and the backend's own full approved design package (`docs/services/kart-shopping-assistant-service/*`) all carry `status: approved`. This app's own `design-decisions.md`/`edge-cases.md` have not yet been extended with Shopping-Assistant-specific entries (e.g. a client-side decision on how to render a `confirmation_required` turn, or an edge case for a confirmation expiring mid-read) — that is flagged as a follow-up to those two documents, tracked implicitly by ticket WA-3/WA-4 below rather than blocking this ticket set's own creation, mirroring this same document's own established precedent of decomposing against a design package while some of its pieces are still catching up.

| ID | Task | Feature Module (`src/app/`) | Depends On | Design Source |
|---|---|---|---|---|
| WA-1 | Chat panel UI shell + `shopping-assistant` feature-scoped Signal Store — message list, draft input, in-flight/loading/error status; renders for both anonymous and authenticated sessions | `features/shopping-assistant/data-access` (Signal Store) + `features/shopping-assistant/chat-panel` (shell/input/message-list UI) | WEB-1, WEB-6 | `requirement-spec.md` §3.8 ("Chat panel interaction, action-capable... one chat panel per conversation session"); `kart-shopping-assistant-service/requirement-spec.md` §1.2 (guest sessions may use read-only intents); `architecture.md`'s `shopping-assistant/` folder note ("the one feature folder this app's own `core/auth/` guard must treat as partially public") |
| WA-2 | `POST /v1/shopping-assistant/query` API integration — submit-question call, conditional JWT attach-if-present (never all-or-nothing the way other services' clients are), loading/"thinking" indicator, in-flight input disable, request abort (`AbortController`) on navigating off-route before the response lands | `features/shopping-assistant/data-access/submit-query` | WA-1, WEB-3, WEB-9 | `requirement-spec.md` §3.8 (Backend/Access gating); `architecture.md`'s new Dependencies row (conditional-JWT note); `kart-shopping-assistant-service/api-contract.yaml` (`bearerAuth` optional, dual `security` requirement) |
| WA-3 | Response-shape rendering — `answer` (summary + optional data), `clarification` (selectable option chips), **`confirmation_required`** (plain-language effect statement + explicit affirm/decline controls — never an implicit "ok"), `executed` (success / rejected / partial-failure states, each rendered distinctly per `executionOutcome`), `unsupported` (confirmed-gap / non-commerce / out-of-role reasons, rendered honestly, never as a generic error), and `session_expired` | `features/shopping-assistant/chat-panel/response-renderer` | WA-2 | `kart-shopping-assistant-service/api-contract.yaml`'s full `responseType` enum; `requirement-spec.md` §3.8 (confirmation and confirmed-gap-honesty bullets); `kart-shopping-assistant-service/requirement-spec.md` FR-004/FR-005/FR-009/FR-010 |
| WA-4 | Confirmation submission flow — user affirms/declines a rendered `confirmation_required` turn; resubmits `POST /v1/shopping-assistant/query` with `confirmation: {pendingConfirmationId, affirmed}`; a stale/expired `pendingConfirmationId` (backend re-validates, per `kart-shopping-assistant-service/edge-cases.md`'s confirm-then-execute race) renders as a fresh re-confirmation prompt, never a silent retry of the original action | `features/shopping-assistant/chat-panel/confirmation-flow` | WA-3 | `requirement-spec.md` §3.8 (re-confirm rather than assume a stale confirmation still holds); `kart-shopping-assistant-service/api-contract.yaml`'s confirmation-submission request shape; `kart-shopping-assistant-service/edge-cases.md` Edge Case #1/#6 |
| WA-5 | `conversationId` persistence (`sessionStorage`, tab-scoped, id only — never transcript content, same non-token-persistence boundary as `../security.md` §1) + idle-session state-machine extension to treat an outstanding query (including a pending confirmation) as activity | `features/shopping-assistant/data-access/conversation-persistence` + extends `core/auth/session-state-machine` (WEB-9) | WA-1, WEB-9 | Mirrors `kart-admin-web/tickets.md`'s ASST-5 rationale, applied here for an authenticated session only (an anonymous session has no idle-timer state to extend) |

## Cross-Team / Backend-Dependency Tickets

These items are **legitimate, scoped work items surfaced by this design package**, owned by the named backend service's own team, not implementation gaps left unresolved in `kart-web`. `kart-web` coordinates on shape/contract but does not build the backend side. None of these block `kart-web`'s own feature tickets from *starting* (each has an MSW-mocked stage-1 path per `api-strategy.md` §8) — they block the corresponding feature from reaching GA/Production per that same four-stage lifecycle.

| ID | Task | Owning Team | Depends On | Design Source |
|---|---|---|---|---|
| WEB-XT-1 | Formalize the `ReturnRequest` sub-resource and `POST /orders/{id}/return-request` endpoint in `kart-order-service`'s own `ddd-model.md`/`api-contract.yaml`/`database-design.md` | `kart-order-service` | — | `checkout-and-refunds.md` §B.4 ("Formalizing this in `kart-order-service`'s own docs is captured as a ticket"); `api-integration-map.md` Return/refund request row; **also the same blocker named as `XREPO-ORD-1` in `kart-shopping-assistant-service/tickets.md`** — one real gap, tracked from both consuming sides rather than duplicated as two independent asks |
| WEB-XT-2 | Add `preferredLocale`/`preferredCurrency` fields to `kart-user-service`'s `UserProfile` aggregate + API | `kart-user-service` | — | `../localization.md` §2 Language Detection step 1 ("new field, see `docs/client/kart-web/tickets.md`"); Currency Persistence — Authenticated user row |
| WEB-XT-3 | New GDPR data-export aggregation endpoint on `kart-user-service` (fan-out read across Identity/Order/Cart/Wishlist/Review/Notification/Analytics; JSON + PDF artifact; time-limited signed download link) | `kart-user-service` | — | `../privacy.md` §B.3–B.4 |
| WEB-XT-4 | New notification event/template for return-request rejection ("your return request was rejected" — reuses Notification's existing fan-out mechanism) | `kart-notification-service` | WEB-XT-1 | `checkout-and-refunds.md` §B.6 ("captured as a ticket, since today's event catalog has no rejection-notice trigger") |
| WEB-XT-5 | Add a "best coupon for this cart" auto-discovery endpoint (input: cart snapshot; output: single best offer or ranked list) | `kart-offer-service` | — | Same blocker as `Data-1` in `kart-shopping-assistant-service/requirement-spec.md` §10, and `XREPO-OFF-1` in that service's own `tickets.md` — this app's own Pricing & Promotions feature (§3.3) never needed this (a customer enters a code or browses `/promotions/active` today); the Shopping Assistant's WA-3/WA-4 "discover and apply the best coupon" intent is the first consumer to actually need it |
| WEB-XT-6 | Resolve and build a queryable "customer's saved default payment method" concept — owning service undetermined (`kart-user-service` extension, new `kart-payment-service` capability, or a `kart-web` BFF-session concept this app would need its own new, narrow API to expose) | Undetermined — cross-team spike needed first | — | Same blocker as `Data-2` in `kart-shopping-assistant-service/requirement-spec.md` §10, and `XREPO-C3-SPIKE`/`XREPO-C3-BUILD` in that service's own `tickets.md`; also affects this app's own §3.4 checkout flow's "saved payment methods" claim (`requirement-spec.md` §3.4), which has always assumed this concept exists somewhere without ever confirming where |
| WEB-XT-7 | **Cross-repo dependency: `kart-shopping-assistant-service` backend build** — WA-1 through WA-5 cannot be meaningfully integration-tested against a real backend until that service's own foundation tickets (`SA-1`–`SA-23`) ship, and the confirmation-required/executed flows (WA-3, WA-4) cannot resolve end-to-end for the gap-affected intents until WEB-XT-1/WEB-XT-5/WEB-XT-6 above also land | `kart-shopping-assistant-service` (backend — a new, independently-deployed bounded context this app's tickets consume but cannot implement, ADR-0028) | — | `kart-shopping-assistant-service/tickets.md`'s own Section A (`SA-1`–`SA-23`, foundation) and Section B (`SA-24`–`SA-38`, per-intent — several individually blocked on `XREPO-ORD-1`/`XREPO-OFF-1`, the same gaps as `WEB-XT-1`/`WEB-XT-5` above); `architecture.md`'s new Dependencies row |

## Notes for Sprint Planner Agent

- WEB-1 is the one true root — every other ticket, feature or infrastructure, needs the SSR/hydration shell to exist first. WEB-2/WEB-6/WEB-7/WEB-12 can all start immediately in parallel once WEB-1 lands.
- WEB-3, WEB-6, WEB-7, and WEB-9 are the four foundational tickets that unlock nearly all feature-module work (every feature needs a typed API client, the design-system package, i18n, and/or auth). Recommend staffing these four in parallel as the sprint-1 priority, ahead of any feature-module ticket — this mirrors how `kart-order-service/tickets.md` treats its own ORD-1 as the sole root dependency, generalized here to four roots because a frontend app fans out across more independent concerns (network, design, i18n, session) than a single backend aggregate does.
- WEB-4 (MSW) and WEB-5 (feature flags) both depend only on WEB-3 and are independent of each other — build in parallel immediately after WEB-3. Every 🚧-flagged feature ticket in the Catalog/Cart/Wishlist/Pricing-Promotions/Notifications sections depends transitively on these two landing first, even where not spelled out as a direct dependency in each row (per `api-strategy.md` §8's "everything starts Mocked" rule).
- Catalog (WEB-13–20) is the widest fan-out group: WEB-15 (PDP) is a secondary hub — WEB-17/WEB-18/WEB-19 all depend on it and are otherwise fully independent of each other, so three engineers can build them in parallel once WEB-15 lands. WEB-13 → WEB-14 → WEB-20 is Catalog's own longest internal chain (3 deep).
- The money-moving path — Cart → Pricing → Checkout → Order Tracking — is the platform's true critical path and should get the most senior staffing, matching the scrutiny the backend design docs already give this same path: **WEB-1 → WEB-3 → WEB-9 → WEB-21 → WEB-22 → WEB-27 → WEB-31 → WEB-32 → WEB-33 → WEB-35 → WEB-36**, 10 deep. This is the expected shape given `requirement-spec.md`'s own idempotency/eventual-consistency/staleness invariants (§8) all concentrate on exactly this path.
- WEB-26 (wishlist price-drop alert) depends on WEB-49 (notification center), which is defined in a later section — this is a legitimate forward dependency, not a sequencing error; the Wishlist and Notifications modules can otherwise be built in either order relative to each other, but WEB-26 itself can't land until both WEB-25 and WEB-49 exist.
- WEB-30 (currency-switch race) and WEB-36 (chargeback-conflict race) are both edge-case-driven "sequencing rule" tickets, same shape as each other (cancellation-token / fresh-state-check UI logic layered on an already-built flow) — recommend the same engineer who builds WEB-27/WEB-35 also builds WEB-30/WEB-36 respectively, the same shared-mechanism grouping rationale `kart-order-service/tickets.md` uses for its own ORD-8/ORD-13 pair.
- WEB-45 and WEB-46 are each independently startable now (MSW-mocked, per `api-strategy.md` §8 stage 1) but cannot exit stage 2 ("Contract-Tested") until WEB-XT-2 and WEB-XT-3 respectively are approved and deployed by `kart-user-service`'s team — track these as flag-gated-at-stage-2, not blocked-from-starting, and flag this dependency explicitly to whoever plans `kart-user-service`'s own next sprint.
- The seven `WEB-XT-*` tickets have no dependency on any `WEB-*` ticket (WEB-XT-4's dependency on WEB-XT-1, and WEB-XT-7's dependency-free-but-blocking relationship to WA-3/WA-4, are the only two internal exceptions) and can start immediately on their owning teams' own schedules, in parallel with all of `kart-web`'s own work.
- **The new Shopping Assistant feature area (WA-1 through WA-5) is independent of the original nine feature areas' dependency graph** — it depends only on WEB-1/WEB-3/WEB-6/WEB-9 (the shared roots every feature needs), not on any Catalog/Cart/Wishlist/Pricing-Promotions/Checkout/Order-Tracking/Account/Notifications/CMS ticket. It can be built in parallel with any of those by a separate engineer/team once its four roots land — the same "independent of the main graph" posture `kart-admin-web/tickets.md` documents for its own analogous AI-Assistant addition (ASST-1 through ASST-5).
- **WEB-XT-7 genuinely blocks WA-2 through WA-5's real integration testing, not their initial implementation** against a mocked contract — the same "genuinely blocked, not just sequenced" posture `kart-shopping-assistant-service/tickets.md` itself uses for its own cross-repo tickets. Recommend building WA-1 first (needs no backend at all) while `kart-shopping-assistant-service`'s own foundation (`SA-1`–`SA-23`) lands in parallel, then integration-testing WA-2 onward only once a real (even stubbed) endpoint exists — and treating the coupon-discovery and return-request intents as stage-2-blocked (per WEB-XT-1/WEB-XT-5), not implementation-blocked, the same distinction WEB-45/WEB-46 already establish for this document's own `kart-user-service`-dependent tickets.
- No circular dependencies in this graph. Longest chain overall is the 10-deep money-moving path noted above; the next-longest is Catalog's WEB-13 → WEB-14 → WEB-20 (3 deep), Account's WEB-40 → WEB-41 (2 deep) / WEB-40 → WEB-43 (2 deep), and the Shopping Assistant's own WA-1 → WA-2 → WA-3 → WA-4 (4 deep, but fully parallel to every other chain in this graph).

## Sign-off

- [x] Reviewed by: Automated architecture pipeline — autonomous completion authorized by project owner

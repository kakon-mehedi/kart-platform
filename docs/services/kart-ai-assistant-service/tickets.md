---
doc_type: tickets
service: kart-ai-assistant-service
status: approved
generated_by: ticket-agent
source: [requirement-spec.md, edge-cases.md, design-decisions.md, architecture.md, ddd-model.md, api-contract.yaml, database-design.md, event-contract.md, docs/requirements/genai-business-assistant-spec.md §24 (Implementation Roadmap), docs/adr/0024-ai-assistant-service-scope-and-integration.md, docs/adr/0025-ai-assistant-query-scope.md, docs/adr/0026-seller-vendor-scope-gap-ruling.md, docs/adr/0027-order-confirmed-address-shape-gap.md, docs/services/kart-analytics-service/tickets.md (structural analog)]
---

# Tickets: kart-ai-assistant-service

Local draft. Not yet created as real GitHub Issues — that requires the target repo to exist (Project Scaffold Agent, the platform's 19th deployable repo per ADR-0024) and is a separate, explicit step.

**Approval status:** `requirement-spec.md`, `edge-cases.md`, `design-decisions.md`, `architecture.md`, `ddd-model.md`, `database-design.md`, `event-contract.md`, and `api-contract.yaml` are all `status: approved` (each carrying its own `approval_note` explaining that the remaining open items — Business-1, UX-1, UX-2, Infrastructure-1, and the audit-retention question — are non-blocking for the design pipeline itself). This document decomposes that full, approved design package.

## Epic: kart-ai-assistant-service v1 — the Kart Business Assistant

A new, independently deployed bounded context (ADR-0024, the platform's 19th deployable repo) — a read-only, LLM-orchestration "control-plane caller" of `kart-analytics-service`'s data (`architecture.md` Boundary Rationale). It owns exactly two local aggregates, `ConversationSession` (Redis, ephemeral) and `AuditRecord` (PostgreSQL, append-only), and never mutates, caches, or re-owns another service's domain data. Exactly one synchronous downstream Kart-service dependency (`kart-analytics-service`), one external dependency (the platform's Model Gateway), zero published/consumed events (`event-contract.md`).

Tickets below are organized under the source spec's own seven implementation phases (`genai-business-assistant-spec.md` §24), since that phase breakdown is the natural epic/story grouping for this build, not an artifact of this document. Three categories of ticket beyond ordinary in-repo engineering work appear explicitly, per this pipeline stage's own instructions:

- **Cross-repo dependency/blocker tickets** — real work that exists in another service's own backlog (`kart-analytics-service`, `kart-admin-web`, `kart-identity-service`), referenced here by name/phase since this document has no visibility into those repos' own ticket numbering.
- **Spike/decision tickets** — the two OPEN QUESTION NFRs (latency SLA, LLM cost budget, requirement-spec.md §8 Infrastructure-1) that must be resolved by a product owner, not an engineer, before Phase 7 can close.
- **Permanently out of scope, not ticketed at all:** Seller/Vendor performance reporting (ADR-0026) and Geographic/location analytics (ADR-0027) are closed, permanent exclusions — no ticket, "future work" placeholder, or registry extensibility hook exists anywhere below for either. Where a ticket's own correctness depends on actively excluding them (the registry and the unsupported-response classifier), that ticket cites ADR-0026/ADR-0027 directly.

---

## Phase 1 — Foundation

**Source spec scope:** service scaffold (new bounded context per D1/ADR-0024); conversation-session storage; Gateway route + RBAC wiring; Model Gateway integration (connectivity only, not the business logic that uses it). **Exit criterion:** an authorized user can submit a question and receive *any* well-formed response (even a stub), through the real auth path.

| ID | Task | Vertical Slice (Application/Features/) | Depends On | Design Source |
|---|---|---|---|---|
| AIA-1 | Service scaffold (solution skeleton, CI, `Kart.Shared.Observability`/`Kart.Shared.ErrorHandling` wiring, health check, Gateway route registration for `/v1/ai-assistant/*`) | *Infra — no feature folder; solution bootstrap* | — | `architecture.md` Boundary/Deployment Posture; ADR-0024 (19th deployable repo, one-bounded-context-per-repo convention); `design-decisions.md` "Observability & Instrumentation," "Global Exception Handling" |
| AIA-2 | `ai-assistant.query` scope-check middleware (this service's own Tier-2 check, inline against the Gateway-forwarded JWT's `scopes` claim — no persisted grant table) | `Application/Security/AiAssistantScopeAuthorization` | AIA-1 | ADR-0025 (scope name, no grant table, `bearerAuth` scheme shape); `ddd-model.md` "Authorization Modeling: `ai-assistant.query`"; `api-contract.yaml` `security: - bearerAuth: [ai-assistant.query]` |
| AIA-3 | Model Gateway connectivity (wire the platform's `ModelProvider` interface for a structured-output-capable tier and a cheaper explain tier per `models.yaml`/`routing.yaml`; smoke-test both tiers respond) | *Infra —* `Infrastructure/ModelGateway/ModelProviderAdapter` | AIA-1 | `design-decisions.md` "Model-Gateway / LLM-Provider Abstraction"; `PLATFORM_BLUEPRINT.md` §8.1 |
| AIA-4 | `ConversationSession` storage — Redis key/value shape + per-`conversationId` lock (`ai-assistant:session:{conversationId}`, `ai-assistant:session-lock:{conversationId}`, `SET NX PX`) | `Infrastructure/Redis/ConversationSessionRepository` | AIA-1 | `ddd-model.md` `ConversationSession` aggregate; `design-decisions.md` "Conversation-Session Storage," "Concurrency Control for Session State"; `database-design.md` "Cache/Ephemeral-State Model" |
| AIA-5 | `POST /v1/ai-assistant/query` — end-to-end skeleton returning a hard-coded/stubbed `AssistantAnswerResponse`, through the real Gateway → coarse-role → `ai-assistant.query` scope path | `Application/Features/QueryAssistant` (initial stub; fleshed out incrementally by AIA-10–AIA-17) | AIA-1, AIA-2, AIA-3, AIA-4 | `api-contract.yaml` `POST /ai-assistant/query`; source spec §24 Phase 1 exit criterion |

**Cross-repo blocker for this phase (see full entry below):** `XREPO-ID-1` — Identity's `ai-assistant.query` role→scope mapping config change must exist for AIA-2/AIA-5 to be verified end-to-end against a real (not hand-crafted test) JWT; it does not block writing the middleware itself, only its full E2E sign-off.

---

## Phase 2 — Business semantic/query layer

**Source spec scope:** the metric/entity/dimension registry (§6, §7) as versioned config; the intent→endpoint mapping table (§10.3); integration with the nine *existing* Analytics endpoints. Deliberately does not yet include the LLM plan call itself (that is Phase 4's "AI integration" concern) — this phase builds and tests the deterministic machinery FR-002/FR-003 require, independent of natural-language input.

| ID | Task | Vertical Slice (Application/Features/) | Depends On | Design Source |
|---|---|---|---|---|
| AIA-6 | Metric/entity/dimension/endpoint registry as versioned, in-process-cached application config (§6/§7/§10.3's field lists; the nine existing endpoints + their documented parameter bounds) | `Application/Registry/MetricEndpointRegistry` | AIA-1 | `design-decisions.md` "Caching" (registry cached in-process, never a domain aggregate); `ddd-model.md` Modeling Decision 7; **excludes any Seller/Vendor or Geography dimension by design — ADR-0026 (Decision 3: no extensibility hook), ADR-0027** |
| AIA-7 | Deterministic intent→endpoint `QueryPlan` builder + second validation gate (resolved plan checked against the target endpoint's own documented contract before execution) + `RankingSpec.limit` clamp-and-disclose | `Application/Features/QueryAssistant/BuildQueryPlan` | AIA-6 | `ddd-model.md` `QueryPlan`/`RankingSpec` value objects; `edge-cases.md` "Intent Passes Validation but the Resolved Query Plan Is Itself Malformed," "Schema-Valid Intent Requests a Rank/Limit Analytics Cannot Fulfill"; FR-002 |
| AIA-8 | Execute a resolved `QueryPlan` against `kart-analytics-service`'s nine existing dashboards + order-conversion funnel (OAuth2 Client-Credentials, `analytics.dashboards.read`) | `Application/Features/QueryAssistant/ExecuteQueryPlan` | AIA-7 | FR-003; `architecture.md` Dependencies table; `requirement-spec.md` §5 API Surface (existing endpoints row) |

---

## Phase 3 — Product performance capability

**Source spec scope (verbatim, §24 Phase 3):** "the new `product_performance_dashboard` read model + endpoint (§10.3), **owned and built by the Analytics team**." Per this task's own instruction, the actual read-model/projector/endpoint implementation is **not** duplicated here — it belongs in `kart-analytics-service/tickets.md`, under that service's own backlog. This phase is represented here only as the cross-repo dependency it creates for this service's own registry/mapping work.

| ID | Task | Vertical Slice (Application/Features/) | Depends On | Design Source |
|---|---|---|---|---|
| AIA-9 | Add the `product-performance` endpoint to this service's registry (AIA-6) and intent→endpoint mapping table (AIA-7) once it ships from Analytics; wire the ranking-query intent shape (§8's `ranking`/`RankingSpec`) end-to-end against it | `Application/Registry/ProductPerformanceEndpointMapping` | AIA-6, AIA-7, **XREPO-ANL-1** | `requirement-spec.md` §5 (`GET /internal/v1/dashboards/product-performance`, "new — owned and built by the Analytics team"); source spec §10.3, §25-D4 |

**Cross-repo dependency ticket — `XREPO-ANL-1` (kart-analytics-service, not this repo):**
- **What:** Build the new `product_performance_dashboard` MongoDB read model, its projector, and `GET /internal/v1/dashboards/product-performance` — ranked/aggregated product list by revenue, units sold, or order count over a window, following that service's own existing dashboard-addition pattern.
- **Owner:** `kart-analytics-service`'s own team/backlog, now tracked as `ANL-16` through `ANL-20` under that service's own "Product Performance Addition (v1.1)" epic (`kart-analytics-service/tickets.md`): `ANL-16` (collection + schema validation), `ANL-17` (projector), `ANL-18` (indexes), `ANL-19` (endpoint + contract tests), `ANL-20` (ranking-tie/reconciliation-reorder handling). This entry previously predated that epic's own ticket-agent pass; updated now that it exists.
- **Blocks:** AIA-9 here, and transitively every ranking-query worked example (source spec §23 Example 1) in Phase 4/5's acceptance tests. Specifically gated on `ANL-19`/`ANL-20`, per that service's own dependency chain.
- **Basis:** source spec §25-D4 ("Build one new Analytics read model... rather than compute rankings client-side or defer the capability entirely" — explicitly rejects `kart-ai-assistant-service` computing rankings itself as "a second computer of Analytics' own aggregates from raw data it was never meant to own," ADR-0024's ownership boundary applied by analogy); §24 Phase 3's own "Dependencies: none blocking" note refers to *Analytics'* own build having no blocking prerequisite (`OrderCreated.items`'s shape is already confirmed) — it is this service's *own* Phase 4 work that is sequenced behind Analytics shipping it, not the other way around.
- **Reference:** `kart-analytics-service/tickets.md`, "Product Performance Addition (v1.1)" section, which cross-references this ticket back as `XREPO-AIA-1`.

---

## Phase 4 — AI integration & conversation

**Source spec scope:** clarification handling (§15), follow-up/context-carrying (§14), grounding validation (FR-004, §16) — the LLM-touching half of the turn, layered on top of Phase 2/3's deterministic machinery. **Exit criterion:** the four worked examples in §23 all pass as acceptance tests.

| ID | Task | Vertical Slice (Application/Features/) | Depends On | Design Source |
|---|---|---|---|---|
| AIA-10 | NL→intent translation: structured-output "plan" call (single-turn, no prior context yet) — schema-validated against §8's `ResolvedIntent` shape, closed metric/entity/dimension enums only | `Application/Features/QueryAssistant/TranslateQuestionToIntent` | AIA-3, AIA-7 | FR-001, FR-002; source spec §11/§12/§21.4 (D2, D3); `api-contract.yaml` `ResolvedIntent` schema (closed enums, **no `geography`/`seller`/`vendor` value — ADR-0026, ADR-0027**) |
| AIA-11 | Follow-up context merge (FR-007: unaddressed fields carry forward unchanged; low-confidence refinement-vs-reset triggers a clarification) + stale cross-turn provenance disclosure | `Application/Features/QueryAssistant/MergeFollowUpIntent` | AIA-10, AIA-4 | FR-007; `edge-cases.md` "Follow-up Ambiguous Between Refinement and Topic Change," "Follow-up Against Data That Has Since Changed"; `ddd-model.md` `TurnProvenance`, Cross-Aggregate Interaction step 1/4 |
| AIA-12 | Ambiguity detection & clarification (FR-008: metric-superlative hard-stop trigger; unresolvable entity/time-phrase fuzzy-match-and-suggest) | `Application/Features/QueryAssistant/DetectAmbiguityAndClarify` | AIA-10 | FR-008; `edge-cases.md` "Ambiguous Entity or Time Reference (Beyond the No-Metric Superlative Case)"; `api-contract.yaml` `AssistantClarificationResponse` |
| AIA-13 | Grounded explanation generation (explain call) + numeric-grounding validation (format-normalized literal match against `data`, all derived figures pre-computed application-side) + template-fallback on grounding failure | `Application/Features/QueryAssistant/GenerateGroundedExplanation` | AIA-8, AIA-3 | FR-004, §16; `edge-cases.md` "Grounding-Check Failure, False Positive, and False Negative" |
| AIA-14 | "Unsupported" vs. "supported, zero rows" classification (FR-009) — registry-driven for any intent with no matching capability, including the permanently-excluded Geography/Seller-Vendor dimensions | `Application/Features/QueryAssistant/ClassifyNoDataOutcome` | AIA-6, AIA-10 | FR-009; `api-contract.yaml` `AssistantErrorResponse.error.type = unsupported`; **cites ADR-0026 (Seller/Vendor) and ADR-0027 (Geography) as the authority for the specific "not captured" message content, not merely a generic capability-gap message** |
| AIA-15 | Conversation session expiry handling — distinguishable `expired-session` error, never a silent guess against an absent prior intent | `Application/Features/QueryAssistant/HandleExpiredSession` | AIA-4, AIA-11 | `edge-cases.md` "Conversation Session Expiry Mid-Conversation"; `api-contract.yaml` `error.type = expired-session`. **Concrete idle-TTL duration is UX-1 (requirement-spec.md §8) — open, not resolved by this ticket; this ticket implements the mechanism against whatever number Product ultimately sets on AIA-4's Redis key** |
| AIA-16 | Resilience pattern: three independent circuit breakers (plan call, explain call, Analytics call), each with a bounded single retry and its own already-specified fallback | `Infrastructure/Resilience/CircuitBreakerPolicies` | AIA-8, AIA-10, AIA-13 | `design-decisions.md` "Resilience Pattern for LLM-Provider and Analytics Outages"; `edge-cases.md` "LLM Provider Unavailable/Timeout," "`kart-analytics-service` Unavailable/Timeout." **Concrete per-call timeout budgets are carved from the still-open latency SLA — see `SPIKE-1` below; this ticket implements the mechanism, not the numbers** |

---

## Phase 5 — Visualization

**Source spec scope:** the response contract (§13.2) fully wired into `kart-admin-web`'s new "AI Assistant" feature area; chart rendering per §13.1's rule table. **Exit criterion:** every query type renders with its recommended visualization; `table_only` fallback verified.

| ID | Task | Vertical Slice (Application/Features/) | Depends On | Design Source |
|---|---|---|---|---|
| AIA-17 | Deterministic visualization-type selection (§13.1 rule table; LLM's advisory `visualizationHint` overridden except on an explicit user chart-type request) + full response-envelope assembly (`answer`/`data`/`visualization`/`metadata`, table never omitted) | `Application/Features/QueryAssistant/AssembleAssistantResponse` | AIA-13 | FR-005, FR-006; `api-contract.yaml` `AssistantAnswerResponse`, `AssistantVisualization`; `edge-cases.md` "LLM `visualizationHint` Conflicts With the Deterministic Rule Table" (already-resolved, no new mechanism, only the response-assembly step applying it) |

**Cross-repo dependency ticket — `XREPO-WEB-1` (kart-admin-web, not this repo):**
- **What:** New "AI Assistant" feature area in `kart-admin-web` — chat-style intake UI, table + chart rendering per the `AssistantAnswerResponse`/`AssistantClarificationResponse`/`AssistantErrorResponse` discriminated-union contract, following the platform's `dataviz` skill guidance for chart-design consistency with the rest of the admin design system.
- **Owner:** `kart-admin-web`'s own team/backlog — not tracked as a ticket in this file beyond this blocker entry, per this task's own instruction to reference by name/phase rather than invent a ticket ID in a repo this document has no visibility into.
- **Blocks:** nothing on this service's own critical path (this service's contract is already fully specified in `api-contract.yaml` independent of the UI consuming it) — but Phase 5's own stated exit criterion ("every query type renders with its recommended visualization") cannot be demonstrated end-to-end until this ships, and Phase 6's security-review pass should include this UI surface once available.
- **Basis:** source spec §24 Phase 5 ("Deliverables: table + chart UI... Dependencies: Phase 2"); `requirement-spec.md`'s own framing of `kart-admin-web` as the sole surfacing surface for this capability (§1).
- **Reference:** see `kart-admin-web/tickets.md` once that service's own backlog is updated to include this feature area.

---

## Phase 6 — Security & observability hardening

**Source spec scope:** full audit logging (§20); the new `ai-assistant.query` scope's concrete issuance mechanism (needs resolution — closed by ADR-0025, but the *Identity-side config change* is still real, cross-repo work); prompt-injection test pass; rate limiting on the new endpoint. **Exit criterion:** security review passes; every acceptance criterion in §22 is demonstrable end-to-end.

| ID | Task | Vertical Slice (Application/Features/) | Depends On | Design Source |
|---|---|---|---|---|
| AIA-18 | `AuditRecord` write path — exactly one row per turn (`ai_assistant_audit_records`), covering every outcome (successful, clarification, unsupported, unavailable, expired-session) with no field beyond §20's own list (`resultSize` a count only, no raw LLM payload) | `Application/Features/QueryAssistant/WriteAuditRecord` | AIA-10, AIA-11, AIA-12, AIA-13, AIA-14, AIA-15 | FR-011; `ddd-model.md` `AuditRecord` aggregate; `database-design.md` `ai_assistant_audit_records` DDL + indexes |
| AIA-19 | Prompt-injection containment verification & test pass — confirm the explain-call prompt template treats fetched Analytics data as inert, delimited content never re-parsed as instructions; confirm the plan call cannot emit an unregistered metric/entity/endpoint even under adversarial input | *Testing —* `Application/Security/PromptInjectionContainmentTests` | AIA-10, AIA-13 | `edge-cases.md` "Prompt Injection via User Question or Reflected Tool-Result Data"; source spec §17; §24 Phase 6 "prompt-injection test pass" |
| AIA-20 | Rate limiting on `POST /v1/ai-assistant/query`, mirroring the Gateway's existing tiered token-bucket approach | `Application/Security/RateLimiting` (Gateway-side route config, applied within this service's own Gateway route registration from AIA-1) | AIA-5 | source spec §24 Phase 6 ("rate limiting... mirroring the Gateway's existing tiered token-bucket approach") |
| AIA-21 | Security review pass (this repo's own `security-review` skill) against the full built service | *Process —* no feature folder; a review-and-remediate pass | AIA-2, AIA-18, AIA-19, AIA-20 | source spec §24 Phase 6 deliverable; requires `XREPO-ID-1` to be live for full E2E RBAC verification (see below) |

**Cross-repo dependency ticket — `XREPO-ID-1` (kart-identity-service, not this repo):**
- **What:** Add `ai-assistant.query` to the role→scope mapping config that resolves both `Admin` and `Support Agent` roles at JWT-mint time (Identity's existing role-resolution step) — a config addition to an already-approved mechanism, not a new mechanism, no new `kart-identity-service` ADR or contract change required.
- **Owner:** `kart-identity-service`'s own team/backlog — not tracked as a ticket in this file beyond this blocker entry.
- **Blocks:** full end-to-end verification of AIA-2 (this service can build/unit-test its own scope-check middleware against a hand-crafted test JWT without this, but cannot demonstrate the real, Identity-issued token path until this ships) and AIA-21's security review sign-off (a review of RBAC enforcement is incomplete if the scope this service checks for is never actually issuable by Identity).
- **Basis:** ADR-0025 ("`kart-identity-service`'s own docs (`requirement-spec.md`, and its role→scope mapping config) gain one line item: `Admin` and `Support Agent` roles both resolve to include `ai-assistant.query` in the minted JWT's `scopes` claim... a config addition to an already-approved mechanism").
- **Reference:** see `kart-identity-service/tickets.md` (or equivalent config-change tracking) once that service's own backlog reflects this addition.

---

## Phase 7 — Production hardening

**Source spec scope:** load testing at realistic internal-tool concurrency; cost monitoring/alerting on LLM token spend; latency SLA sign-off. **Exit criterion:** capability is GA for `Admin`/`Support Agent` users inside `kart-admin-web`.

| ID | Task | Vertical Slice (Application/Features/) | Depends On | Design Source |
|---|---|---|---|---|
| AIA-22 | Load test `POST /v1/ai-assistant/query` at a handful-to-low-hundreds concurrent-user profile (matching `kart-admin-web`'s own stated internal-tool population) | *Testing —* `Testing/LoadTest/AiAssistantQueryLoadTest` | AIA-21 | requirement-spec.md §3 Scalability NFR; `database-design.md` "Partitioning/Sharding" (volume basis); source spec §24 Phase 7 |
| AIA-23 | Cost monitoring/alerting dashboards (Grafana, per-turn `llmTokenUsage` from `AuditRecord`) | `Observability/Dashboards/LlmCostMonitoring` | AIA-18, **SPIKE-2** | requirement-spec.md §3 Cost NFR row; `design-decisions.md` "Observability & Instrumentation." **Cannot alert meaningfully until a budget/ceiling number exists — see `SPIKE-2`** |
| AIA-24 | Latency SLA dashboards/alerts (end-to-end turn latency, per-call-site circuit-breaker state panel) | `Observability/Dashboards/TurnLatencyMonitoring` | AIA-16, **SPIKE-1** | requirement-spec.md §3 Latency NFR row (OPEN QUESTION); `design-decisions.md` "Observability & Instrumentation" (circuit-breaker-state panel). **Cannot alert meaningfully until an SLA number exists — see `SPIKE-1`** |

### Spike/Decision Tickets (Product Owner — not engineering work)

These correspond one-to-one to requirement-spec.md §8's Infrastructure-1 open item, which explicitly states both must be resolved "before Phase 7 (production hardening) of the source spec's own roadmap is considered complete." Neither is a code change; both are product/business decisions this pipeline stage is not authorized to make on its own, and both are assigned to a product owner, not an engineer.

| ID | Decision Needed | Assigned To | Blocks | Basis |
|---|---|---|---|---|
| SPIKE-1 | Set the end-to-end turn latency SLA (intake → response) — no BRD or ADR figure exists for a GenAI-inclusive round-trip; the platform's existing P95/P99 figures don't bound an LLM-inclusive turn | Product owner (whoever owns this capability's production-readiness sign-off) | AIA-16's concrete per-call timeout/retry budgets; AIA-24's alert thresholds; Phase 7 close-out | requirement-spec.md §3 (Latency NFR row, OPEN QUESTION), §8 Infrastructure-1; source spec §26-Infrastructure-1 |
| SPIKE-2 | Set the LLM token-usage cost budget/ceiling, per-turn and/or aggregate — no source document states one | Product owner (same as above) | AIA-23's alert thresholds; Phase 7 close-out | requirement-spec.md §3 (Cost NFR row, OPEN QUESTION), §8 Infrastructure-1; source spec §26-NFR-2 |

**Not ticketed, flagged only (non-blocking, same category as the two spikes above but not required by this task's own scope):** Business-1 (revenue gross vs. net — Analytics' own definition, not this service's call), UX-1 (session TTL — affects AIA-4/AIA-15's concrete numbers), UX-2 (default date range — affects AIA-10's plan-call defaults), and the `ai_assistant_audit_records` retention period (`database-design.md`'s own open item, affects AIA-18's long-term storage posture). Each requires the same category of human/product-owner decision as SPIKE-1/SPIKE-2 but is not one of the two Infrastructure-1 NFRs this document was asked to ticket explicitly; each is called out inline on the specific ticket it affects above rather than given its own spike ticket.

---

## Notes for Sprint Planner Agent

- **Critical path:** AIA-1 → AIA-2/AIA-3/AIA-4 (parallel) → AIA-5 (Phase 1 exit) → AIA-6 → AIA-7 → AIA-8 (Phase 2) → AIA-10 → {AIA-11, AIA-12, AIA-14 in parallel} → AIA-13 → AIA-16 → AIA-17 (Phase 5) → AIA-18/19/20 (parallel) → AIA-21 (Phase 6 exit) → AIA-22 (Phase 7). Longest chain is roughly 12 tickets deep; most of Phase 4's tickets (AIA-11, AIA-12, AIA-14, AIA-15) fan out from AIA-10 and can be parallelized across engineers once it lands.
- **AIA-9 (Phase 3) is genuinely blocked, not just sequenced** — it cannot start implementation until `XREPO-ANL-1` ships from `kart-analytics-service`'s own team. Do not schedule an engineer against AIA-9 before confirming that cross-repo ticket's own status; everything else in Phases 1–2 and most of Phase 4 (AIA-10 through AIA-16, using the nine *existing* endpoints only) can proceed without it — only the ranking-query worked example (§23 Example 1) is gated on it.
- **XREPO-ID-1 is a small, low-risk cross-repo dependency** (a config-only addition to an already-approved Identity mechanism, ADR-0025) — recommend filing it early even though it only blocks *full E2E verification* of AIA-2/AIA-21, not their initial implementation.
- **SPIKE-1 and SPIKE-2 should be raised with the product owner at the start of Phase 4**, not deferred until Phase 7 — AIA-16's resilience-pattern timeout budgets (Phase 4) and AIA-4/AIA-15's session-TTL numbers already need *a* number to implement against concretely, even though their absence doesn't block building the mechanism itself; waiting until Phase 7 to ask risks a late rework of already-shipped timeout constants.
- **No ticket exists anywhere in this file for Seller/Vendor or Geographic analytics**, by design (ADR-0026, ADR-0027 — both permanently closed, not deferred). If a future planning pass is tempted to add a "future work" placeholder for either, that would be reopening a decision this pipeline stage has no authority to reopen — raise a new ADR instead, per ADR-0026's and ADR-0027's own Consequences sections.
- **No ticket exists for messaging/event-bus setup** — `event-contract.md` confirms zero published and zero consumed events; there is no queue, exchange, or DLQ for this service to own.

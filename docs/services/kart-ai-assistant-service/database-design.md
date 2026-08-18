---
doc_type: database-design
service: kart-ai-assistant-service
status: approved
approval_note: >
  Approved to proceed per the same explicit human pipeline directive already
  recorded in requirement-spec.md/ddd-model.md/design-decisions.md/
  architecture.md's own approval_notes for this capability's commissioning
  task. Per the Database Design Agent's own approval rule, the read/cache
  model (Redis, ConversationSession) is approved directly — its shape was
  already fixed by design-decisions.md/ddd-model.md, nothing new is decided
  here beyond documenting it. The write-model schema
  (`ai_assistant_audit_records`) still carries its own, narrower sign-off
  item below, since a write-model schema is costly to change post-migration
  — that item is not covered by the blanket pipeline directive and is left
  genuinely open for a human DBA/reviewer pass, consistent with every other
  service's database-design.md on this platform reserving its own explicit
  "write-model schema" checkbox regardless of the doc's overall status.
generated_by: database-design-agent
source:
  - docs/services/kart-ai-assistant-service/requirement-spec.md
  - docs/services/kart-ai-assistant-service/ddd-model.md
  - docs/services/kart-ai-assistant-service/design-decisions.md
  - docs/services/kart-ai-assistant-service/architecture.md
  - docs/requirements/genai-business-assistant-spec.md (§17, §18, §20)
  - docs/adr/0024-ai-assistant-service-scope-and-integration.md
  - docs/adr/0025-ai-assistant-query-scope.md
  - docs/services/kart-analytics-service/database-design.md (structural analog — CQRS Postgres+Mongo split, PII-classification section format)
  - docs/services/kart-admin-service/database-design.md (`admin_actions`/`AdminActionPerformed` — direct audit-trail precedent, FR-011)
  - docs/requirements/kart-requirements.md §24 (RLS, sensitive-column, and created_by/updated_by conventions)
---

# Database Design: kart-ai-assistant-service

This service owns exactly two local aggregates, closed by ADR-0024 and confirmed unchanged by `ddd-model.md`: **`ConversationSession`** (ephemeral, Redis, TTL-bound) and **`AuditRecord`** (durable, append-only, one row per turn). Nothing else is designed here — no analytics-shaped read model, no copy of Order/Product/Revenue data, per ADR-0024's ownership table. `ConversationSession`'s storage technology (Redis) was already decided and closed at the design-decisions.md stage, not re-opened here; this document documents its concrete key/value shape as the read/cache-model half of this stage's output, and designs the write-model schema for `AuditRecord` from scratch, since — per `ddd-model.md`'s own framing — "the Database Design Agent runs after this document, not before it," with no pre-existing schema to reconcile against.

## Storage Topology Decision: Single PostgreSQL Table, Not a Postgres-Write/MongoDB-Read Split

**Decision: `AuditRecord` is backed by a single PostgreSQL table (`ai_assistant_audit_records`), source of truth and only queryable store. No MongoDB (or any second read-optimized store) is introduced for this service.** This is a deliberate departure from `kart-analytics-service/database-design.md`'s own two-database pattern (PostgreSQL raw-event write model + MongoDB per-dashboard read model) — reasoned explicitly here, not copied reflexively, per this stage's own instructions.

**Why Analytics needs the CQRS split and this service doesn't — the read/write profile is materially different, not just smaller:**

- **Analytics' read side exists to serve a sub-5-second dashboard-query budget (P95<2s/P99<5s, `kart-analytics-service/architecture.md`) against a fan-in of every platform event, aggregated across arbitrary `(granularity, bucketStart, sku?, category?, channel?)` combinations.** That shape — pre-aggregated documents computed by two continuously-running projection consumers (near-real-time + nightly reconciler), queried at high fan-out by ten dashboards/funnels — is exactly what a document-oriented read model earns its keep on. `kart-ai-assistant-service` produces no comparable read-heavy, latency-budgeted query surface: its NFR table (requirement-spec.md §3) states this service's own consistency posture as "not applicable to a write path — this service has none [beyond its own two local aggregates]," and its Availability/Latency rows exist to describe the *inbound* turn-processing path (dominated by LLM/Analytics call latency), not an *audit-query* path with its own SLA. No document states an audit-query latency budget at all — because none is needed at this table's actual access pattern (see below).
- **`AuditRecord`'s stated query pattern (`ddd-model.md`'s own words) is "queried by `turnId`, by `conversationId` ... or by `userId`/date range for compliance audit."** This is precisely the same shape `kart-admin-service`'s `admin_actions` table already serves — for the same reason: a compliance/audit-trail lookup ("what did this turn do," "reconstruct this conversation," "what did this user ask last month") is an occasional, low-QPS, point-or-range query against an append-only log, not a high-fan-out aggregate query recomputed continuously from a raw event stream. `kart-admin-service/database-design.md` reaches the identical conclusion for the identical reason: "Admin has no read-heavy, latency-budgeted query path of its own... No read model (MongoDB/Redis) is introduced." `AuditRecord`'s access pattern is a direct structural match to `admin_actions`, not to `analytics_raw_events`/the ten dashboard collections — the comparable precedent this task explicitly asks to be checked bears this out.
- **There is nothing here to project.** MongoDB's role in Analytics is to hold *pre-aggregated, incrementally-recomputed* documents derived from a much larger raw-event stream, because computing a revenue-by-SKU-by-day rollup per request would blow the dashboard latency budget. An `AuditRecord` row is already the finished, final shape of exactly one turn — there is no aggregation, rollup, or bucketing step between "what got written" and "what a compliance query needs to read." Introducing a second, denormalized store here would be maintaining a projection of a table that is already the answer, which is the definition of unwarranted CQRS complexity for this profile, not a defensible split.
- **Volume confirms this is not a close call.** `genai-business-assistant-spec.md` §18 (Phase 7 scope note) sizes this service at "a handful to low hundreds of concurrent users" — an internal back-office tool, the same population `kart-admin-service`'s own back-office `admin_actions` table serves, and smaller in per-turn throughput than even that: every turn is additionally rate-limited by two sequential LLM round-trips plus one Analytics call (architecture.md, "LLM call latency, not this service's own compute, is the natural rate-limiting factor"), which bounds this table's write rate well below `admin_actions`' own human-paced write rate, let alone `analytics_raw_events`' full-platform event fan-in. A single, indexed PostgreSQL table comfortably serves point/range lookups at this volume with no aggregation layer required.

**Conclusion:** the CQRS Postgres-write + MongoDB-read split is the right tool for Analytics' actual profile (continuous fan-in, pre-aggregated high-fan-out dashboard reads under a hard latency SLA) and the wrong tool here (append-only audit writes, occasional point/range compliance reads, no latency SLA stated or needed). Defaulting to copying Analytics' two-database shape would add an unrebuildable second store with no query it's actually needed for — the opposite of "no speculative index/store." This document instead follows the closer structural precedent, `kart-admin-service`'s single-table `admin_actions` design, adapted to this service's own field list (`ddd-model.md` §20).

**Not a violation of the platform's read-model rule either way.** `ddd-cqrs-standards.md`'s "a read model must always be rebuildable from the write model + event log, never written to outside a projection consumer" describes the relationship between an aggregate's own write-side table and a *derived* read-side projection of that same data. `ConversationSession` (Redis) is not a projection of `AuditRecord` (PostgreSQL) in that sense — `ddd-model.md`'s Cross-Aggregate Interaction section is explicit that "`ConversationSession` never reads or references `AuditRecord`, and `AuditRecord` never reads `ConversationSession` at write time" — they are two independent aggregates in two independent stores, not a write-model/read-model pair. Redis here plays the role Redis plays for `kart-cart-service`/`kart-identity-service`'s own ephemeral state (an aggregate's own primary store, because its data has no durability requirement, not a cache-of/projection-from a durable table) — not the role MongoDB plays for Analytics.

## Write Model (PostgreSQL): `AuditRecord`

One row per turn, written once, in the same synchronous request-handling flow that processed it (never via an async consumer/outbox, per design-decisions.md's "Communication Style" decision — "the audit record ... is written directly to this service's own local append-only store, never published as a platform event"). Field list is `ddd-model.md`'s `AuditRecord` aggregate, one-to-one, with no field invented beyond what that document (source spec §20) already states.

```sql
-- AuditRecord — the durable, append-only, one-per-turn record FR-011 requires (source spec §20).
-- Written synchronously in the same request that processed the turn, by this service's own
-- application role — the acting human (user_id) is this table's own BRD §24.3 `created_by`
-- equivalent (the same treatment kart-admin-service's admin_actions.admin_id already gets, since
-- both tables are written synchronously within the same request the named principal authorized —
-- unlike analytics_raw_events, which is written by an async ingestion consumer on someone else's
-- behalf and so needs a "system:..." pseudo-principal instead). `timestamp` is the `created_at`
-- equivalent, kept under its existing domain name (§20) rather than duplicated.
CREATE TABLE ai_assistant_audit_records (
    turn_id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Opaque reference to ConversationSession's key (Redis) — never a foreign key (ddd-model.md's
    -- own invariant: a dangling conversation_id referencing an expired/evicted Redis session is the
    -- expected, permanent steady state for every row once enough time has passed, not a data-
    -- integrity error). NULL only if a turn is rejected before a conversationId is ever assigned
    -- (should not occur per §21.1's contract, which always issues one on the first turn; nullable
    -- defensively rather than assumed).
    conversation_id         UUID NOT NULL,

    user_id                 TEXT NOT NULL,          -- Identity-issued principal id (JWT `sub`), reference only — this service never models Identity's own state
    role                    TEXT NOT NULL CHECK (role IN ('Admin', 'Support Agent')),

    user_question           TEXT NOT NULL CHECK (char_length(user_question) <= 1000), -- raw text, FR-001's own ≤1000-char input bound enforced here too, not just at the API boundary

    resolved_intent         JSONB NULL,             -- ResolvedIntent value object (§8 schema), NULL if the turn never reached a resolved intent (e.g. a hard pre-translation rejection)
    clarification_issued    BOOLEAN NOT NULL DEFAULT false,
    clarification_detail    JSONB NULL,             -- ClarificationDetail {question, options}; present iff clarification_issued

    tools_invoked           TEXT[] NOT NULL DEFAULT '{}', -- endpoint(s) called this turn, e.g. '{"GET /internal/v1/dashboards/product-performance"}' — a list since a comparison issues two calls (FR-002/§5)
    data_source             TEXT NULL,              -- constant 'kart-analytics-service' whenever a query executed, else NULL
    query_parameters        JSONB NULL,              -- the realized from/to/granularity/filters actually sent (that turn's QueryPlan.parameters) — never the rows/columns Analytics returned

    execution_time_ms       INT NULL,               -- Analytics call latency
    llm_latency             JSONB NOT NULL DEFAULT '{}', -- LlmLatency {planCallMs, explainCallMs} — explainCallMs absent if the plan call itself failed and explain never ran
    llm_token_usage         JSONB NOT NULL DEFAULT '{}', -- LlmTokenUsage {plan:{promptTokens,completionTokens}, explain:{...}}

    result_size             INT NULL,               -- a row COUNT only, never a copy of the rows (ddd-model.md's own field-level enforcement of ADR-0024's ownership boundary) — NULL when no query executed
    turn_provenance         JSONB NULL,              -- TurnProvenance {isProvisional, reconciledThrough} — present iff the underlying result carried it (FR-012)

    grounding_check_result  TEXT NULL CHECK (grounding_check_result IN ('pass', 'fail')), -- FR-004; NULL if no explain call happened at all (e.g. turn ended in a hard error before generation)

    -- ErrorDetail[] — {category, message}; category is the closed FR-009/§19 enum as extended by
    -- edge-cases.md (a plan-vs-explain sub-classification under `unavailable`, plus `expired-session`).
    -- Deliberately NOT a CHECK-constrained scalar column here: `category`'s exact string values are
    -- the API Design Agent's own error-type enum (requirement-spec.md §5, "Final contract detail...
    -- is the API Design Agent's job, not this spec's"), which is being authored in parallel with this
    -- document (both are fifth-stage-in-parallel per PLATFORM_BLUEPRINT.md §8.2) — hardcoding a second,
    -- possibly-diverging enum here ahead of that contract would create two competing sources of truth
    -- for the same closed set. Validated application-side against whatever that contract settles on.
    errors                  JSONB NOT NULL DEFAULT '[]',

    final_response_summary  JSONB NULL,              -- FinalResponseSummary {answerText, visualizationType} — bounded, never the full raw LLM payload (§20)

    "timestamp"              TIMESTAMPTZ NOT NULL DEFAULT now() -- this table's BRD §24.3 created_at equivalent
);

-- turnId lookup is the primary key itself — no additional index needed for FR-011's "queryable by turnId."

-- "Reconstruct this conversation's full transcript, in order" — ddd-model.md's own stated query
-- pattern (Cross-Aggregate Interaction: "a later, out-of-band audit query joining on conversationId
-- ... ordered by timestamp").
CREATE INDEX idx_ai_assistant_audit_records_conversation
    ON ai_assistant_audit_records (conversation_id, "timestamp");

-- "What did this user ask, over what window" — the compliance-review pattern FR-011 explicitly
-- names, direct structural analog of kart-admin-service's idx_admin_actions_admin_category
-- (admin_id, category, performed_at), simplified here since this table has no second dimension
-- comparable to Admin's five-way `category` (ai-assistant.query is a single action, ADR-0025).
CREATE INDEX idx_ai_assistant_audit_records_user
    ON ai_assistant_audit_records (user_id, "timestamp");

-- Plain date-range compliance sweep with no user_id filter (e.g. "every turn in the last 30 days,
-- across all askers") — FR-011 states "queryable... by userId, date range" as two independently
-- listed dimensions, not date-range-only-as-a-refinement-of-user. The (user_id, timestamp) index
-- above cannot serve a user-agnostic range scan without a full-index or full-table scan (user_id is
-- its leading column); this index closes that gap explicitly rather than leaving it unsupported the
-- way kart-admin-service's own admin_actions design does for the equivalent query (its
-- idx_admin_actions_admin_category is likewise admin_id-first only) — self-critiqued against here
-- rather than silently copied.
CREATE INDEX idx_ai_assistant_audit_records_timestamp
    ON ai_assistant_audit_records ("timestamp");
```

No GIN/JSONB-path indexes are added on `resolved_intent`, `query_parameters`, or `errors` — no stated query pattern (FR-011, `ddd-model.md`) ever filters by intent field, query parameter, or error category as a standalone lookup; the only documented access patterns are by `turnId`, `conversationId`, `userId`, and date range, all covered above. Adding a speculative JSONB index for a filter no source document asks for would be exactly the "no speculative indexes" case this stage's own rule forbids.

## Cache / Ephemeral-State Model (Redis): `ConversationSession`

Technology is already closed (design-decisions.md's "Conversation-Session Storage" decision) — Redis, not re-litigated here. This section documents the concrete key/value shape only.

```text
Key:   ai-assistant:session:{conversationId}
Type:  String, holding one JSON-serialized ConversationSession value (not a Hash)
Value: {
         "conversationId":      "<uuid>",
         "lastResolvedIntent":  <ResolvedIntent JSON | null>,
         "lastTurnProvenance":  { "isProvisional": <bool>, "reconciledThrough": "<ts>" } | null,
         "lastActivityAt":      "<ISO-8601 timestamp>"
       }
TTL:   Redis native key expiry (EXPIRE), refreshed (sliding) on every turn that touches this key.
       Concrete duration = UX-1 (requirement-spec.md §8) — genuinely open, not set here or anywhere
       downstream of it; this design fixes only the mechanism (native TTL + sliding refresh on write).

Key:   ai-assistant:session-lock:{conversationId}
Type:  String, `SET ... NX PX <lockTimeoutMs>` (design-decisions.md's "Concurrency Control" decision)
Value: an opaque lock token (compare-and-delete on release, avoiding one turn releasing a lock a
       later turn already re-acquired after the first's own PX expiry)
TTL:   A short safety-net expiry bounding "how long can one turn legitimately hold this lock" —
       derived from whatever the end-to-end turn latency SLA (Infrastructure-1, requirement-spec.md
       §8, also still open) ultimately resolves to, plus margin; not invented as a number here for
       the same reason Infrastructure-1 isn't given one anywhere else in this service's docs.
```

**Why a String holding a whole-value JSON blob, not a Hash with per-field `HSET`:** `ddd-model.md`'s own `ResolvedIntent`-as-value-object reasoning is explicit that a follow-up turn "doesn't edit fields on an existing `ResolvedIntent` instance... always produces a brand-new intent value... naturally replaced wholesale, not mutated in place." A Redis Hash invites exactly the per-field mutation this aggregate's own invariant forbids (a caller `HSET`-ing one field while leaving `lastTurnProvenance` stale, for instance); a single `GET`/`SET` of one JSON blob is the storage-level enforcement of "the whole session value is replaced atomically, once per turn, or not at all" — matching how the aggregate is actually written in step 4 of `ddd-model.md`'s Cross-Aggregate Interaction ("Upsert `ConversationSession`... overwriting `lastResolvedIntent`/`lastTurnProvenance` wholesale").

**No schema migration concerns.** Unlike the PostgreSQL table above, this value has no DDL to version — a shape change here is an application-code deploy (the JSON envelope's own version, if ever needed, is an application concern, not a database-design one), consistent with why this half of the design carries no separate "write-model schema" sign-off gate (see frontmatter).

## Row-Level Security Policy (BRD §24.1.4)

Per BRD §24.1.4, the service whose database physically holds a row decides and enforces that row's row-level security. `ai_assistant_audit_records` is this service's one PostgreSQL table; `ConversationSession`'s Redis key has no RLS-equivalent concept (Redis has no native row-level policy mechanism, and design-decisions.md already scopes protection there to key-namespacing plus this service's own exclusive access to the shared Redis deployment).

**No RLS policy is added to `ai_assistant_audit_records`, for the same reason `kart-admin-service`'s `admin_actions` carries none.** `user_id` identifies the *asker* whose turn a row describes, not a caller who is meant to see only their own rows: an audit/compliance review of this trail (mirroring `GET /admin/actions`'s own stated purpose — "any admin must be able to review any other admin's actions") structurally requires cross-user visibility. A `user_id = current_setting('app.current_principal')` policy would contradict the audit trail's own reason for existing (FR-011: "queryable the same way `kart-admin-service`'s `AdminActionPerformed` audit trail already is"), the identical reasoning `kart-admin-service/database-design.md` gives for leaving `admin_actions` unrestricted by row-ownership.

**What actually protects this table, in the absence of a row-ownership policy:** (1) there is no public API Gateway route directly onto this table — whatever audit-query endpoint the API Design Agent defines sits behind its own coarse-role/scope check first, the same layering `admin_actions` already uses; (2) direct database access is restricted to this service's own application role via native `GRANT`, excluded from any other database principal, mirroring both sibling services' precedent for tables with no per-row ownership concept.

## Sensitive / PII Column Classification (BRD §24.1.5)

| Column | Full Value Visible To | Masked/Omitted For | Masking Rule |
|---|---|---|---|
| `ai_assistant_audit_records.user_question` | Any caller authorized to read this service's audit trail at all (whatever coarse-role/scope check the API Design Agent's audit-query endpoint applies — mirrors `admin_actions.context`'s treatment: audit detail gated at the endpoint, not per-column) | No caller outside that authorized set — this column is never exposed by any public-facing endpoint | **Not customer PII in the query types this spec covers** — source spec §20/§17's own explicit framing: "the user's own question text *is* logged... it's operational business-question metadata, analogous to `AdminActionPerformed`'s own audit posture, not customer PII... reassess if geographic/seller analytics are ever added." Carried forward verbatim: both Geography (ADR-0027) and Seller/Vendor (ADR-0026) are *permanently* out of scope for this build, not merely deferred, so this classification is not "pending" a near-term change — but it must be revisited from scratch, not assumed still valid, if either ADR is ever formally reopened |
| `ai_assistant_audit_records.user_id` / `.role` | Same audit-trail-authorized caller set as above | No caller outside that set | Internal-staff identifiers (an `Admin`/`Support Agent` principal id and coarse role claim) — not customer PII, identical treatment to `kart-analytics-service`'s own `admin_audit_log.adminId` classification ("an internal-staff identifier, not customer PII") |
| `ai_assistant_audit_records.resolved_intent` / `.query_parameters` / `.final_response_summary` | Same audit-trail-authorized caller set | No caller outside that set | These describe *which metric/dimension/date-range the query targeted* and *the generated answer text* — both drawn exclusively from Analytics' own pre-aggregated dashboards, which source spec §17 states explicitly never carry raw, potentially-PII-bearing event data ("every data source in scope is pre-aggregated... the assistant never has access to `analytics_raw_events.payload`"). No customer PII can appear in these columns by construction of what this service is ever permitted to fetch (ADR-0024/FR-003) |

**Why this list and no more:** `turn_id`, `conversation_id`, `timestamp`, `clarification_issued`/`clarification_detail`, `tools_invoked`, `data_source`, `execution_time_ms`, `llm_latency`, `llm_token_usage`, `result_size`, `turn_provenance`, `grounding_check_result`, and `errors` are all opaque identifiers, operational/observability metadata, or counts — none of them is, or could contain, personal data about an identifiable end customer, by the same "this service never receives raw customer data in the first place" argument above. `user_question` is the one column carrying free text a human typed, which is why it — alone among this table's columns — gets its own explicit classification and forward-looking caveat, rather than being bundled into the "no more to say" bucket with the rest.

**Enforcement point:** primarily whatever response DTO the API Design Agent's audit-query endpoint defines, scoped to the same coarse-role/scope check gating that endpoint (§24.1.5's stated primary control, mirroring `admin_actions.context`'s own enforcement); secondarily, native `GRANT SELECT` restricted to this service's own application role for any direct database connection, consistent with both sibling services' precedent.

## Retention Policy — Open Question, Not Decided Here

**No source document states a retention period for `ai_assistant_audit_records`.** This is a genuine gap, not silence-as-permission: `kart-admin-service/requirement-spec.md` explicitly commits `admin_actions` to a 5-year retention window (cited directly in that service's own `database-design.md`'s Partitioning/Sharding section), and `kart-analytics-service/requirement-spec.md` §6 D3 explicitly commits `analytics_raw_events` to indefinite retention — both are stated, closed decisions this document's structural analogs could design against. Neither `genai-business-assistant-spec.md` (§18, §20, §26) nor `requirement-spec.md`/`ddd-model.md`/`design-decisions.md` for this service states a number, or even a working recommendation the way UX-1/UX-2 each get one (requirement-spec §8).

Per this pipeline stage's own standing convention (matching how UX-1's session TTL and Infrastructure-1's latency/cost budgets are carried forward rather than invented), **no retention/purge job is included in this design, and no number is assumed.** The table above has no TTL, no scheduled deletion, and no partition-based cold-tiering — the safe default in the absence of a stated policy is to retain rows (an audit trail that under-retains defeats its own FR-011 purpose; one that over-retains is a cost/compliance question, not a correctness one). This is flagged here as a new, this-stage-surfaced open question — **AuditRecord Retention Period** — requiring the same category of human/product-owner decision UX-1 and Infrastructure-1 already require before this capability is production-ready; it does not block this design from proceeding, the same way UX-1/Infrastructure-1 don't block DDD/architecture.

## Indexing Rationale

| Index | Query it supports | Why needed |
|---|---|---|
| `ai_assistant_audit_records` PK on `turn_id` | FR-011 "queryable by turnId" | Direct point lookup for "reconstruct/audit this exact turn," e.g. investigating a disputed answer (source spec §20: "any disputed answer can be reconstructed and checked") |
| `idx_ai_assistant_audit_records_conversation` `(conversation_id, timestamp)` | `ddd-model.md`'s stated pattern: "reconstruct a conversation's full transcript across turns... ordered by `timestamp`" | Without it, listing one conversation's turns degrades to a full-table scan as the append-only table grows; the compound key gives both the filter and the required ordering in one index |
| `idx_ai_assistant_audit_records_user` `(user_id, timestamp)` | FR-011 "queryable by... userId" for compliance review, direct structural analog of `kart-admin-service`'s `idx_admin_actions_admin_category` | "What did this Admin/Support Agent ask, over what window" — the same compliance pattern Admin's own audit index exists for, simplified here since this service has no `category` dimension to compound on (single `ai-assistant.query` scope, ADR-0025) |
| `idx_ai_assistant_audit_records_timestamp` `(timestamp)` | FR-011 "queryable by... date range," independent of `userId` | Self-critiqued gap-closure: FR-011 states `userId` and `date range` as two independently queryable dimensions, but the `(user_id, timestamp)` index above cannot serve a `userId`-agnostic range scan (its leading column is `user_id`) — a plain-date-range compliance sweep across all askers needs its own index, or it would silently degrade to a full scan even though the requirement explicitly names this access pattern |
| No JSONB/GIN index on `resolved_intent`, `query_parameters`, or `errors` | — | No stated query pattern filters on intent/parameter/error-category content as a standalone lookup (only `turnId`/`conversationId`/`userId`/date-range are named, all covered above) — adding one would be a speculative index this stage's own rule forbids |
| Redis `ai-assistant:session:{conversationId}` (key-based, no secondary index — Redis has none) | Every turn's session read/upsert (design-decisions.md) | Native O(1) key lookup is the entire access pattern; no query-by-field ever occurs against session state |

## Partitioning/Sharding

**Not needed at current scale, for either store.** `ai_assistant_audit_records` accumulates one row per turn in a service explicitly sized at "a handful to low hundreds of concurrent users" (`genai-business-assistant-spec.md` §18), with per-turn throughput additionally bounded by two sequential LLM round-trips and one Analytics call on the request's own critical path (architecture.md: "LLM call latency, not this service's own compute, is the natural rate-limiting factor") — a materially lower write rate than `kart-admin-service`'s own already-unpartitioned `admin_actions` table, which absorbs 5 years of human-paced back-office writes on a single table with no partitioning. A single, indexed PostgreSQL table is sufficient; the retention question above (currently open) is the one input that could change this calculus, and — mirroring `admin_actions`' own stated fallback — if retention is eventually set to a long, multi-year window and the table's size later proves unwieldy for compliance-query latency or backup/restore time, range-partitioning by `timestamp` (e.g., yearly) is the natural next step, flagged for that future point, not decided now, since no throughput or retention figure currently calls for it.

**Redis requires no sharding decision at this scale either.** One key per active conversation, evicted by TTL on idle — bounded by concurrent-conversation count (the same "handful to low hundreds" population), several orders of magnitude below any documented Redis-sharding threshold on this platform; the platform's existing shared Redis deployment (already used by `kart-cart-service`/`kart-identity-service` for a comparable ephemeral-state profile) absorbs this without a dedicated cluster or shard-key decision.

## Sign-off

- [x] Read/cache-model (Redis `ConversationSession`) — approved directly per this stage's own rule; no new decision made here beyond documenting an already-closed technology choice.
- [ ] Write-model schema (`ai_assistant_audit_records`) — reviewed by: _pending human/DBA review before migration is run._
- [ ] AuditRecord retention period — open question surfaced by this document; requires a human/product-owner decision (same category as UX-1/Infrastructure-1) before this table's storage-cost/compliance posture is production-ready. Does not block this schema from being implemented in the meantime.

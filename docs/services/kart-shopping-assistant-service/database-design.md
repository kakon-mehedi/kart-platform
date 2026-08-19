---
doc_type: database-design
service: kart-shopping-assistant-service
status: approved
approval_note: >
  Approved to proceed through the pipeline per the same commissioning
  directive already recorded in requirement-spec.md/design-decisions.md/
  architecture.md/ddd-model.md/api-contract.yaml's own approval_notes. Per
  the Database Design Agent's own approval rule, the read/cache model
  (Redis, `ShoppingAssistantSession`) is approved directly — its storage
  technology and key-namespace were already fixed by design-decisions.md/
  architecture.md/ddd-model.md; nothing new is decided here beyond
  documenting its concrete value shape and TTL mechanism. The write-model
  schema (`shopping_assistant_audit_records`,
  `shopping_assistant_idempotency_keys`) still carries its own, narrower
  sign-off item below, consistent with every other service's
  database-design.md on this platform reserving a distinct "write-model
  schema" checkbox regardless of the doc's overall status — a write-model
  schema is costly to change post-migration.
generated_by: database-design-agent
source:
  - docs/services/kart-shopping-assistant-service/requirement-spec.md (status: approved)
  - docs/services/kart-shopping-assistant-service/design-decisions.md (status: approved)
  - docs/services/kart-shopping-assistant-service/architecture.md (status: approved)
  - docs/services/kart-shopping-assistant-service/ddd-model.md (status: approved — ShoppingAssistantSession,
    ShoppingAssistantAuditRecord, ShoppingAssistantIdempotencyKey, ExternalResourceReference,
    PendingConfirmation, ResolvedShoppingIntent)
  - docs/services/kart-shopping-assistant-service/api-contract.yaml (status: approved)
  - docs/adr/0028-shopping-assistant-service-scope-and-integration.md (data-ownership ceiling — this
    service may own ONLY conversation/session state, its audit log, and idempotency keys — nothing
    else, ever)
  - docs/adr/0030-shopping-assistant-python-stack-exception.md (Postgres via SQLAlchemy, not EF Core;
    native PostgreSQL RLS gated on SET LOCAL app.current_principal, per kart_shared.db; kart_shared.
    auditing's before_flush hook for audit-field injection)
  - docs/adr/0029-shopping-assistant-scope-and-guest-access.md (guest-vs-authenticated split — informs
    the nullable principal_id / guest-pseudo-principal design below)
  - docs/services/kart-payment-service/database-design.md (idempotency_keys unique-constraint
    table-shape precedent — adapted to this service's own composite scope tuple, not copied verbatim)
  - docs/services/kart-ai-assistant-service/database-design.md (closest structural precedent for an
    audit-log-only Postgres schema — adapted, not copied: that service has no idempotency-key table
    since it never mutates, and folds its audit columns into domain-named fields rather than the
    explicit created_at/updated_at/created_by/updated_by columns ADR-0030 requires here)
  - docs/requirements/kart-requirements.md §24.1.4 (RLS mechanism), §24.3 (mandatory audit columns)
---

# Database Design: kart-shopping-assistant-service

This service owns **exactly three** local structures, closed by ADR-0028 and confirmed unchanged by `ddd-model.md`: **`ShoppingAssistantSession`** (ephemeral, Redis, TTL-bound — including the FR-004 `PendingConfirmation` sub-state), **`ShoppingAssistantAuditRecord`** (durable, append-only, one row per turn, Postgres), and **`ShoppingAssistantIdempotencyKey`** (durable, reserve-then-reuse, Postgres). Nothing else is designed here, and nothing else is designed anywhere else in this schema — **no `Order`/`Cart`/`Product`/`Coupon`/`User`-shaped table or read model of any kind, and no local cache table for downstream eligibility/price/stock data**, the same invariant `kart-ai-assistant-service/database-design.md` states for its own analogous ownership ceiling ("no analytics-shaped read model of its own"), restated here for ADR-0028's own three-structure ceiling. Every `ExternalResourceReference` this service's own tables hold is an opaque `{resourceType, resourceId}` pair only (ddd-model.md), never an embedded copy of the referenced order/cart/coupon/address/product record.

This is this service's own PostgreSQL database (database-per-service, ADR-0030) — no other service's schema is reached into, and no other service reaches into this one.

## Storage Topology Decision: Two PostgreSQL Tables, No MongoDB Read Model — Redis for Ephemeral State Only

**Decision: `ShoppingAssistantAuditRecord` and `ShoppingAssistantIdempotencyKey` are each backed by a single PostgreSQL table, source of truth and only queryable store for durable data. No MongoDB (or any second read-optimized store) is introduced.** This mirrors `kart-ai-assistant-service/database-design.md`'s own reasoning against copying `kart-analytics-service`'s CQRS Postgres-write/MongoDB-read split reflexively:

- **No stated read-heavy, latency-budgeted query path exists for either table.** `ddd-model.md`'s own stated query patterns for `ShoppingAssistantAuditRecord` are "queried by `turnId`, by `sessionId` (transcript reconstruction), or by `principalId`/date range for compliance" — occasional, low-QPS, point-or-range lookups against an append-only log, the identical shape `kart-admin-service`'s `admin_actions` and `kart-ai-assistant-service`'s `ai_assistant_audit_records` already serve with a single indexed table, not a continuously-recomputed, high-fan-out aggregate-query shape. `ShoppingAssistantIdempotencyKey`'s one query pattern (reserve-or-reuse lookup by its composite scope tuple) is a single-row equality lookup on its own primary key — the textbook case for a plain indexed table, not a projection.
- **There is nothing to project.** Both tables already hold the finished shape of the fact they exist to record (one turn's outcome; one attempted mutating action's dedup scope) — there is no aggregation/rollup/bucketing step between "what got written" and "what a later query needs to read."
- **Volume does not call for it.** This service's own NFR posture (requirement-spec.md §4, "secondary tier," Availability 99.9%) and its per-turn call-count profile (bounded by up to nine synchronous downstream calls plus LLM round trips, architecture.md's Distributed-Monolith Risk section) place a natural ceiling on write throughput per turn, well below the volume that would justify a second, denormalized store.

`ShoppingAssistantSession` (including its `PendingConfirmation` sub-state) is **not** a projection of either Postgres table — it is Redis because its data has no durability requirement past its own idle TTL (design-decisions.md's Confirmation-State-Machine decision), the same role Redis plays for `kart-cart-service`/`kart-identity-service`'s own ephemeral state, not the role MongoDB plays for Analytics. `ddd-model.md`'s Cross-Aggregate Interaction section is explicit that no aggregate here is derived from another at write time — three independent stores/consistency boundaries, coordinated only by the per-session lock, never a two-phase commit.

## Write Model (PostgreSQL): `shopping_assistant_audit_records`

One row per turn — successful, clarification, confirmation-pending, or error alike (requirement-spec.md §6's own acceptance criterion; `ddd-model.md`'s `ShoppingAssistantAuditRecord` invariant "exactly one... per turn, always"). Written synchronously in the same request that processed the turn (design-decisions.md's zero-async-edges posture; no outbox, no async consumer). Append-only, immutable once written — no field is ever updated after creation (`ddd-model.md`).

Field list is `ddd-model.md`'s `ShoppingAssistantAuditRecord` aggregate, one-to-one, plus the four mandatory audit columns ADR-0030 requires be enforced at the ORM layer, not folded into a domain-named field the way the read-only sibling's `ai_assistant_audit_records.timestamp`/`user_id` double as its own audit-field equivalents — this document keeps them as their own explicit columns, per this pipeline run's own instruction, since ADR-0030's guarantee table binds this service to the *same* audit-field guarantee every other platform service gets, not a service-specific substitute.

```sql
-- ShoppingAssistantAuditRecord (ddd-model.md) — the durable, append-only, one-per-turn record
-- requirement-spec.md §6 requires. Written synchronously by this service's own application role,
-- via SQLAlchemy (ADR-0030) — never by an async consumer.
CREATE TABLE shopping_assistant_audit_records (
    turn_id                     UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Opaque reference to ShoppingAssistantSession's Redis key — never a foreign key
    -- (ddd-model.md's own invariant: a dangling session_id referencing an already-expired/evicted
    -- Redis session is the expected, permanent steady state for every row once enough time has
    -- passed, not a data-integrity error; Redis and Postgres cannot share a referential-integrity
    -- constraint across stores in any case).
    shopping_assistant_session_id UUID NOT NULL,

    -- ShoppingAssistantSession.principalId, mirrored here (ddd-model.md's own field). NULL for a
    -- guest turn (ADR-0029's read-only-eligible anonymous caller) — never populated with a
    -- fabricated value. This is the customer-ownership column the RLS policy below reads; it is
    -- deliberately distinct from created_by/updated_by (below), which are never NULL (kart-
    -- requirements.md §24.3's "system-initiated mutations still get an actor, never NULL" —
    -- extended here to a well-known guest pseudo-principal, not only system jobs).
    principal_id                UUID NULL,

    -- ddd-model.md: (Successful | ClarificationIssued | ConfirmationPending | Error). Left as a
    -- plain CHECK, not a shared enum type, mirroring how design-decisions.md's Bounded
    -- Tool-Calling Registry decision keeps the intent catalog itself an application-layer,
    -- never-DB-enforced registry (avoiding two independently-migratable sources of truth for one
    -- closed set).
    turn_outcome                TEXT NOT NULL
                                    CHECK (turn_outcome IN ('successful', 'clarification_issued', 'confirmation_pending', 'error')),

    -- ResolvedShoppingIntent {intentName, resolvedSlots} — a frozen copy of whatever intent this
    -- turn resolved, if any (ddd-model.md). intent_name is deliberately NOT its own CHECK-
    -- constrained column here for the same registry-duplication reason above; it is validated
    -- against the closed catalog (requirement-spec.md §2) at resolution time, in application code,
    -- not re-derived at the DB layer.
    resolved_intent             JSONB NULL,

    -- ProposedActionRecord {intentName, targetResource: ExternalResourceReference,
    -- actionDescription} — present only when this turn's resolved intent is mutating.
    -- targetResource is always the opaque {resourceType, resourceId} shape (ADR-0028's ownership
    -- table) — never an embedded order/cart/coupon/address/product sub-object, and never a raw
    -- gateway_token or unmasked PCI-scoped value anywhere inside this JSON (ddd-model.md's own
    -- invariant; see Sensitive/PII Column Classification below).
    proposed_action             JSONB NULL,

    -- ConfirmationOutcome {confirmedAt, affirmed} — present iff FR-004's confirmation step was
    -- actually reached this turn.
    confirmation_outcome        JSONB NULL,

    -- ExecutionOutcome {executedAt, downstreamEndpoint, downstreamOutcome
    -- (Success|Rejected|PartialFailure), downstreamStatusCode, idempotencyKeyRef} — present only
    -- if the mutating call actually fired. idempotencyKeyRef is a value-correlation to
    -- shopping_assistant_idempotency_keys' own composite scope (below), not a foreign key —
    -- mirrors kart-payment-service's own "correlated by value, not by mandatory FK" pattern among
    -- its own three aggregates (ddd-model.md's Cross-Aggregate Interaction).
    execution_outcome           JSONB NULL,

    -- ClarificationDetail {question, options} — present iff turn_outcome = 'clarification_issued'.
    clarification_detail        JSONB NULL,

    -- ErrorDetail[] — {category, message}; zero or more per turn. category is one of:
    -- unsupported-gap, ambiguous, unauthenticated-mutating-rejected, downstream-rejected,
    -- partial-failure, injection-detected-non-actionable, expired-session,
    -- timeout-or-circuit-open (ddd-model.md's own closed enumeration) — validated
    -- application-side, not DB-CHECK-constrained, for the same reason intent_name isn't above.
    errors                      JSONB NOT NULL DEFAULT '[]',

    -- ResponseSummary {summaryText} — the natural-language text actually sent this turn (bounded
    -- length, never the full raw fetched payload), extended per edge-cases.md's "Confirmation or
    -- Summary Text Implies a Capability That Doesn't Exist" decision to also cover FR-004
    -- confirmation text, not only FR-006 summaries. Modeled as a plain bounded TEXT column, not
    -- JSONB, since the value object carries exactly one field. The 4000-char ceiling is a generous
    -- technical schema bound (double the request message's own 2000-char API-level bound,
    -- api-contract.yaml), not a business-rule figure this document invents.
    response_summary            TEXT NOT NULL
                                    CHECK (char_length(response_summary) <= 4000),

    -- GroundingCheckResult — Pass|Fail. Nullable: a turn that never reaches a generation step at
    -- all (e.g. a hard pre-LLM FR-001 rejection, or an unsupported-gap short-circuit) has no
    -- grounding check to record, mirroring ai_assistant_audit_records.grounding_check_result's own
    -- nullability reasoning.
    grounding_check_result       TEXT NULL
                                    CHECK (grounding_check_result IN ('pass', 'fail')),

    -- architecture.md's Observability requirement: end-to-end trace propagation into every
    -- downstream call.
    correlation_id               TEXT NOT NULL,

    -- Mandatory audit columns (kart-requirements.md §24.3), populated exclusively by
    -- kart_shared.auditing's SQLAlchemy `before_flush` Session-event hook (ADR-0030,
    -- design-decisions.md's "kart_shared Auditing Equivalent" decision) — never client-suppliable,
    -- never set by handler/repository code directly. The acting principal stamped here is the
    -- same resolved principal ShoppingAssistantSession.principalId carries when present, or a
    -- well-known guest pseudo-principal (e.g. 'guest:<sessionId>') when the turn was anonymous —
    -- kart-requirements.md §24.3's "system-initiated mutations still get an actor, never NULL"
    -- convention, applied here to an anonymous end-user caller rather than only a system job.
    -- Because this table is append-only, updated_at/updated_by are populated once, at insert, and
    -- never change again in practice — still present because kart-requirements.md §24.3 mandates
    -- all four columns on every mutable table platform-wide, and this table's own physical schema
    -- migration for them cannot be centralized (§24.3's own "what genuinely cannot be centralized"
    -- caveat).
    created_at                   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at                   TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by                   TEXT NOT NULL,
    updated_by                   TEXT NOT NULL
);

-- "Reconstruct this session's full transcript, in order" — ddd-model.md's own stated query
-- pattern.
CREATE INDEX idx_shopping_assistant_audit_records_session
    ON shopping_assistant_audit_records (shopping_assistant_session_id, created_at);

-- "What did this Customer ask, over what window" — the compliance-review pattern ddd-model.md
-- names ("queried by principalId/date range for compliance"), scoped to only the (non-guest) rows
-- that pattern can ever apply to — a guest turn has no principal to review a compliance trail
-- against, so a partial index (excluding NULL principal_id) avoids indexing rows this query
-- pattern structurally never targets.
CREATE INDEX idx_shopping_assistant_audit_records_principal
    ON shopping_assistant_audit_records (principal_id, created_at)
    WHERE principal_id IS NOT NULL;

-- Plain date-range sweep with no principal_id filter (e.g. "every turn in the last 30 days,
-- across all sessions, guest or authenticated") — the (principal_id, created_at) index above
-- cannot serve a principal-agnostic range scan (principal_id is its leading column, and it
-- excludes guest rows entirely by construction), so this index closes that gap explicitly, the
-- same self-critique kart-ai-assistant-service/database-design.md already applied to its own
-- equivalent pair of indexes.
CREATE INDEX idx_shopping_assistant_audit_records_created_at
    ON shopping_assistant_audit_records (created_at);
```

No GIN/JSONB-path index is added on `resolved_intent`, `proposed_action`, `execution_outcome`, `clarification_detail`, or `errors` — no stated query pattern (`ddd-model.md`, requirement-spec.md §6) ever filters by a field inside one of these JSON payloads as a standalone lookup; the only documented access patterns are by `turnId` (the primary key), `sessionId`, `principalId`, and date range, all covered above. Adding a speculative JSONB index here would be exactly the "no speculative indexes" case this stage's own rule forbids.

## Write Model (PostgreSQL): `shopping_assistant_idempotency_keys`

Backs `ShoppingAssistantIdempotencyKey` (ddd-model.md) — reserved *before* the downstream mutating call fires; the same stored `minted_key_value` is returned (never re-minted) on any retry whose scoping tuple matches. Directly mirrors `kart-payment-service`'s own `idempotency_keys` table-shape precedent (a composite natural-key `PRIMARY KEY`, not a surrogate id + separate `UNIQUE` constraint), adapted to this service's own composite scope tuple, per design-decisions.md's Idempotency-Key Persistence decision and `ddd-model.md`'s own resolution that this is its own aggregate, not a field on `ShoppingAssistantAuditRecord`.

```sql
-- ShoppingAssistantIdempotencyKey (ddd-model.md) — one row per attempted mutating action's
-- reservation scope. Inserted before the downstream call fires; read (and, if a matching scope is
-- found, reused verbatim) on every retry attempt.
CREATE TABLE shopping_assistant_idempotency_keys (
    shopping_assistant_session_id UUID NOT NULL,   -- same opaque, non-FK reference as the audit table above
    intent_name                   TEXT NOT NULL,   -- validated against the closed catalog at resolution time, not DB-CHECK-constrained (see reasoning above)
    resolved_slots_hash           TEXT NOT NULL,    -- deterministic hash of the mutating intent's resolved slots at the moment of the attempt (ddd-model.md) — never the slots themselves in cleartext on this row

    minted_key_value               TEXT NOT NULL,   -- the actual string value forwarded as the downstream Idempotency-Key header (kart-order-service/kart-cart-service/kart-offer-service)

    -- Schema-only addition (not itself a ddd-model.md-defined field on this entity) purely to
    -- support the RLS policy below with the identical ownership-scoping shape used on
    -- shopping_assistant_audit_records — see Row-Level Security Policy. NULL for a reservation
    -- made during a guest turn (though in practice every mutating intent requires an authenticated
    -- Customer per FR-001, so this column is expected to always be non-null in steady state; kept
    -- nullable defensively rather than assumed).
    principal_id                   UUID NULL,

    -- Mandatory audit columns (kart-requirements.md §24.3), same kart_shared.auditing mechanism as
    -- above. This row is never updated after insert (reuse-not-re-mint is a read, not a write), so
    -- updated_at/updated_by are populated once, identically to created_at/created_by, and never
    -- change again in practice — present for the same platform-wide mandatory-column reason noted
    -- on the audit table above.
    created_at                     TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at                     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by                     TEXT NOT NULL,
    updated_by                     TEXT NOT NULL,

    -- The DB unique constraint design-decisions.md names, realized as this table's own natural
    -- composite primary key (mirroring kart-payment-service's `PRIMARY KEY (idempotency_key,
    -- endpoint)` shape) rather than a surrogate id + separate UNIQUE — the reserve-or-reuse lookup
    -- *is* an equality match on exactly this tuple, so the primary key itself is the only index
    -- this table's one query pattern needs.
    PRIMARY KEY (shopping_assistant_session_id, intent_name, resolved_slots_hash)
);
```

No additional index is added beyond the primary key — the reserve-then-check-scope lookup (`ddd-model.md`'s own Cross-Aggregate Interaction step 3: "insert-if-absent; if a matching scope already exists, reuse its stored `mintedKeyValue`") is a single equality lookup on exactly the primary key's own three leading columns; no other query pattern is stated anywhere upstream for this table.

**Replay-window TTL/cleanup is explicitly not designed here.** `ddd-model.md`'s own invariant states "the replay-window TTL/expiry is not decided here... this document does not invent one," and design-decisions.md's "Out of Scope" list names "idempotency-key TTL/replay window" as an open figure. This table therefore carries no `expires_at` column and no scheduled deletion job today — the safe default in the absence of a stated policy is to retain rows (an idempotency guard that under-retains defeats its own dedup purpose). Once that figure is set, the natural evolution (mirroring `kart-payment-service/database-design.md`'s own `idempotency_keys` design exactly) is an `expires_at` column plus range-partitioning by `created_at` so TTL cleanup becomes an efficient partition-drop rather than a scanning `DELETE` competing with request-path writes — flagged for that future point, not decided now.

## Row-Level Security Policy (kart-requirements.md §24.1.4, ADR-0030)

Per ADR-0030's guarantee table row 3 ("more a 'confirm and configure' item than a 'build a substitute' item... Postgres RLS is enforced by Postgres itself regardless of which language's driver connects") and design-decisions.md's RLS-Equivalent Enforcement decision: a SQLAlchemy engine-level hook, wired once in `kart_shared.db`, issues `SET LOCAL app.current_principal = '<resolved-principal-id>'` as the first statement of every request-scoped transaction — the same resolved-principal value `kart_shared.auditing`'s `before_flush` hook (above) already reads to stamp `created_by`/`updated_by`. Both `shopping_assistant_audit_records` and `shopping_assistant_idempotency_keys` enable native RLS gated on this session variable.

**Concrete policy predicate.** Two structurally distinct checks are needed, not one:

1. **A write-time guard (`WITH CHECK` on `INSERT`), keyed on `created_by`** — the always-non-null actor column, populated identically to the currently-set `app.current_principal` session variable by the same auditing hook. This backstops "this service's own code can only ever insert a row attributed to the principal it currently claims to be acting as" — the database-layer analogue of kart-requirements.md §24.1.4's "even if a service's application code contains a bug... the database itself refuses to," applied here to a write rather than only a read, and covers a guest turn's well-known pseudo-principal (`created_by` is never null) just as well as an authenticated `Customer`'s.
2. **A read-time guard (`USING` on `SELECT`), keyed on `principal_id`** — the nullable, customer-ownership-specific column. This is the concrete answer to this task's own framing: **"a customer should only ever see their own audit trail if this data is ever exposed back to them."** No `GET` endpoint over either table exists in the currently-approved `api-contract.yaml` — this is a forward-looking, defense-in-depth control, designed now (per ADR-0030's own "confirm and configure" framing) so that whenever/if a future customer-facing read endpoint is added, per-customer row scoping is already enforced at the database layer, never bolted on later as an easily-forgotten application-level `WHERE` clause. A guest turn's row (`principal_id IS NULL`) is never selectable under this policy by any principal, by construction — correct, since an anonymous session has no account to log back into and review a transcript from (mirrors this service's own posture that guest state is never durably attributable to an identity).

`Support Agent`/`Admin` have **no access to this service at all** (ADR-0028/ADR-0029's role-exclusivity) — unlike `kart-payment-service`'s Support-Agent-facing refund tooling or `kart-ai-assistant-service`'s Admin/Support-Agent-facing audit trail (which, per that service's own `database-design.md`, deliberately carries **no** RLS policy because a cross-user compliance review is exactly what that trail exists to support), this service's audit trail has no legitimate cross-customer reader at all — so a strict, unqualified per-principal ownership policy is the correct shape here, not the carve-out the sibling and `kart-payment-service` each independently reach for their own, materially different caller populations.

```sql
ALTER TABLE shopping_assistant_audit_records ENABLE ROW LEVEL SECURITY;
-- The application's runtime connection role must be distinct from whatever role owns/migrates
-- this table — PostgreSQL table owners bypass RLS by default; ENABLE (not FORCE) ROW LEVEL
-- SECURITY relies on the runtime role never being the table owner, the same assumption
-- kart-requirements.md §24.1.4's own worked mechanism makes implicitly. If a future deployment
-- ever collapses the migration and runtime roles into one, ALTER TABLE ... FORCE ROW LEVEL
-- SECURITY must be added at that time.

CREATE POLICY shopping_assistant_audit_records_write_own_principal
    ON shopping_assistant_audit_records
    FOR INSERT
    WITH CHECK (created_by = current_setting('app.current_principal', true));

CREATE POLICY shopping_assistant_audit_records_read_own_principal
    ON shopping_assistant_audit_records
    FOR SELECT
    USING (
        principal_id IS NOT NULL
        AND principal_id::text = current_setting('app.current_principal', true)
    );

ALTER TABLE shopping_assistant_idempotency_keys ENABLE ROW LEVEL SECURITY;

CREATE POLICY shopping_assistant_idempotency_keys_write_own_principal
    ON shopping_assistant_idempotency_keys
    FOR INSERT
    WITH CHECK (created_by = current_setting('app.current_principal', true));

CREATE POLICY shopping_assistant_idempotency_keys_read_own_principal
    ON shopping_assistant_idempotency_keys
    FOR SELECT
    USING (created_by = current_setting('app.current_principal', true));
```

`shopping_assistant_idempotency_keys`'s own read policy is keyed on `created_by` rather than `principal_id` — this table has no customer-facing exposure surface even in principle (it is a purely internal dedup ledger, never referenced by any endpoint in `api-contract.yaml`), so there is no "own audit trail" read-back concept to key on the way there is for the audit table; `created_by` is used here purely so that this service's own reserve-or-reuse lookup (`ddd-model.md`'s Cross-Aggregate Interaction step 3) cannot, even under an application bug, be satisfied by a row reserved under a *different* principal's transaction context — a correctness guard against cross-customer idempotency-scope collision, not a privacy-exposure guard, but implemented via the identical native-RLS mechanism for consistency with the audit table above.

## Cache / Ephemeral-State Model (Redis): `ShoppingAssistantSession`

**This is not a new database this service introduces.** Per architecture.md's Deployment/Scaling Posture and design-decisions.md's Confirmation-State-Machine decision, this is the platform's **existing, shared Redis deployment** (the same one `kart-cart-service`/`kart-identity-service`/`kart-ai-assistant-service` already use for their own comparable ephemeral state) — this service's only footprint on it is its own service-namespaced key prefix (`shopping-assistant:`), never a dedicated cluster, shard, or standalone Redis instance of its own.

```text
Key:   shopping-assistant:session:{sessionId}
Type:  String, holding one JSON-serialized ShoppingAssistantSession value (not a Hash) — mirrors
       kart-ai-assistant-service/database-design.md's own "whole-value String, not per-field Hash"
       rationale: ddd-model.md's Modeling Decision resolves pendingConfirmation to live *inside*
       this same aggregate's consistency boundary specifically so that "clear pendingConfirmation
       atomically with any new intent resolution" is a single GET/SET replace, never a per-field
       HSET that could partially fail mid-turn.
Value: {
         "sessionId":            "<uuid>",
         "principalId":          "<uuid> | null",       -- null for a guest session (ADR-0029)
         "lastResolvedIntent": {
           "intentName":          "<string> | null",
           "resolvedSlots":       { "<slotName>": "<scalar> | ExternalResourceReference" }
         } | null,
         "pendingConfirmation": {
           "intentName":          "<string>",
           "resolvedSlotsSnapshot": { ... },
           "createdAt":           "<ISO-8601 timestamp>",
           "expiresAt":           "<ISO-8601 timestamp>"  -- see TTL strategy below
         } | null,
         "lastActivityAt":       "<ISO-8601 timestamp>"    -- the sliding-idle-TTL anchor
       }

Key:   shopping-assistant:session-lock:{sessionId}
Type:  String, `SET ... NX PX <lockTimeoutMs>` — mirrors kart-ai-assistant-service's own
       `ai-assistant:session-lock:{conversationId}` pattern, generalized here to be load-bearing
       for the idempotency invariant itself, not merely a UX/consistency nicety (ddd-model.md:
       "at most one turn is processed at a time per ShoppingAssistantSessionId... it only
       guarantees 'one key per attempted action' if exactly one turn is ever allowed to attempt a
       given mutating action at a time").
Value: an opaque lock token (compare-and-delete on release, avoiding one turn releasing a lock a
       later turn already re-acquired after the first's own PX expiry).
```

**TTL strategy — mechanism fixed, no number invented (UX-2 remains open).** Two independently-expiring concerns exist inside one physically atomic value, resolved as follows:

- **The outer key's own native Redis `EXPIRE`** governs the whole session's idle lifetime — refreshed (sliding) on every turn that touches this key, exactly as `lastActivityAt` records. This is what makes an orphaned follow-up resolve to `session_expired` (api-contract.yaml) once the key is gone. The concrete duration is UX-2 (requirement-spec.md §10) — genuinely open, not set here or anywhere downstream of it.
- **`pendingConfirmation.expiresAt` is an embedded field, not a second native Redis TTL.** design-decisions.md requires `pendingConfirmation` to carry "its own short TTL, independent of and shorter than the broader conversation-context TTL" — but a single Redis key can only carry one native `EXPIRE`. Splitting `pendingConfirmation` into a second, independently-TTL'd Redis key was considered and rejected for the identical reason `ddd-model.md`'s own Modeling Decision already gives: two independently-written keys reintroduce exactly the "crash between the two writes leaves a stale confirmation alive alongside a freshly-resolved, different intent" failure mode FR-007 forbids. Instead, `pendingConfirmation.expiresAt` is a plain timestamp field inside the same JSON value, checked at application read-time ("if `now() > pendingConfirmation.expiresAt`, treat `pendingConfirmation` as absent even though the outer key hasn't expired") — this preserves the one-key/one-atomic-overwrite guarantee `ddd-model.md` requires while still giving the confirmation its own, shorter functional expiry window. The concrete duration of this shorter window is likewise not invented here (same UX-2/open-figure posture).
- **Explicit clear-on-new-resolution is still unconditional and TTL-independent** — `ddd-model.md`'s own invariant ("`pendingConfirmation` is explicitly cleared, not left to passive TTL expiry, the instant FR-002 (re-)resolves any new intent") is enforced by the application overwriting the whole JSON value (a fresh `SET`) on every turn, not by either TTL mechanism above; both TTLs are a passive backstop for an *idle* session/confirmation, never the primary mechanism for a topic-switch discard.

**Not designed here, for completeness:** `purgatory`'s own circuit-breaker state (design-decisions.md's Per-Downstream-Service Circuit Breaker decision) is a separate, non-domain use of this same shared Redis deployment — nine named breaker instances, one per downstream peer — already fully specified by design-decisions.md/architecture.md and out of scope for this document's aggregate design, since it is not part of `ShoppingAssistantSession` or any of this service's own three owned structures.

## Sensitive / PII Column Classification (kart-requirements.md §24.1.5)

This capability's PII surface is **substantially larger** than the read-only sibling's aggregate-metadata-only logs (requirement-spec.md §6) — order contents, full address detail, and masked payment-method references are a structural consequence of what this assistant does, and will routinely appear inside `shopping_assistant_audit_records`' JSON/text columns.

| Column | Full Value Visible To | Masked/Omitted For | Masking Rule |
|---|---|---|---|
| `shopping_assistant_audit_records.response_summary` | The owning `principal_id` only, if/when a future customer-facing transcript-review endpoint is ever built (gated by the RLS `SELECT` policy above as the primary control, and by that future endpoint's own coarse-role/scope check as a second layer) | Any other caller, including guests, `Admin`, and `Support Agent` — ADR-0028/0029's role-exclusivity means no internal-staff caller population exists for this table at all, unlike `kart-payment-service`'s Support-Agent-facing tables | Order/address/product content that appears here is bounded, natural-language text describing what was said this turn — **never a raw `gateway_token` or any unmasked PCI-scoped value** (ADR-0028's ownership table item 3, requirement-spec.md §6); only a masked display reference (e.g. "Visa ending 4242") an owning service has *already* returned may ever appear. This is an application-level construction-time invariant (the summarization/confirmation-text generation step never has access to an unmasked token to begin with, per ddd-model.md's own restraint on modeling a masked-payment-reference value object) — Postgres itself cannot structurally validate the *contents* of a free-text column, so this is enforced upstream, not by a DB `CHECK` |
| `shopping_assistant_audit_records.proposed_action` / `.execution_outcome` (both JSONB) | Same as `response_summary` above | Same as `response_summary` above | `targetResource` inside either payload is always the opaque `{resourceType, resourceId}` shape (ddd-model.md) — never an embedded order/cart/coupon/address/product sub-object; no PII beyond an opaque foreign identifier can appear in this specific sub-field by construction of the value object itself |
| `shopping_assistant_audit_records.resolved_intent` / `.clarification_detail` | Same as `response_summary` above | Same as `response_summary` above | May echo a slot value the customer themselves typed (an order id, a SKU, a coupon code) — never a richer fetched record; per `ddd-model.md`'s `ExternalResourceReference` invariant, this service never durably stores more than an opaque id + type tag for any foreign resource anywhere in its own schema |
| `shopping_assistant_audit_records.principal_id` / `created_by` / `updated_by` | Same as `response_summary` above (`principal_id`); internal-only, never API-surfaced (`created_by`/`updated_by`, kart-requirements.md §24.3) | All other callers | Opaque identifiers (a `Customer`'s `UserId`, or a well-known guest pseudo-principal string) — not directly-identifying PII on their own, the same treatment `kart-ai-assistant-service/database-design.md` gives its own `user_id`/`role` columns |
| `shopping_assistant_idempotency_keys.*` | No caller — this table is never exposed by any endpoint, read only by this service's own reserve-or-reuse code path | Everyone else | Holds only a session reference, an intent name, a slots hash, and a minted dedup token — no PII/PCI-scoped value of any kind (mirrors `kart-payment-service/database-design.md`'s own `idempotency_keys` classification: "no plaintext sensitive column to mask in the first place") |

**Why no column here is ever a raw `gateway_token` or any other unmasked PCI value:** this service never calls `kart-payment-service` directly (ADR-0028's permanent exclusion) and never receives a raw gateway token from any of its nine allowlisted peers — the only payment-adjacent fact it can ever observe is whatever an owning service (e.g. `kart-order-service`'s own cancel/refund response) has already itself masked before returning it. There is structurally no unmasked value for this service's own schema to leak, mirroring `kart-payment-service/database-design.md`'s own "no unmasked PCI data exists to protect in the first place" conclusion, reached here for a different reason (this service simply never touches the data at all, rather than tokenizing it at the source).

## GDPR / Right-to-Erasure and Retention — Open Question, Not Decided Here

**No source document states a retention period for `shopping_assistant_audit_records` or `shopping_assistant_idempotency_keys`.** This is requirement-spec.md §10's own Data-3, carried forward verbatim: "retention policy for these conversation transcripts is not decided here... also touches whether `kart-user-service`'s existing GDPR Right-to-Delete/export fan-out needs to be extended to this service's own conversation store." Neither `design-decisions.md` nor `ddd-model.md` invents a number or a working default the way UX-2's session-TTL carries one forward as "genuinely open, not this stage's to set" — this document does the same: **no retention/purge job, no `expires_at`/cold-tiering column, and no `UserDataErased`-consuming erasure handler is included in this design.** The safe default in the absence of a stated policy is to retain rows (an audit trail that under-retains defeats requirement-spec.md §6's own auditability purpose), the identical posture `kart-ai-assistant-service/database-design.md` already takes for its own analogous open retention question.

This is flagged here as a standing open item — **`ShoppingAssistantAuditRecord`/`ShoppingAssistantIdempotencyKey` Retention & Erasure Policy** — requiring the same category of privacy/legal/product-owner decision Data-3 already requires before this capability is production-ready; it does not block this schema from being implemented in the meantime, the same way Data-3 itself does not block DDD/Architecture/API Design work.

## Indexing Rationale

| Index | Query it supports | Why needed |
|---|---|---|
| `shopping_assistant_audit_records` PK on `turn_id` | Point lookup by `turnId` (`ddd-model.md`'s own stated query pattern) | Direct point lookup for reconstructing/auditing one exact turn |
| `idx_shopping_assistant_audit_records_session (shopping_assistant_session_id, created_at)` | `ddd-model.md`: "reconstruct this session's full transcript... ordered by" time | Without it, listing one session's turns degrades to a full-table scan as the append-only table grows; the compound key gives both the filter and the required ordering in one index |
| `idx_shopping_assistant_audit_records_principal (principal_id, created_at) WHERE principal_id IS NOT NULL` | `ddd-model.md`: "queried by... `principalId`/date range for compliance" | Supports "what did this Customer's assistant do, over what window"; partial (excludes guest rows, which this query pattern can never target) so the index isn't padded with rows it never serves |
| `idx_shopping_assistant_audit_records_created_at (created_at)` | `ddd-model.md`'s date-range dimension, independent of `principalId` | The `(principal_id, created_at)` index above cannot serve a principal-agnostic range scan (its leading, filtered column is `principal_id`) — self-critiqued gap-closure mirroring `kart-ai-assistant-service/database-design.md`'s own equivalent fix |
| `shopping_assistant_idempotency_keys` PK on `(shopping_assistant_session_id, intent_name, resolved_slots_hash)` | "Does a reservation already exist for this attempt's scope?" — the reserve-or-reuse lookup design-decisions.md/`ddd-model.md` both name as this table's one job | Must be a unique index, not incidental — this *is* the mechanism that guarantees "one attempt per resolved mutating action" |
| No JSONB/GIN index on any `shopping_assistant_audit_records` JSONB column | — | No stated query pattern filters on intent/action/outcome/error content as a standalone lookup (only `turnId`/`sessionId`/`principalId`/date-range are named, all covered above) — adding one would be a speculative index this stage's own rule forbids |
| Redis `shopping-assistant:session:{sessionId}` / `shopping-assistant:session-lock:{sessionId}` (key-based, no secondary index — Redis has none) | Every turn's session read/upsert and per-session serialization lock (design-decisions.md, `ddd-model.md`) | Native O(1) key lookup is the entire access pattern; no query-by-field ever occurs against session state |

## Partitioning / Sharding

**Not needed at current scale, for either Postgres table.** No capacity-plan figure specific to this service states a write-throughput number distinct from the platform-wide figures every other synchronous, secondary-tier customer-facing service already inherits (requirement-spec.md §4's own "99.9%, secondary tier" framing); per-turn write volume on both tables is additionally bounded by this service's own up-to-nine-peer synchronous fan-out and LLM round-trip latency on the same request's critical path (architecture.md's Distributed-Monolith Risk section), the same rate-limiting argument `kart-ai-assistant-service/database-design.md` makes for its own comparably-shaped audit table. A single, indexed PostgreSQL table is sufficient for both `shopping_assistant_audit_records` and `shopping_assistant_idempotency_keys` today.

`shopping_assistant_idempotency_keys` is the one table here with a plausible future partitioning need, once the currently-open replay-window TTL figure (`ddd-model.md`, design-decisions.md) is set — mirroring `kart-payment-service/database-design.md`'s own `idempotency_keys` reasoning exactly: range-partitioning by `created_at` (e.g. daily) would let TTL cleanup become an efficient partition-drop rather than a scanning `DELETE` competing with request-path writes. This is flagged as the natural next step for that future point, not decided or built now, since no TTL number currently exists to partition against.

**Redis requires no sharding decision at this scale either.** One key pair (`session` + `session-lock`) per active conversation, evicted by TTL on idle — the platform's existing shared Redis deployment (already used by `kart-cart-service`/`kart-identity-service`/`kart-ai-assistant-service` for a comparable ephemeral-state profile) absorbs this without a dedicated cluster or shard-key decision.

## Explicit Ownership-Ceiling Invariant (ADR-0028)

Restated once more, directly, so this document is unambiguous on its own terms: **this schema contains no `Order`, `Cart`, `Product`, `Coupon`, `User`, `Address`, `Wishlist`, `SearchDocument`, `RecommendationSet`, `TrackingRecord`, or `PaymentIntent`-shaped table or column anywhere** — only `shopping_assistant_audit_records`, `shopping_assistant_idempotency_keys` (both Postgres, this document), and `ShoppingAssistantSession` (Redis, this document). Every reference to another bounded context's data, anywhere in either Postgres table or the Redis value, is an opaque `{resourceType, resourceId}` `ExternalResourceReference` pair (ddd-model.md) — never a cached price, stock level, order status, or any other business fact that resource's owning service itself computes. No local cache table for downstream eligibility/price/stock data exists, matching ADR-0028's Consequences section verbatim: "the Database Design Agent designs storage only for conversation-session state and the audit log [and, per `ddd-model.md`'s own resolution, the idempotency ledger] — no domain-data read model of its own, and no local cache of any downstream service's eligibility/price/stock data."

## Sign-off

- [x] Read/cache-model (Redis `ShoppingAssistantSession`, including `pendingConfirmation`) — approved directly per this stage's own rule; no new storage-technology decision made here beyond documenting an already-closed choice's concrete key/value shape and TTL mechanism.
- [ ] Write-model schema (`shopping_assistant_audit_records`, `shopping_assistant_idempotency_keys`) — reviewed by: _pending human/DBA review before migration is run._
- [ ] `ShoppingAssistantAuditRecord`/`ShoppingAssistantIdempotencyKey` retention & erasure policy — open question surfaced by this document (mirrors Data-3, requirement-spec.md §10); requires a privacy/legal/product-owner decision before this schema's storage-cost/compliance posture is production-ready. Does not block this schema from being implemented in the meantime.

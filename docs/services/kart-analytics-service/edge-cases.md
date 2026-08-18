---
doc_type: edge-cases
service: kart-analytics-service
status: approved
generated_by: edge-case-analyzer-agent
source: docs/services/kart-analytics-service/requirement-spec.md
---

# Edge Cases: kart-analytics-service

## Edge Case: Schema Evolution Breaking Downstream Consumers

- **What happens:** An upstream publisher changes an event's shape (renamed/removed/retyped field, new required field) with no compatibility contract, and Analytics either crashes on ingest, silently drops the event, or writes corrupted rows to the warehouse.
- **Why it happens:** Requirement-spec Domain Invariant #1 asserts schema versioning discipline is forced on Analytics (BRD §2.2), but no compatibility scheme or registry is defined anywhere in the BRD — nothing gates a publisher from shipping a breaking change before it reaches Analytics.
- **Solutions available (3):** Schema registry with enforced backward/forward compatibility checks at publish time (e.g. Avro/Protobuf + compatibility mode) · Consumer-side tolerant reader (ignore unknown fields, default missing ones) with dead-lettering on unparseable payloads · Contract tests between publisher and Analytics gating CI/CD on schema diff
- **Decision (3-5 bullets max):**
  - Chosen: Confluent-compatible schema registry (Avro payloads), enforced `BACKWARD` compatibility mode as the primary gate, plus a consumer-side tolerant reader as defense in depth.
  - Why: Requirement-spec calls this a *forced* discipline, not an optional nicety — only a registry stops a bad change before it ships, versus catching it only after Analytics already broke. The concrete format (Avro + registry-assigned schema ID as the version pointer, `MAJOR.MINOR` subject metadata for human-readable additive-vs-breaking distinction) is now decided in requirement-spec.md §6 (D2), closing the residual gap this edge case previously escalated.
  - Trade-off accepted: Every publisher now owns a schema contract and compatibility gate — added process coupling across otherwise-independent service teams. A `MAJOR` bump additionally requires a dual-publish transition window per event type, mirroring the platform's existing RabbitMQ→Kafka dual-publish precedent (BRD §15).

## Edge Case: Event Volume / Backpressure at Full Platform Fan-In

- **What happens:** At platform peak (BRD NFR: 1M RPS flash-sale burst), aggregate event volume across every publisher overwhelms Analytics' ingestion consumers, producing consumer lag, unbounded topic backlog, or warehouse write saturation.
- **Why it happens:** Analytics' load scales with total platform event volume, not one bounded stream (requirement-spec §2/§5 — fan-in from most or all published events); the BRD itself names Analytics' fan-out across 10+ consumer groups per event as the reason RabbitMQ's throughput ceiling was breached in the first place.
- **Solutions available (2):** Kafka partitioned parallel consumption with an autoscaled consumer group (already the BRD's stated direction for Analytics) · Load-shedding/sampling of low-priority events (e.g. audit-only `NotificationSent`) under sustained backlog
- **Decision (3-5 bullets max):**
  - Chosen: Kafka partitioning + horizontally autoscaled consumer group (K8s HPA on consumer-group lag), with micro-batched warehouse writes to cut write amplification.
  - Why: The requirement-spec's Throughput NFR already commits Analytics to Kafka specifically for high-throughput partitioned consumption — this follows the platform's existing decision rather than inventing a new one.
  - Trade-off accepted: No load-shedding means Analytics must be provisioned (and pay) for full-fidelity ingestion at burst volume instead of degrading gracefully when overwhelmed.

## Edge Case: Replay Correctness (Reprocessing Without Double-Counting)

- **What happens:** Reprocessing historical events (the BRD's own scenario: 30 days of `OrderCreated` after a bug fix) re-ingests events already reflected in existing dashboard/funnel aggregates, inflating metrics such as order counts or revenue.
- **Why it happens:** Replay is a BRD-mandated capability (requirement-spec FR, BRD §14), but requirement-spec Domain Invariant #3 notes nothing in a plain "sum this stream" aggregation is inherently idempotent — replaying the same events into an incrementing counter double-counts them.
- **Solutions available (2):** Idempotent upserts keyed by event ID at raw-event storage, with aggregates recomputed from raw storage rather than incrementally mutated · Replay-aware shadow-table mode: replay writes to a separate table, diffed and swapped in rather than replayed into the live aggregate path
- **Decision (3-5 bullets max):**
  - Chosen: Idempotent upserts on raw event storage (dedup by event ID), with all aggregates recomputed from raw storage rather than maintained as incrementing counters.
  - Why: This makes live ingestion and replay share one code path safely — a separate incrementing-counter path would need its own replay-safe variant that could drift from live behavior.
  - Trade-off accepted: Recomputing aggregates from raw storage costs more compute per refresh than maintaining running counters — correctness is bought with recompute cost.

## Edge Case: Out-of-Order Event Arrival Skewing Funnel/Time-Series Accuracy

- **What happens:** Events from independent publishers (or the same logical event observed twice during the RabbitMQ→Kafka dual-publish strangler window, BRD §15) arrive out of causal order — e.g. `PaymentCompleted` observed before `OrderCreated` — producing funnels with impossible sequences or time buckets that undercount/overcount a window.
- **Why it happens:** The BRD's Event Catalog gives each publisher its own independent retry/backoff policy (1x-5x depending on event), and no BRD section states a global ordering guarantee across the union of all events Analytics consumes; the dual-publish migration window (BRD §15) adds a second, transient source of skew.
- **Solutions available (3):** Event-time windowing with watermarks/allowed lateness in the stream processor · Delayed funnel computation, finalizing a stage's metrics only after a fixed grace period closes · Real-time dashboards marked "provisional," reconciled by a nightly batch recompute from the full event log
- **Decision (3-5 bullets max):**
  - Chosen: Event-time windowing with watermarks/allowed lateness, combined with nightly batch reconciliation from raw storage.
  - Why: Reuses the raw-storage-first design already chosen for replay correctness — one source of truth handles both "replay after a bug" and "events arriving late," instead of a bespoke mechanism per failure mode.
  - Trade-off accepted: Real-time dashboards stay provisional/approximate until the nightly reconciliation runs — exact funnel numbers are not available instantly.

## Edge Case: Ranking Ties at the Limit Cutoff (product-performance)

**Added this pass — driven by requirement-spec.md §6 item 7 / §2's new FR (11th dashboard), sourced from `docs/requirements/genai-business-assistant-spec.md` §10.3's `GET /internal/v1/dashboards/product-performance` endpoint. None of the original ten dashboards sorts or truncates a result set, so no existing entry addresses this.**

- **What happens:** Two or more products share the exact same metric value (`revenue`/`units_sold`/`order_count`) at the boundary between the last included rank and the first excluded rank — e.g. `limit=5`, but the products that would rank 5th and 6th are tied on revenue. A naive `$sort` + `$limit` MongoDB aggregation gives no ordering guarantee among equal-valued documents, so which of the tied products is returned can vary nondeterministically across otherwise-identical repeat calls.
- **Why it happens:** requirement-spec §6 item 7 and genai-business-assistant-spec.md §10.3 define the cutoff purely on the primary requested metric, with no secondary sort key stated anywhere in either spec; MongoDB (the read model's own store, per §10.2/§10.3) does not guarantee stable ordering for equal sort-key values unless a tiebreaker field is included in the sort.
- **Solutions available (3):** Deterministic secondary sort key appended by convention (e.g. `sku` ascending) so ties resolve the same way on every repeat call · Return every tied product even if that means more than `limit` rows (treat `limit` as "at least N," not exact) · Leave tie order undefined/arbitrary (fastest to implement, but silently non-reproducible)
- **Decision (3-5 bullets max):**
  - Chosen: Deterministic secondary sort key — `sku` ascending — appended after the primary metric in the `$sort` stage; the endpoint always returns exactly `limit` rows.
  - Why: `kart-ai-assistant-service` treats this endpoint as ground truth for a conversational answer (genai-business-assistant-spec.md FR-003: "the returned data matches exactly what a direct call to the same Analytics endpoint... would return") — a business user re-asking "top 5 products" moments later must not see a different 5th product purely from tie-order nondeterminism, which would read as a data bug even though no underlying total changed.
  - Trade-off accepted: the distinction between the Nth and (N+1)th product in a genuine tie is arbitrary (by SKU), not a substantive business signal — accepted in favor of keeping the "exactly `limit` rows" contract stable, which both the assistant's table renderer (§13.2) and any other caller depend on.

## Edge Case: Rank Order/Membership Changes on Reconciliation (product-performance)

**Added this pass — driven by requirement-spec.md §6 item 7 / §2's new FR, sourced from `docs/requirements/genai-business-assistant-spec.md` §10.3. This is a genuinely new wrinkle, not a duplicate of "Out-of-Order Event Arrival Skewing Funnel/Time-Series Accuracy" above — see "Why it happens" for the distinction; that entry's `isProvisional`/nightly-reconciliation decision is reused as-is, not re-litigated.**

- **What happens:** A query against a window that includes today's still-provisional bucket can return a top/bottom-N list whose *order, or even membership,* differs from what the same query returns once nightly reconciliation finalizes that bucket — e.g. a product provisionally ranked 5th is actually 3rd once late-arriving events settle, which can push a different product out of the visible top-N entirely rather than merely correcting one product's displayed value in place.
- **Why it happens:** The existing "Out-of-Order Event Arrival" edge case already resolves provisional *values* for every dashboard (nightly reconciliation + `isProvisional`/`reconciledThrough`), but none of the original ten dashboards ever sorts and truncates a result set — a value drifting up or down provisionally is still visible in place, so it reads as "the number moved," not "the list changed." A ranked, `limit`-bounded list is qualitatively different: reconciliation can move a product across the cutoff boundary in either direction, changing *which* products are shown at all. Nothing in the existing entry's decision anticipates this because no prior dashboard had a cutoff to cross.
- **Solutions available (3):** Rely on the existing whole-response `isProvisional`/`reconciledThrough` `DashboardEnvelope` fields as sufficient disclosure, with no ranking-specific addition · Suppress ranked queries entirely for any window containing an unreconciled bucket (reject with 409/"not yet available" until reconciliation completes) · Serve the provisional ranking as requested but pad the response with a wider "shadow" window (e.g. top-(N+buffer)) so near-cutoff volatility is visible without an extra round-trip
- **Decision (3-5 bullets max):**
  - Chosen: No new mechanism — the existing `isProvisional`/`reconciledThrough` envelope fields, applied at the whole-response level exactly as the other ten dashboards already do, are treated as sufficient disclosure for this endpoint too.
  - Why: `kart-ai-assistant-service`'s own spec (genai-business-assistant-spec.md FR-012, G4) already defines a whole-response `isProvisional` flag as sufficient disclosure for any metric drift, including a ranked list's reordering — its own worked example (§3.3) states "today's figures are still provisional" without promising positional stability. Introducing a second, ranking-specific staleness signal would be a bespoke mechanism for a caller that has already said the coarse flag is enough, and this endpoint's only consumer today is that caller.
  - Trade-off accepted: a business user asking "top 5 products" during the provisional window can see a product silently disappear from tomorrow's "same question" answer, with no explicit warning that *rank*, not just magnitude, was unstable — accepted because the alternative (blocking ranked queries on any unreconciled bucket) would make the endpoint unusable for its single most-cited use case: a rolling "last 7 days" window that always includes today's still-provisional bucket (genai-business-assistant-spec.md §3.3).

## Note: `limit` Parameter Bounds — No New Edge Case

**Checked this pass, per the task that commissioned this addition.** Invalid `limit` values (`0`, negative, or unreasonably large, e.g. `10000`) were evaluated for a new entry and rejected: no existing dashboard in this service has ever exposed a `limit`/pagination-style parameter (`from`/`to`/`granularity` are the only precedents, per api-contract.yaml), so there is no existing generic "parameter validation" edge case to inherit from — but this is also not a consequence of any domain rule, NFR, or invariant this service's requirement-spec states (per this agent's own scope rule: only include edge cases that are real consequences of the service's domain rules, not a generic input-validation checklist). Rejecting out-of-range `limit` values with a `400` and clamping/documenting a sane maximum (e.g. product catalog size) is ordinary OpenAPI schema work (`minimum`/`maximum` constraints) for the API Design Agent when this endpoint's contract is finalized, not a domain edge case requiring a decision record here.

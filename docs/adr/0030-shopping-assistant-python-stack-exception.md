---
doc_type: adr
status: accepted
---

# ADR-0030: `kart-shopping-assistant-service` Is Built in FastAPI (Python) — a Single-Service Stack Exception, With Named Python-Native Equivalents Required for the Platform's .NET-Coupled Shared Conventions

## Status

Accepted

## Context

Every one of the platform's 19 existing deployable repos is built on **.NET 9 / ASP.NET Core** (`kart-requirements.md` line 5's stack banner, applied uniformly, including `kart-ai-assistant-service`). `docs/services/kart-shopping-assistant-service/requirement-spec.md` §1.1 states, per this capability's own commissioning directive, that **`kart-shopping-assistant-service` is to be built in FastAPI (Python) instead** — the first deliberate single-service language exception anywhere on the platform. The requirement-spec explicitly declines to resolve this itself beyond flagging it (its own escalation rule: a decision this consequential and cross-cutting is not the requirement-agent stage's to make unilaterally), and names three platform-wide shared-library conventions with no Python equivalent today:

1. **`Kart.Shared.Auditing`'s automatic audit-field injection** — `created_at`/`updated_at`/`created_by`/`updated_by` on every mutable table, populated via an EF Core `SaveChangesInterceptor` reading the resolved principal (`kart-requirements.md` §24.3), never client-suppliable.
2. **The JSON-manifest-driven RabbitMQ topology** — each service declares its own exchange/queues/DLQ/retry ladder idempotently at startup by reading a JSON manifest (`kart-requirements.md` §8/§9), implemented today via a .NET startup hook.
3. **EF Core interceptor patterns generally** — row-level-security session-variable injection (`ICurrentPrincipalAccessor`, `kart-requirements.md` §24.1.4), optimistic-concurrency `version`-column handling, and similar cross-cutting guarantees every .NET service gets "for free."

This ADR exists to (a) formally accept the stack exception and name what the platform takes on by allowing it, and (b) fix the **policy constraint** every Python-native equivalent must satisfy — without itself picking the specific library/ORM/pattern, which is properly the Design-Decision Agent's job (the same division of labor ADR-0024 used: that ADR closed `kart-ai-assistant-service`'s data-ownership *boundary*, while its own `design-decisions.md` picked the concrete mechanisms satisfying that boundary).

## Decision

**Accepted.** `kart-shopping-assistant-service` is built in FastAPI (Python), per the explicit commissioning directive for this capability. This is a conscious, named exception to the platform's otherwise-uniform .NET stack, not an inadvertent drift — recorded here exactly as the requirement-spec's own escalation rule required, mirroring how ADR-0010/0024/0025 each formalized a decision already directed by that capability's own founding instruction rather than inventing one fresh.

**Rationale for the exception:** an LLM-orchestration-heavy, tool-calling, structured-output-constrained service (requirement-spec.md §8; source spec §12) sits closer to Python's GenAI/LLM tooling ecosystem (structured-output/function-calling libraries, provider SDKs, the broader agent-orchestration tooling landscape) than to .NET's — the same category of reasoning that already justifies OpenSearch-instead-of-Mongo for `kart-search-service` or MongoDB-instead-of-Postgres for read models elsewhere on this platform: pick the tool that fits the job, don't force a uniform default where the job's own shape argues against it.

**Operational cost accepted, named explicitly rather than left implicit:** running one Python service inside an otherwise all-.NET fleet means the platform now needs — at minimum — a Python base container image and multi-stage Dockerfile pattern (mirroring `kart-requirements.md` §21's existing .NET multi-stage convention, restated for Python), a Python-specific CI/CD template (lint/type-check/test/build stages equivalent to the .NET template), Python-aware dependency-scanning tooling alongside the existing .NET one, and an on-call runbook expectation that at least some responders can read/operate a Python service. None of these are designed by this ADR — each is a concrete, non-blocking follow-up ticket (see the eventual `tickets.md`), not a reason to reverse this decision.

**Policy constraint on every Python-native equivalent — binding on the Design-Decision Agent stage that follows this ADR:** each of the three named conventions above must be replaced by a Python-native mechanism that provides the **same guarantee**, not a weaker "best effort" version:

| .NET convention | Guarantee it provides | Python-native equivalent must... |
|---|---|---|
| `Kart.Shared.Auditing` (`SaveChangesInterceptor`) | `created_at`/`updated_at`/`created_by`/`updated_by` are populated automatically from the resolved principal and are **never client-suppliable**, on every mutable row | Be enforced at the ORM/data-access layer (e.g. an ORM-level event hook), not by convention/discipline in application code — a developer must not be able to accidentally bypass it by writing a raw insert |
| JSON-manifest-driven RabbitMQ topology | Topology changes are a reviewable JSON diff, declared idempotently at startup, identical schema across every service regardless of language | Read the **same JSON manifest schema** already established (`kart-requirements.md` §9) via a Python RabbitMQ client — the manifest format is already language-agnostic; only the declaring code is new. Not required at all unless a future pass (requirement-spec.md §9 item 4 / Event Design Agent) determines this service needs any publish/consume relationship — resolved as zero for v1, so this equivalent is a documented-but-unbuilt pattern until that changes |
| EF Core interceptor patterns (RLS session-variable injection, optimistic concurrency) | Row-level security is enforced at the database layer even against buggy application code (`kart-requirements.md` §24.1.4); concurrent writes don't silently clobber each other | Achieve equivalent database-layer enforcement (e.g. PostgreSQL native RLS policies, which are enforced by Postgres itself regardless of which language's driver connects — the RLS mechanism `kart-requirements.md` §24.1.4 already describes is database-native, not EF-Core-native, so this is more a "confirm and configure" item than a "build a substitute" item) |

**A new shared internal package convention is required, not ad hoc per-file solutions:** the concrete implementations of the above must live in one internal, reusable Python package (recommended name: `kart_shared`, mirroring the `Kart.Shared.*` .NET naming convention) — so that if a second Python service is ever added to the platform, it inherits these guarantees rather than re-solving them. This ADR does not design that package's internals (module structure, dependency choices) — that is the Design-Decision Agent's job, working from this ADR's constraint table.

**This ADR does not pick a specific ORM, ASGI server, dependency-injection pattern, or structured-output/tool-calling library** — those are concrete implementation choices properly made by the Design-Decision Agent stage that follows, working within the guarantees this ADR fixes.

## Consequences

- Requirement-spec.md §10 Architecture-7 is closed at the policy level: the stack exception is accepted, its rationale and accepted operational cost are named, and the binding constraint every Python-native shared-convention equivalent must satisfy is fixed. The *concrete* mechanisms (which ORM, which RLS configuration, which structured-output library) are explicitly left to `design-decisions.md`, the next pipeline stage.
- Platform-wide onboarding/runbook documentation should eventually note that `kart-shopping-assistant-service` is the platform's first non-.NET service — a documentation follow-up, not something this ADR itself rewrites.
- Any future second Python service should reuse the `kart_shared` package this ADR requires, rather than re-deriving its own audit/RLS/manifest equivalents independently.
- If the Design-Decision Agent stage finds that any of the three named guarantees genuinely cannot be met in Python at parity with the .NET original, that finding must come back as a new blocking issue against this ADR, not be silently downgraded to "best effort" in `design-decisions.md`.

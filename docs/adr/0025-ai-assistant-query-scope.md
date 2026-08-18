---
doc_type: adr
status: accepted
---

# ADR-0025: New `ai-assistant.query` Scope Is Identity-Issued via Role→Scope Mapping, Not a Persisted Grant Table

## Status

Accepted

## Context

`docs/requirements/genai-business-assistant-spec.md` §17 introduces a second-tier authorization check this capability needs but that no existing document names: "`kart-ai-assistant-service`'s own check: a new scope, e.g. `ai-assistant.query`" — flagged explicitly as **§26-Security-1, OPEN QUESTION**: "The exact mechanism and name for the new `ai-assistant.query`-equivalent scope... is a new decision this spec introduces and flags, not something any existing document already answers."

This has to be reconciled against the platform's existing two-tier RBAC model (`kart-requirements.md` §24.1):

- **Tier 1 (coarse role)** — Identity Service is the single issuer, embedded in the JWT at token-mint time (`roles: [...]`, `scopes: [...]`), per §24.1.1's "Issuance" bullet.
- **Tier 2 (fine-grained CanRead/CanWrite/CanDelete)** — owned per-resource-owning-service, and §24.1.2 explicitly leaves the *mechanism* up to each service: "a persisted grant table, an inline rule, or a plain ownership check are all valid implementations of this tier... Each service's own `requirement-spec.md`/`ddd-model.md`/`design-decisions.md` records which mechanism it chose and why."

The closest existing precedent is `kart-analytics-service`'s `analytics.dashboards.read` scope (`requirement-spec.md`: "an inline scope check at the internal-only query layer, not a persisted grant table, since dashboards are platform-wide aggregate data with no individual owner to compare a caller against") — issued the same way, embedded directly in a service-principal's OAuth2 Client-Credentials token via Identity's role/principal→scope mapping. But `analytics.dashboards.read` gates a **service-to-service, client-credentials** caller; `ai-assistant.query` must gate a **human user's own JWT** (an `Admin`/`Support Agent` calling `POST /v1/ai-assistant/query` through the Gateway, per spec §21.1, §10.4) — a materially different caller shape that this ADR must not silently conflate.

There is a second-order question this ADR must also close: the platform has **two coexisting OpenAPI security-scheme conventions** for the same underlying JWT mechanism — `kart-admin-service/api-contract.yaml` uses `bearerAuth: [admin]` (bare coarse-role value, because Admin's fine-grained check happens in-process against `admin_permission_grants`, never expressed as an OpenAPI scope), while `kart-analytics-service/api-contract.yaml` uses `internalClientCredentials: [analytics.dashboards.read]` (dotted `<service>.<resource>.<verb>` scope value, because Analytics' CanRead check *is* the scope). `kart-ai-assistant-service`'s API contract needs one of these — or a justified third pattern — decided now, not left to the API Design Agent to invent ad hoc.

## Decision

**New scope name: `ai-assistant.query`** — adopted verbatim as the spec's own working name (§17), not expanded to a stricter 3-segment `<service>.<resource>.<verb>` form (e.g. `ai-assistant.query.execute`) because there is no separate "resource" here distinct from the action itself: unlike Analytics' ten-plus dashboards (a resource collection you `read`), this service exposes exactly one action (`POST /v1/ai-assistant/query`) gating exactly one capability. A 2-segment `<service>.<action>` scope is the more honest shape for a single-action service and avoids inventing a resource noun that doesn't correspond to anything in this service's own domain model (§10.3's read-only orchestration boundary, formalized in ADR-0024).

**Issuance mechanism: embedded directly in the human user's JWT at token-mint time, via Identity's existing role→scope mapping — no new persisted grant table.** Concretely: Identity Service's role-resolution step (`kart-identity-service/requirement-spec.md`: "resolve the authenticated user... to a role set and embed it as scoped claims (`roles: [...]`, `scopes: [...]`)") adds `ai-assistant.query` to the `scopes` claim for both `Admin` and `Support Agent` roles — the same two roles already gated at the Gateway's coarse check (spec §17 check 1). This is a Tier-1-shaped addition (Identity-issued, role-driven, platform-wide, rarely-changing) layered as a Tier-2 check at `kart-ai-assistant-service`'s own boundary, exactly the way `analytics.dashboards.read` is a Tier-1-issued claim enforced as a Tier-2 inline check at Analytics' boundary — the same reasoning the spec's own §17 OPEN QUESTION recommendation used, now made a closed decision. A persisted grant table is rejected for the same reason `analytics.dashboards.read` rejected one: this capability has no individual-row resource to compare a caller's grant against (every `Admin`/`Support Agent` who can reach the feature at all gets the same scope; there is no per-user variation to persist).

**Two distinct scopes for two distinct callers — do not conflate them:**

| Caller | Token type | Scope | Enforced at |
|---|---|---|---|
| `Admin`/`Support Agent` human user, via `kart-admin-web` → Gateway | Identity-issued user JWT, forwarded unchanged by the Gateway (ADR-0023) | `ai-assistant.query` | `kart-ai-assistant-service`'s own boundary (§17 check 2) |
| `kart-ai-assistant-service` itself, calling Analytics | OAuth2 Client-Credentials service-principal token | `analytics.dashboards.read` (unchanged, pre-existing) | `kart-analytics-service`'s own boundary (§17 check 3) |

This is exactly the three-check flow `kart-requirements.md` §24.1.3 already specifies, now with every check's concrete scope named:
1. Gateway coarse check: JWT carries `Admin` or `Support Agent` role claim.
2. `kart-ai-assistant-service`'s own check: the same (now Gateway-forwarded, unmodified) JWT's `scopes` claim carries `ai-assistant.query`.
3. `kart-analytics-service`'s own check: the downstream service-to-service call carries `analytics.dashboards.read`.

**API contract convention: follow Admin's `bearerAuth` scheme shape, not Analytics' `internalClientCredentials` shape**, because the caller here is a human JWT forwarded through the Gateway (Admin's caller shape), not a service-principal Client-Credentials token (Analytics' caller shape). Concretely:

```yaml
securitySchemes:
  bearerAuth:
    type: http
    scheme: bearer
    bearerFormat: JWT
    description: >
      Identity-issued RS256 JWT carrying an Admin or Support Agent role claim
      plus the `ai-assistant.query` scope (ADR-0025), forwarded unchanged by
      the API Gateway (ADR-0023) — kart-ai-assistant-service re-validates the
      same token rather than trusting a minted internal header.
paths:
  /v1/ai-assistant/query:
    post:
      security:
        - bearerAuth: [ai-assistant.query]
```

## Consequences

- §26-Security-1 is closed: the scope's name (`ai-assistant.query`), issuance mechanism (Identity role→scope mapping, no grant table), and enforcement point are all now decided, not left to the DDD/API Design Agents to invent independently (which risked two agents making inconsistent choices).
- `kart-identity-service`'s own docs (`requirement-spec.md`, and its role→scope mapping config) gain one line item: `Admin` and `Support Agent` roles both resolve to include `ai-assistant.query` in the minted JWT's `scopes` claim. This is a config addition to an already-approved mechanism, not a new mechanism — no `kart-identity-service` ADR or contract change is required.
- The DDD Agent must model authorization against this scope (not a persisted grant/permission aggregate) when it models `kart-ai-assistant-service`'s own boundary — there is no `PermissionGrant`-shaped entity to design here, unlike Admin Service's `admin_permission_grants`.
- The API Design Agent gates `POST /v1/ai-assistant/query` with `security: - bearerAuth: [ai-assistant.query]`, following the `securitySchemes` shape above verbatim.
- `kart-analytics-service`'s own contract and scope (`analytics.dashboards.read`) are unchanged — this ADR adds a new scope for a new caller-shape, it does not touch Analytics' existing security model (consistent with spec §17's own "No change to Analytics' security model" statement).

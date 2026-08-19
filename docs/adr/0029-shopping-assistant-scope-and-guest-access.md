---
doc_type: adr
status: accepted
---

# ADR-0029: New `shopping-assistant.act` Scope, Customer-Role-Only, With an Explicit Guest-Read-Only Carve-Out

## Status

Accepted

## Context

`docs/services/kart-shopping-assistant-service/requirement-spec.md` §10 (Security-2) flags a second-tier authorization check this capability needs but that no existing document names — the requirement-spec recommends `shopping-assistant.act` "by analogy to ADR-0025's `ai-assistant.query`," but explicitly leaves the name/mechanism as a human/architecture-agent decision, "the same category ADR-0025 itself closed for the sibling service."

Unlike `kart-ai-assistant-service` (which, per ADR-0025, has no unauthenticated use case at all — every caller is an `Admin`/`Support Agent` JWT), this service has a genuine mixed-access shape: requirement-spec.md §1.2 states "a guest (unauthenticated) session may use read-only intents only... every mutating intent requires an authenticated `Customer` JWT" — and §2's intent catalog marks rows 5 (product discovery), 6 (compare), and part of 12 (general Q&A) as legitimately guest-accessible, mirroring the platform's own anonymous-browsing posture for `kart-search-service`/`kart-product-service` (`kart-requirements.md` §4.4). ADR-0025's own reasoning ("no separate resource beyond the single action, so a 2-segment scope is the honest shape") is reusable here, but its underlying assumption — that every caller reaching the service already holds the scope-gating role — does not transfer cleanly, and this ADR must resolve that difference explicitly rather than silently copying ADR-0025's all-or-nothing gate.

This also needs the same OpenAPI security-scheme-convention decision ADR-0025 made for its own service: whether this service's contract uses the `bearerAuth` (human JWT forwarded through the Gateway) shape or the `internalClientCredentials` (service-principal Client-Credentials) shape.

## Decision

**New scope name: `shopping-assistant.act`** — adopted verbatim as the requirement-spec's own working recommendation, following ADR-0025's exact reasoning for why a 2-segment `<service>.<action>` scope (not a stricter 3-segment `<service>.<resource>.<verb>` form) is the honest shape: this service exposes one conversational action surface, not a resource collection with independently-gatable verbs.

**Issuance mechanism: embedded in the human `Customer`'s JWT at token-mint time, via Identity's existing role→scope mapping — no new persisted grant table.** `kart-identity-service`'s role-resolution step adds `shopping-assistant.act` to the `scopes` claim for the `Customer` role only (never `Admin`, `Support Agent`, or `Partner API` — mirroring ADR-0028's explicit non-reopening of the sibling service's role gate in the other direction). As with `ai-assistant.query`, there is no per-resource grant to persist: every authenticated `Customer` who can reach the feature at all gets the same scope; per-resource ownership (which order, which cart, which address) is enforced downstream by each service's own existing check (`kart-requirements.md` §24.1.2/§24.1.4), not by this scope.

**Guest access is an explicit carve-out, not a gap in this ruling.** This service's Gateway route (per ADR-0028) accepts both anonymous and `Customer`-authenticated traffic — the Gateway's coarse check for this route is "reject `Admin`/`Support Agent`/`Partner API` JWTs" (the mirror image of `ai-assistant.query`'s route, which rejects everyone *except* those two roles), not "require `Customer`." The finer distinction is enforced at this service's own boundary (Tier 2), intent-by-intent:

| Request shape | Tier 1 (Gateway) | Tier 2 (this service) | Tier 3 (downstream) |
|---|---|---|---|
| Anonymous caller, read-only intent (discovery, compare, general Q&A — requirement-spec.md §2 rows 5, 6, 12) | Allowed through (no role required) | Allowed — no `shopping-assistant.act` scope present or required for a read-only intent | Downstream read-only endpoints already permit anonymous callers (`kart-search-service`, `kart-product-service`) |
| Anonymous caller, mutating intent (any row marked "Yes" in §2) | Allowed through (Gateway cannot distinguish intent shape at the routing layer) | **Rejected before the LLM is invoked** (requirement-spec.md FR-001) — explicit sign-in prompt returned, no `shopping-assistant.act` scope present | Never reached |
| `Customer`-authenticated caller, any intent | JWT carries `Customer` role, not `Admin`/`Support Agent`/`Partner API` | JWT `scopes` claim carries `shopping-assistant.act` — checked once per turn, not per-intent, since every authenticated `Customer` holds it uniformly | Each downstream call carries the forwarded `Customer` JWT/assertion; that service's own ownership check gates the specific resource (ADR-0028's data-ownership table; requirement-spec.md §5's "no cross-customer action" invariant) |

This is the platform's existing three-check flow (`kart-requirements.md` §24.1.3), restated with every check's concrete gate named, including the one structural difference from ADR-0025's all-or-nothing gate: **Tier 1 here is a role-exclusion, not a role-requirement**, because this service — unlike `kart-ai-assistant-service` — has a genuine anonymous-caller population.

**API contract convention: `bearerAuth`, not `internalClientCredentials`** — same reasoning ADR-0025 gave: the caller is a human JWT forwarded through the Gateway, not a service-principal token. The scheme additionally must tolerate an **absent** bearer token for read-only-eligible routes, which `kart-admin-web`'s all-authenticated-caller convention never had to express:

```yaml
securitySchemes:
  bearerAuth:
    type: http
    scheme: bearer
    bearerFormat: JWT
    description: >
      Identity-issued RS256 JWT carrying a Customer role claim plus the
      `shopping-assistant.act` scope (ADR-0029), forwarded unchanged by the
      API Gateway. Optional for read-only-eligible requests (anonymous
      callers permitted); required for any request that resolves to a
      mutating intent — enforced by this service's own application code
      (FR-001), not by the OpenAPI security requirement itself, since a
      single conversational endpoint serves both guest and authenticated
      traffic and the intent shape is not known until after NL resolution.
paths:
  /v1/shopping-assistant/query:
    post:
      security:
        - bearerAuth: [shopping-assistant.act]
        - {}   # anonymous also permitted; mutating-intent rejection is enforced in application code, not at this layer
```

## Consequences

- Requirement-spec.md §10 Security-2 is closed: the scope's name (`shopping-assistant.act`), issuance mechanism (Identity role→scope mapping for `Customer` only, no grant table), and the guest-vs-authenticated enforcement split are all decided, not left to the DDD/API Design Agents to invent independently.
- `kart-identity-service`'s own role→scope mapping config gains one line item: the `Customer` role resolves to include `shopping-assistant.act` in the minted JWT's `scopes` claim. This is a config addition to an already-approved mechanism, not a new mechanism — no `kart-identity-service` ADR or contract change is required (mirrors ADR-0025's own equivalent consequence).
- The API Design Agent gates `POST /v1/shopping-assistant/query` with the dual `security` requirement shape above (`bearerAuth` with the scope, or anonymous), and documents in the endpoint's own description that mutating-intent rejection for an unauthenticated caller is an application-level 401/403 (FR-001), not an OpenAPI-level `403` on the route itself.
- The DDD Agent models authorization against this scope, not a persisted grant/permission aggregate — there is no `PermissionGrant`-shaped entity to design here, consistent with ADR-0028's data-ownership table (this service owns only `ConversationSession`/`AuditRecord`).
- `kart-ai-assistant-service`'s own scope (`ai-assistant.query`, ADR-0025) is unchanged — this ADR adds a new scope for a new, disjoint caller population; it does not touch the sibling service's security model.

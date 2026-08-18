---
doc_type: architecture
service: kart-admin-web
status: approved
approval_note: >
  The kart-ai-assistant-service dependency addition is approved per
  explicit human pipeline directive — it is reached only via the Gateway
  (no violation of the "never bypass the gateway" rule), matching the
  cumulative container-diagram edge already added during that service's
  own architecture pass.
generated_by: human-authored (architecture-agent equivalent, client tier)
source: docs/client/kart-admin-web/requirement-spec.md
---

# Architecture: kart-admin-web

## Boundary Rationale

Same shape as [`kart-web`](../kart-web/architecture.md) — a thin, fan-out `kart-api-gateway` consumer with no domain logic of its own, holding none of `kart-admin-service`'s actual authorization decisions client-side (`requirement-spec.md` §5). It differs from `kart-web` in traffic profile and rendering strategy (client-side rendered SPA, no SSR — `requirement-spec.md` §2), not in its position in the platform's dependency graph.

## Dependencies

| Direction | Peer | Mechanism | Type | Notes |
|---|---|---|---|---|
| Outbound | `kart-api-gateway` | REST | **Sync** | All backend calls, same "never bypass the gateway" rule as `kart-web` |
| Outbound | `kart-api-gateway` (WS/SSE, if adopted) | Real-time | **Async, push** | Optional at launch — an audit/action feed could be real-time, but nothing in §3 requires it day one; add only when a concrete workflow needs it (e.g., live fulfillment-exception queue) |
| Outbound | Enterprise IdP | SAML/OIDC federation, browser redirect | **Sync** | `Admin` login only, per `system-context.md`'s existing edge |
| Outbound | `kart-ai-assistant-service` (new, ADR-0024) | REST, `POST /v1/ai-assistant/query`, via `kart-api-gateway` only | **Sync** | New dependency edge, added for §3.6 (AI Assistant). Reached exclusively through `kart-api-gateway` — this app never opens a direct address to `kart-ai-assistant-service`, so the "never bypass the gateway" rule above is not violated, only extended to a new peer behind the same Gateway (`docs/requirements/genai-business-assistant-spec.md` §10.4). |

## Distributed-Monolith Risk

None, for the same structural reason as `kart-web` — the Gateway is the only address this app knows.

## Repository Folder Structure

```
kart-admin-web/
├── src/
│   ├── app/
│   │   ├── core/
│   │   │   ├── auth/                  # SSO federation flow (Admin) + native login (Support Agent); idle-timer/warning-modal/multi-tab sync per ../security.md §2.2
│   │   │   ├── http/                  # generated Gateway API clients
│   │   │   └── config/
│   │   ├── shared/
│   │   │   └── ui/                     # consumes @kart/design-system (../design-system.md) — kart-web's brand tokens via the shared package, not a direct repo dependency
│   │   ├── features/
│   │   │   ├── catalog-management/     # §3.1
│   │   │   ├── order-exceptions/       # §3.2
│   │   │   ├── support-console/        # §3.3, Support Agent's capped surface — includes the Refund Requests queue
│   │   │   ├── identity-admin/         # §3.4
│   │   │   ├── audit-compliance/       # §3.5, extended with a Privacy Requests view (../privacy.md §B.9)
│   │   │   └── ai-assistant/           # §3.6 (new) — chat panel + table/chart rendering surface, calls kart-ai-assistant-service via the Gateway only
│   │   └── app.routes.ts               # route guards reflect role, per requirement-spec.md §5 — UX only, not enforcement
├── tests/
├── Dockerfile
└── .github/workflows/ci.yml
```

`support-console/` is deliberately separate from the four `Admin`-only feature folders — it's the one area a `Support Agent` (a different, capped role) actually uses, and keeping it isolated makes it easy to verify that role's surface never accidentally gains a link into an `Admin`-only feature.

`ai-assistant/` sits alongside the other five feature folders, not nested inside either the `Admin`-only group or `support-console/` — per `requirement-spec.md` §3.6, `Admin` and `Support Agent` have identical access to this feature area, so it needs no role-based isolation the way `support-console/` does.

## Hosting / Deployment

Standard SPA hosting (static build served from a CDN or a lightweight origin, no SSR runtime tier needed per `requirement-spec.md` §2) — a simpler, cheaper deployment topology than `kart-web`'s, appropriate to its internal, lower-traffic profile. Same CI/CD gate shape (`kart-devops` reusable workflow), scaled-down where a gate is genuinely not applicable (e.g., no CDN-edge cache-header verification needed the way `kart-web`'s public assets require).

## Sign-off

- [ ] Reviewed by: _pending_

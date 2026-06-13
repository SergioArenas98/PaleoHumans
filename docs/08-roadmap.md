# 08 — Roadmap

> **How the roadmap is organised.** This file owns the **cross-cutting**
> roadmap — items that span the frontend ↔ backend boundary, chiefly the shared
> API contract wishlist. Workspace-internal gaps live in
> [frontend/07-roadmap.md](./frontend/07-roadmap.md) and
> [backend/07-roadmap.md](./backend/07-roadmap.md).

## Cross-cutting API contract wishlist

These are backend API changes the frontend wants. The "status" column records
the backend's verified implementation state; "not implemented" items are the
active wishlist. Verify against the live backend before scheduling.

| Item | Why | Status |
|---|---|---|
| `GET /api/individuals/list` projection | shrink the public list payload | **implemented** |
| `GET /api/sites/countries` | populate the country dropdown without paging the table | **implemented** |
| `GET /api/sites?country=…` filter | global country filter on the Sites list | **implemented** |
| `cultureId` filter on `/api/archaeological-contexts` | context filtering by culture | **implemented** |
| `country` filter on `/api/individuals` | the public Individuals list cannot offer a global country filter today | not implemented |
| `cultureId` filter on `/api/individuals` | same as country | not implemented |
| `lastUpdatedAt` populated on `HomeStats` | the frontend already displays it when present; the line is hidden while the backend omits it | not implemented (field exists; population `Requires code verification`) |
| `GET /api/individuals/{id}/bundle` (or `?expand=`) | collapse the `/individuals/:id` Phase-3 dating fan-out into one request | not implemented |
| `GET /api/stats/analysis` (or per-chart aggregates) | let `/analysis` stop sweeping the full individuals collection | not implemented |
| Multi-id `GET /api/dated-samples` (`boneIds=`, `skeletonIds=`, `archaeologicalContextIds=`) | batch the dating fan-out | not implemented |
| `bone-site-search` summary/inventory split | avoid shipping full `units[]` previews for collapsed rows | not implemented |
| Calibrated BP range field on dating DTOs (`calBpMin`/`calBpMax`) | UI features needing calibrated dates | not implemented (requires schema change) |
| Aggregate count fields on `SiteResponse` | avoid a separate stats call for per-site counts | not implemented (counts come from stats / bone-site-search) |
| Optional admin-search domains (Bones, Skeletons, Funerary Contexts, Dating, Users) + enum-facet matching | broaden the backoffice top-bar search | not implemented |

## Cross-cutting performance / infrastructure

These affect perceived performance but live at the deployment edge or span both
sides:

- **`Cache-Control: public, max-age=…` honoured end-to-end.** The backend emits
  cache headers on read-only endpoints; the hydration transfer cache and edge
  caches only help if the edge honours them. Largest visible cost:
  `GET /api/stats/map-timeline` (~130 KB) re-downloading on every map visit.
- **Brotli (`Content-Encoding: br`) on JSON.** Spring Boot negotiates gzip only;
  Brotli needs a separate filter or an edge layer. The heaviest payloads
  (map/timeline stats, bone-site-search, the `/analysis` sweep) would benefit
  most.
- **Distributed rate limiting.** The in-app `AuthRateLimiter` /
  `PublicEndpointRateLimiter` are JVM-local; a multi-instance deployment needs a
  shared limiter (Redis, gateway, or reverse proxy).

## Out of scope (platform level)

- Database schema changes belong in the backend (`database.sql` + the
  `docs/database/schema.sql` reference copy) — see [04-database.md](./04-database.md).
- Cross-app shared libraries between `projects/web` and `projects/backoffice`:
  the duplication is intentional today.
- Direct client-to-database access: the schema assumes the backend is the only
  DB client (Row Level Security would have to be designed first).

## Workspace roadmaps

- **Frontend gaps** (public-web TODOs, hardcoded stats, Chart.js CDN, dead
  services, speculative design directions): [frontend/07-roadmap.md](./frontend/07-roadmap.md).
- **Backend gaps** (stale legacy security/cache paths, no migration tool, no
  OpenAPI doc, JVM-local rate limiting, test-coverage gaps, technical debt):
  [backend/07-roadmap.md](./backend/07-roadmap.md).

# 02 — Architecture

## Workspace structure

The repository is an **Angular CLI multi-project workspace** with two standalone applications:

```text
paleohumans/                          ← workspace root
├── angular.json                      ← project definitions for both apps
├── package.json                      ← shared dependencies
├── tsconfig.json                     ← shared compiler base
├── projects/
│   ├── web/                          ← public-facing site (SSR)
│   │   └── src/app/
│   │       ├── app.routes.ts         ← public route table
│   │       ├── app.routes.server.ts  ← per-route SSR mode (Server / Prerender)
│   │       ├── app.config.ts         ← bootstrap providers
│   │       ├── pages/                ← route-level components
│   │       ├── features/             ← per-domain (model + service)
│   │       ├── components/           ← header, footer, layout shell, map
│   │       ├── core/                 ← interceptors, shared logic
│   │       ├── utils/                ← delayed loading, map preload, markers
│   │       └── environments/         ← environment.ts / environment.prod.ts
│   └── backoffice/                   ← internal admin app (SSR)
│       └── src/app/
│           ├── app.routes.ts         ← admin route table (guarded under /admin)
│           ├── pages/                ← CRUD pages per entity
│           ├── features/             ← per-domain (model + service)
│           ├── core/auth/            ← JWT store, service, interceptor, guard
│           ├── layout/               ← admin shell
│           ├── shared/               ← shared admin utilities
│           └── environments/
├── docs/                             ← canonical knowledge base (this folder)
└── docs/database/                    ← upstream database documentation (read-only)
```

Workspace definitions live in `angular.json`:

- `projects.web` → builder `@angular/build:application`, `outputMode: server`, SSR entry `projects/web/src/server.ts`.
- `projects.backoffice` → same builder, same SSR mode, also imports Leaflet CSS globally (`leaflet.css` + `leaflet.markercluster` CSS).

Tests run on **Vitest** via `@angular/build:unit-test`.

## Angular conventions

- **Angular 21**, standalone components throughout. **No NgModules anywhere.**
- Services use `inject()` and `providedIn: 'root'`.
- HTTP layer is `HttpClient` with `withFetch()` and the public web also installs a `retryInterceptor` (`projects/web/src/app/core/interceptors/retry-interceptor.ts`). The backoffice installs the JWT auth interceptor (`admin-auth.interceptor.ts`).
- Templates prefer the modern control-flow blocks (`@if`, `@for`, `@switch`) except `analysis-page.html` which still uses `*ngIf`/`*ngFor` (see [10-roadmap.md](./10-roadmap.md)).
- State is **component-local** by default. The only persistent signal-based store is `AdminAuthStore` in the backoffice.

## Rendering model

Both apps build with `@angular/build:application` and `outputMode: server`, so an SSR Node entry is produced. The **per-route rendering mode** for the public web is defined in `projects/web/src/app/app.routes.server.ts`:

| Route | Render mode | Rationale |
|---|---|---|
| `sites/:id` | `RenderMode.Server` | Parametric; rendered on demand. |
| `individuals/:id` | `RenderMode.Server` | Parametric; rendered on demand. |
| `analysis` | `RenderMode.Server` | Data-heavy; avoids long build-time API calls. |
| `**` (everything else) | `RenderMode.Prerender` | Static prerender at build time. |

SSR is **non-negotiable** for SEO. Browser-only APIs (`window`, `document`, `localStorage`, Leaflet, `IntersectionObserver`) must be guarded with `isPlatformBrowser(PLATFORM_ID)`.

The public web enables `provideClientHydration(withEventReplay())` in `app.config.ts`. The previous opt-out via `withNoHttpTransferCache()` was removed; SSR/SSG-fetched payloads now survive hydration. This depends on the backend returning correct `Cache-Control` headers (see [08-performance.md](./08-performance.md)).

## External backend

The backend is an **external Spring Boot 3.5.6 / Java 17 REST API** in a separate repository.

| Setting | Value |
|---|---|
| Base URL (dev) | `http://127.0.0.1:8080/api` (configured in each project's `environment.ts`) |
| Auth | Stateless JWT (`/api/auth`), refresh tokens rotated on use |
| PATCH semantics | `JsonNullable` — absent = no-op, `null` = clear, value = update |
| Pagination | Standard envelope: `content`, `page`, `size`, `totalElements`, `totalPages`, `first`, `last`, `hasNext`, `hasPrevious`, `sort` |

Frontend services always strip `undefined` keys from PATCH payloads (`cleanPatch` helper inside each service / in `projects/backoffice/src/app/features/shared/utils/clean-patch.ts`).

The full endpoint catalog is in [04-backend-api-contract.md](./04-backend-api-contract.md).

## Auth boundaries

The two apps treat auth opposite ways.

### Public web (`projects/web`)

- **No auth layer.** No interceptor adds an `Authorization` header. The retry interceptor only retries idempotent failed GETs.
- All endpoints used by the public web are confirmed **publicly accessible GETs** (see [04-backend-api-contract.md](./04-backend-api-contract.md)).
- No route guards, no login page, no session storage.

### Backoffice (`projects/backoffice`)

- **JWT Bearer required** for every `/api/*` request.
- Auth state is held in `AdminAuthStore` (signal-based), persisted to `localStorage` under the key configured in `environment.sessionStorageKey`.
- `AdminAuthService` handles login (`POST /api/auth/login`), logout, and proactive refresh.
- `adminAuthInterceptor` attaches `Authorization: Bearer <token>` to API calls, transparently retries once on 401 by calling `AuthService.refreshAccessToken()`, and force-signs-out on a failed refresh.
- `adminAuthGuard` protects all routes under `/admin` except `/admin/login`. `adminAnonymousGuard` redirects authenticated users away from `/admin/login`.
- The admin app talks to two endpoint families: the public `/api/*` for read access and the admin-only `/api/admin/*` for lightweight paginated list projections (`/admin/stats/dashboard`, `/admin/sites`, `/admin/individuals`, `/admin/bones`).

## Data flow (public web)

The dominant pattern is **route activates immediately → component subscribes to the relevant `*Service` → renders progressively**. There are **no route resolvers** anywhere in the public web; the previous resolver-based flow was replaced because it blocked navigation while heavy collections loaded.

Typical sequence on `/individuals/:id`:

1. Route activates. `IndividualDetailPageComponent` reads `params.id` via `paramMap.switchMap(...)`.
2. **Phase 1 (primary).** `IndividualService.getById(id)` → renders shell, biological profile, archaeological context, funerary contexts.
3. **Phase 2 (enrichment, in parallel via `forkJoin`).** `BoneService.findByIndividual(id)`, `SkeletonService.search({ individualId })`, `ArchaeologicalContextService.getById(contextId)`.
4. **Phase 3 (dating, after Phase 2).** `DatedSampleService.search({ archaeologicalContextId | boneId | skeletonId })` fanned out per linked entity; results carry the full `datingResults[]` payload so no separate `/dating-results` call is needed.

Each enrichment branch carries its own error signal so a single failure does not blank the page. The full per-page contract is in [05-public-web.md](./05-public-web.md).

## Data flow (backoffice)

CRUD pages call the relevant service directly. List pages use the lightweight `/api/admin/<entity>` paginated projections; detail pages use the standard `/api/<entity>/{id}` endpoint to load full data and call PATCH on save. Multi-entity flows that bundle two records (e.g. dating: `DatedSample` + `DatingResult`) are orchestrated by composite services such as `DatingRecordService` (`projects/backoffice/src/app/features/dating-records/services/dating-record-service.ts`).

## Important architectural constraints

| Constraint | Why |
|---|---|
| SSR must be preserved. | Public site must be indexable by academic search engines and rendered server-side for first paint. |
| No NgModules. | Workspace is fully standalone-components; introducing one is a regression. |
| Backend API contract is fixed. | The Spring repo is upstream; renaming endpoints/fields or inventing filters requires a backend change request. |
| No global state libraries. | Component-local state + signals are enough for current scope; introducing NgRx or similar requires deliberate review. |
| `projects/web` and `projects/backoffice` are isolated. | Domain models are duplicated on purpose; no shared library yet. Do not refactor shared types across the two apps. |
| Material Symbols for icons. | Used for backoffice and parts of the public web; replacement is a design decision tracked in [07-design-system.md](./07-design-system.md). |
| Leaflet + OpenTopoMap/OpenFreeMap for the public map. | Browser-only; SSR must guard with `isPlatformBrowser`. Leaflet/MapLibre structural CSS is imported at the **component** level by `MapPage` and `SiteDetailsPage` so other public routes do not pay for it. OpenFreeMap is rendered through MapLibre GL Leaflet and loaded only in the browser. |
| Chart.js for analysis charts. | Loaded dynamically at runtime inside `AnalysisPage`. See [10-roadmap.md](./10-roadmap.md). |

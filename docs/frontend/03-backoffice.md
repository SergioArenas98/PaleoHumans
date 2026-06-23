# 06 — Backoffice (`projects/backoffice`)

The backoffice at `projects/backoffice` is the admin CRUD application for project editors. It is a standalone Angular 21 app behind a JWT login.

> **Hard separation.** The public web redesign is the active focus. The backoffice must not be visually or behaviorally affected by changes to `projects/web`. Domain models are duplicated between the two apps on purpose; there is no shared library. Do not refactor shared types across the boundary.

## Purpose

The backoffice exists to **curate the database**. It exposes:

- Full Create / Read / Update / Delete for every domain entity exposed by the Spring backend.
- Relational editing UI (link / unlink) for every M2M relation that the backend supports as a single PATCH.
- A dashboard with lightweight scalar counts.
- A user management surface for admin accounts and self-service password change.

The backoffice is **not** designed for consumption by researchers or the public.

## Auth

| Concern | Location | Notes |
|---|---|---|
| Token store | `core/auth/admin-auth.store.ts` | Signal-based, persisted in `localStorage` under `environment.sessionStorageKey`. |
| Auth service | `core/auth/admin-auth.service.ts` | Login, logout, proactive `refreshAccessToken()`. |
| Interceptor | `core/auth/admin-auth.interceptor.ts` | Adds `Authorization: Bearer <token>` for API calls, transparently refreshes on 401 and force-signs-out on a failed refresh. |
| Guards | `core/auth/admin-auth.guard.ts` | `adminAuthGuard` protects all `/admin/*` except login; `adminAnonymousGuard` redirects authenticated users away from `/admin/login`. |
| Models | `core/auth/admin-auth.models.ts` | `normalizeRole` / `pickRole` map Spring `ROLE_ADMIN` / `ROLE_USER` → canonical `ADMIN` / `USER`; unknown roles collapse to `USER`. |

The frontend treats the backend as the **single source of truth** for authorisation. Frontend role checks are convenience-only; the backend re-validates every request.

## Route map

All routes are registered in `projects/backoffice/src/app/app.routes.ts`. Every domain has the same three-route shape: list, create, detail/edit.

| Domain | List | Create | Detail | Notes |
|---|---|---|---|---|
| Dashboard | `/admin/dashboard` | — | — | Counts via `GET /api/admin/stats/dashboard`. |
| Sites | `/admin/sites` | `/admin/sites/new` | `/admin/sites/:id` | Reference link/unlink UI on detail. |
| Cultures | `/admin/cultures` | `/admin/cultures/new` | `/admin/cultures/:id` | `features` captured as textarea, split on newline. |
| References | `/admin/references` | `/admin/references/new` | `/admin/references/:id` | Detail has a **read-only** "Linked sites and contexts" panel (from `GET /api/admin/references/:id/relations`); linking/unlinking ownership stays on the Site/Context side. A reference-type selector (Journal article / Article-chapter in book / Complete book / Unspecified) drives `isBook` + `isArticleInBook`; the form also manages `editor`, `book`, `corporateAuthor`, `city` and shows a live citation preview. Only `authors` is required (`title`/`journal` are optional). |
| Archaeological Contexts | `/admin/archaeological-contexts` | `…/new` | `…/:id` | Site/culture/reference editing. v1 `/admin/osteological-units` redirects here. |
| Individuals | `/admin/individuals` | `…/new` | `…/:id` | Attach/detach archaeological context; FK-based, no link panel. |
| Bones | `/admin/bones` | `…/new` | `…/:id` | Conditional fields by catalog category. Individual association is **M2M** (`bone_individual`): create/detail use a one-or-more individual multi-select, not a single FK. |
| Bone Catalog | `/admin/bone-catalog` | `…/new` | `…/:id` | Hierarchical parent/component editing. Backend path is `/api/bone-catalog`. |
| Skeletons | `/admin/skeletons` | `…/new` | `…/:id` | `individualId` FK required at creation. |
| Funerary Contexts | `/admin/funerary-contexts` | `…/new` | `…/:id` | Individual multi-select grid. v1 `/admin/burial-groups` redirects here. |
| Dating | `/admin/dating` | `/admin/dating/new` | `/admin/dating/:id` | Unified `DatedSample` + `DatingResult` record via the transactional `/api/admin/dating-records` aggregate (`DatingRecordService`). |
| Dating Techniques | `/admin/dating-techniques` | `…/new` | `…/:id` | — |
| Users | `/admin/users` | `…/new` | `/admin/users/:id` plus `/admin/users/me` | Self-service password change at `/me`. |

Compatibility redirects (kept so old bookmarks still resolve):

```
/admin/osteological-units   → /admin/archaeological-contexts
/admin/burial-groups        → /admin/funerary-contexts
/admin/dates                → /admin/dating
/admin/dated-samples        → /admin/dating
/admin/dated-samples/new    → /admin/dating/new
/admin/dated-samples/:id    → /admin/dating
/admin/dating-results       → /admin/dating
/admin/dating-results/new   → /admin/dating/new
/admin/dating-results/:id   → /admin/dating/:id
```

The catch-all redirect (`**` → `admin`) lands unauthenticated users at the protected shell, which then bounces them to `/admin/login` via the auth guard.

## Layout

`AdminShellComponent` (`projects/backoffice/src/app/layout/admin-shell.component.ts`) wraps all authenticated pages. It renders the sidebar navigation, the top bar, and the routed `<router-outlet>`.

`LoginPageComponent` (`projects/backoffice/src/app/pages/login/login-page.component.ts`) is rendered outside the admin shell.

## CRUD pattern

Every list page uses the lightweight `/api/admin/<entity>` projection where one exists. Currently the lightweight admin list endpoints in active use are `/api/admin/sites`, `/api/admin/individuals`, `/api/admin/bones`. Other domains use the standard `/api/<entity>` paginated/list endpoint.

Detail pages use the full `/api/<entity>/{id}` to populate the form. On save they call PATCH; payloads go through `cleanPatch` (`features/shared/utils/clean-patch.ts`) so absent keys mean no-op, explicit `null` means clear, and present values overwrite (matching the backend's `JsonNullable` semantics).

Create pages call POST and navigate to the new detail page on success.

## Relational editing

| Relation | Owning page | Inverse attach UI |
|---|---|---|
| Site ↔ Reference (M2M) | Site detail | Reference detail — read-only "Linked sites and contexts" panel (`GET /api/admin/references/:id/relations`) |
| ArchaeologicalContext.site (FK) | Context detail select | — |
| ArchaeologicalContext.culture (FK) | Context detail select | — |
| ArchaeologicalContext ↔ Reference (M2M) | Context detail | Reference detail — same read-only relations panel |
| Individual.archaeologicalContext (FK) | Individual detail / create select | — |
| Bone ↔ Individual (M2M via `bone_individual`) | Bone detail / create — one-or-more individual multi-select | — |
| Bone.boneCatalog (FK) | Bone detail / create select | — |
| Skeleton.individual (FK) | Skeleton detail / create select | — |
| FuneraryContext.archaeologicalContext (FK) | Funerary context detail select | — |
| FuneraryContext ↔ Individual (M2M via `funerary_context_individual`) | Funerary context detail multi-select grid | — |
| DatedSample.archaeologicalContext | bone | skeleton (FK, exactly one) | Dating create/detail | — |
| DatingResult.datedSample (FK) | Dating create/detail (edited together via the `/api/admin/dating-records` aggregate) | — |
| DatingResult.datingTechnique (FK) | Dating detail select | — |

> **Bone ↔ Individual editing.** The bone create form requires **at least one** individual; the detail form edits the full set. On PATCH, a present non-empty `individualIds` array **replaces the whole association set** (replace-set semantics, matching the backend's `JsonNullable`); an empty selection is rejected. The admin list (`AdminBoneListResponse`) surfaces `individualCount`, not a single individual. Skeletons remain a single-`individualId` FK at creation.

The dating editor (`/admin/dating/:id`) is a unified form for the `DatedSample` + `DatingResult` record via `DatingRecordService` (`projects/backoffice/src/app/features/dating/dating-record.service.ts`), a thin client over the transactional `/api/admin/dating-records` aggregate endpoint. Create, update (PATCH; a present `datingResults` replaces the whole child set) and delete are each a single atomic backend call — the record can no longer be left partial.

## Shared admin UI

- `NoticeStore` — global toast/notice store, exposes success/error/info banners.
- `ConfirmStore` — delete-confirmation dialogs.
- `FormErrorComponent` — renders RFC 7807 Problem Detail responses.
- `AdminDatePipe` — formats ISO timestamps for display.
- `LoadingStateComponent` / `EmptyStateComponent` / `ErrorStateComponent` — list-page state placeholders.
- `cleanPatch()`, `textOrNull()` — pure helpers in `features/shared/utils/clean-patch.ts`.

## Backend dependencies

- Every admin endpoint requires `ROLE_ADMIN`. CORS includes the dev origin (`localhost:*`); production allowance is verified against the live `SecurityConfig` (`Requires code verification`).
- The dashboard endpoint `GET /api/admin/stats/dashboard` returns scalar counts; if the field set changes on the backend, update `AdminDashboardStats` in `projects/backoffice/src/app/features/admin-stats/model/AdminDashboardStats.ts`.
- Some admin list pages still call the public `/api/<entity>` endpoints because no `/api/admin/<entity>` projection exists yet (e.g. cultures, references, skeletons, funerary contexts, dating). Adding lightweight admin projections is a backend task tracked in [07-roadmap.md](./07-roadmap.md).

## Warning to redesign work

If you are working on the **public web** redesign:

- Do not import anything from `projects/backoffice/src/app/*`.
- Do not modify backoffice models or services even if they appear to drift from the public web copies. The duplication is intentional and the two apps evolve at different speeds.
- Do not change CSS tokens under the `--t-*` namespace; those belong to the backoffice TERRAIN theme. The public web uses the `--dt-*` (DEEP TIME) namespace.
- The `projects/backoffice/src/styles.css` file holds the backoffice design tokens (light parchment + ochre accent) and must not import the public web tokens.

If you are working on the **backoffice**:

- Stay inside `projects/backoffice/src/app/*` and `projects/backoffice/public/*`.
- Use the existing CRUD and shared UI patterns; do not introduce new shared abstractions across `web` and `backoffice` unless the change is explicitly scoped.
- When the backend ships new admin list projections, prefer them over the heavy `/api/<entity>` endpoints to keep list pages fast.

# 05 — API Contract

> **Canonical REST contract.** This is the single source of truth for the HTTP
> endpoints the Spring Boot backend exposes and the two Angular apps consume.
> Security details (the public/admin allowlist, JWT, rate limits) are summarised
> here and fully specified in
> [backend/02-security-and-auth.md](./backend/02-security-and-auth.md).
> Validation rules live in
> [backend/03-validation-and-errors.md](./backend/03-validation-and-errors.md).
> The live backend response is the final arbiter; the controller signatures plus
> this catalogue are the source of truth (no OpenAPI document is generated).

## Conventions

- **Base path.** `WebConfig.configurePathMatch` prefixes every controller
  mapping with `/api`. A controller annotated `@RequestMapping("/sites")`
  responds at `/api/sites`. The dev base URL is `http://127.0.0.1:8080/api`
  (configured per app in `environment.ts`); production overrides live in
  `environment.prod.ts`.
- **Auth.** Stateless JWT bearer. Public endpoints accept no token; admin
  endpoints require a token with `ROLE_ADMIN`.
- **Caller column.** `[PUBLIC]` = called by `projects/web`; `[ADMIN]` = called
  by `projects/backoffice`; `[BOTH]` = both. This reflects current frontend
  usage; the **authority on public-vs-admin access** is the backend
  `SecurityConfig` allowlist (see
  [backend/02-security-and-auth.md](./backend/02-security-and-auth.md)).
- **Pagination envelope.**

  ```ts
  interface PaginatedResponse<T> {
    content: T[];
    page: number; size: number;
    totalElements: number; totalPages: number;
    first: boolean; last: boolean;
    hasNext: boolean; hasPrevious: boolean;
    sort: string[];
  }
  ```

  `page` is zero-based (default `0`, `>= 0`); `size` default `20`, max `100`
  (`common/util/PaginationUtils.MAX_PAGE_SIZE`); `sort` is one or more
  `field` or `field,direction` parameters accumulated via repeated `sort=`.
  Allowed sort fields are declared per service; unknown fields return `400`.
- **PATCH semantics (`JsonNullable`).** Key absent → unchanged; key `null` →
  cleared (where the column allows); key with value → set. The frontend strips
  `undefined` keys via a per-service `cleanPatch()` helper before sending.

## Controllers

| Controller | URL prefix |
| --- | --- |
| `AuthController` | `/api/auth` |
| `SiteController` / `AdminSiteController` | `/api/sites` · `/api/admin/sites` |
| `CultureController` | `/api/cultures` |
| `ArchaeologicalContextController` | `/api/archaeological-contexts` |
| `IndividualController` / `AdminIndividualController` | `/api/individuals` · `/api/admin/individuals` |
| `BoneCatalogController` | `/api/bone-catalog` |
| `BoneController` / `AdminBoneController` | `/api/bones` · `/api/admin/bones` |
| `SkeletonController` | `/api/skeletons` |
| `DatedSampleController` | `/api/dated-samples` |
| `DatingResultController` | `/api/dating-results` |
| `DatingRecordController` | `/api/admin/dating-records` |
| `DatingTechniqueController` | `/api/dating-techniques` |
| `ReferenceController` | `/api/references` |
| `FuneraryContextController` | `/api/funerary-contexts` |
| `StatsController` / `AdminDashboardStatsController` | `/api/stats` · `/api/admin/stats` |
| `AdminSearchController` | `/api/admin/search` |
| `BoneSiteSearchController` | `/api/bone-site-search` |
| `DatasetController` | `/api/dataset` |
| `UserController` | `/api/users` |

Removed v1 paths — do not document or reintroduce: `/api/osteological-units`,
`/api/specimens`, `/api/dates`, `/api/burial-groups`. No controller serves
these. (`SecurityConfig` and `CacheControlInterceptor` still carry dead
`permitAll`/cache entries for them — cleanup tracked in
[backend/07-roadmap.md](./backend/07-roadmap.md).)

## Standard CRUD shape

Domain controllers expose `GET /resource` (paginated list with filter+sort),
`GET /resource/{id}` (detail), `POST /resource` (create, `201` + body), `PATCH
/resource/{id}` (partial update via `*UpdateRequest`, returns updated body), and
`DELETE /resource/{id}` (`204`).

## Auth — `/api/auth`

| Method | Path | Body | Response |
|---|---|---|---|
| POST | `/auth/login` | `{ username, password }` | `{ accessToken, refreshToken }` |
| POST | `/auth/refresh` | `{ refreshToken }` | `{ accessToken, refreshToken }` |
| POST | `/auth/logout` | `{ refreshToken }` | `204 No Content` |

All three are public. The token response carries `accessToken` and
`refreshToken` only (no expiry timestamps in the body). Refresh tokens rotate on
use (old revoked, new issued). `/auth/login` and `/auth/refresh` are
rate-limited per IP (and login also per username). See
[backend/02-security-and-auth.md](./backend/02-security-and-auth.md).

## Stats — `/api/stats`  ·  [PUBLIC]

| Method | Path | Response | Called from |
|---|---|---|---|
| GET | `/stats/home` | `HomeStats` | Home |
| GET | `/stats/map-timeline` | `MapTimelineStats` | Map, Timeline |

`HomeStats`: `totalSites` (`COUNT(*)` over `site`), `totalIndividuals`
(**`SUM(individual.mni_statistical)`** — curated minimum count, not the row
count), `totalBoneRemains` (`SUM(bone.bone_quantity_min)`), `totalSkeletons`
(`COUNT(*)` over `skeleton`), and `lastUpdatedAt` (max `updated_at` across the
public domain tables; `null` when all are empty). The public Home page renders
`lastUpdatedAt` only when present and never fabricates a date.

`MapTimelineStats`: `sites[]`, `units[]` (named `units` for frontend backward
compatibility — each row represents an **archaeological context** in v2, keyed
by `archaeologicalContextId`), and `cultures[]`. Per-site aggregates
(`totalMni`, `individualCount`, `boneCount`, `skeletonCount`,
`datedSampleCount`, `dominantCultureId`, `cultureIds`) are computed
server-side; `boneCount` is `COUNT(DISTINCT bone.bone_id)` joined through
`bone_individual`, so a shared bone counts once. The Map and Timeline pages use
**only** this endpoint for core data and must not load unfiltered
`/api/archaeological-contexts`.

## Sites — `/api/sites`

| Method | Path | Query params | Response | Caller |
|---|---|---|---|---|
| GET | `/sites` | `q`, `country`, `page`, `size`, `sort` | `PaginatedResponse<Site>` | [BOTH] |
| GET | `/sites/countries` | — | `string[]` | [PUBLIC] |
| GET | `/sites/:id` | — | `Site` | [BOTH] |
| POST/PATCH/DELETE | `/sites[/:id]` | — | `Site` / `void` | [ADMIN] |

Default sort `siteName,asc`. `country` is an exact-match filter;
`/sites/countries` returns the deduplicated country list for dropdowns.
`SiteResponse` has **no aggregate counts** — counts come from
`/api/stats/map-timeline` or `/api/bone-site-search`.

## Archaeological Contexts — `/api/archaeological-contexts`

| Method | Path | Query params | Response | Caller |
|---|---|---|---|---|
| GET | `/archaeological-contexts` | `q`, `siteId`, `cultureId`, `page`, `size`, `sort` | `PaginatedResponse<ArchaeologicalContext>` | [BOTH] |
| GET | `/archaeological-contexts/:id` | — | `ArchaeologicalContext` | [BOTH] |
| POST/PATCH/DELETE | `…[/:id]` | — | `ArchaeologicalContext` / `void` | [ADMIN] |

Default sort `archaeologicalContextId,asc`. Response embeds `site`, `culture`,
`individuals[]`, `funeraryContexts[]`, `references[]`, optionally
`datedSamples[]`. Create/update accepts `siteId`, `stratigraphicContext`,
`cultureId`, `referenceIds` (replace-set).

## Individuals — `/api/individuals`

| Method | Path | Query params | Response | Caller |
|---|---|---|---|---|
| GET | `/individuals` | `q`, `sex`, `sexCertain`, `ageClassMain`, `individualType`, `archaeologicalContextId`, `siteId`, `page`, `size`, `sort` | `PaginatedResponse<Individual>` | [BOTH] |
| GET | `/individuals/list` | same filters | `PaginatedResponse<IndividualSummary>` | [PUBLIC] |
| GET | `/individuals/:id` | — | `Individual` | [BOTH] |
| POST/PATCH/DELETE | `…[/:id]` | — | `Individual` / `void` | [ADMIN] |

Default sort `individualId,asc`. `/individuals` returns the full nested payload
(heavy); the public list page should prefer the lightweight
`/individuals/list` projection. `sex` is a strict enum
(`MALE`/`FEMALE`/`INDETERMINATE`); combined with the `sexCertain` Boolean it
powers the UI options `Male`, `Male?`, `Female`, `Female?`, `Indeterminate`,
`Indeterminate?`. There is **no `cultureId` or `country` filter** on
`/individuals` (see [Known absent filters](#known-absent-filters-and-caveats)).

## Bones — `/api/bones`

| Method | Path | Query params | Response | Caller |
|---|---|---|---|---|
| GET | `/bones` | `q`, `boneCatalogId`, `individualId`, `page`, `size`, `sort` | `PaginatedResponse<Bone>` _or_ `Bone[]` _(see note)_ | [PUBLIC] |
| GET | `/bones/:id` | — | `Bone` | [BOTH] |
| POST/PATCH/DELETE | `…[/:id]` | — | `Bone` / `void` | [ADMIN] |

`BoneResponse` exposes `individualIds: number[]` (every associated individual),
not a single `individualId`. The `individualId` **query parameter** resolves
through the `bone_individual` bridge (an `EXISTS` subquery); it returns every
bone associated with that individual, but a returned bone may also be associated
with other individuals — do not treat the result as exclusively owned.

The public web has two helpers: `BoneService.search(...)` returns
`PaginatedResponse<Bone>`, while `BoneService.findByIndividual(individualId)`
issues the same `GET /bones?individualId=X` but **types the response as
`Bone[]`** (the endpoint historically returns a plain array in that mode). Always
call `findByIndividual()` for an individual's full inventory; never read
`.content` off a plain array. `Requires code verification` for the canonical
`/bones` shape (paginated vs array).

Write contract: `BoneCreateRequest.individualIds` is a non-empty `number[]`
(`@NotEmpty`, no orphan bones; duplicates ignored), plus `boneCatalogId`,
`boneSource` (non-blank), `boneQuantityMin` (positive). On PATCH,
`individualIds` uses **replace-set** semantics (absent = unchanged; non-empty
array replaces the whole set; `null` or empty array is rejected). Every
referenced individual must exist and must satisfy remains-exclusivity (no
individual that already holds a skeleton). A bone linked to multiple individuals
is still one bone — totals joining `bone_individual` must count `DISTINCT
bone_id`.

## Bone Site Search — `/api/bone-site-search`  ·  [PUBLIC]

| Method | Path | Response |
|---|---|---|
| GET | `/bone-site-search` | `BoneSiteSearchResponse` |

Filters (all optional): `q`, `boneCatalogId`, `skeletonMain`, `skeletonRegion`,
`boneCategory`, `laterality`, `toothType`, `toothVerticalPosition`,
`toothNumber`, `vertebraType`, `vertebraNumber`, `ribNumber`, `phalanxType`,
`phalanxNumber`, `handFootBoneSegment`, `country`, `cultureId`,
`datingAvailable` (`= SIZE(b.datedSamples) > 0`), plus `page`, `size`, `sort`.
Allowed sort: `siteName`, `country`, `minimumBoneRemainCount`,
`matchingBoneRecordCount`, `individualCount`. Default sort
`minimumBoneRemainCount,desc` then `siteName,asc`.

Implemented as a single JPQL constructor projection over `Bone JOIN
Bone.individuals JOIN BoneCatalog JOIN ArchaeologicalContext JOIN Site LEFT JOIN
Culture`, aggregated in Java by **distinct `bone_id`** (a shared bone counts once
in `matchingBoneRecordCount` / `minimumBoneRemainCount` while every associated
individual is counted in `individualCount`). The repository caps the broad-match
budget (`app.search.bone-site.max-match-rows`); the controller caps page size
(`app.search.bone-site.max-page-size`). Anonymous callers pass through
`PublicEndpointRateLimiter`.

The top-level envelope carries `summary`, `filters`, `content[]` plus the
standard pagination fields. v2 renames inside the response (vs. v1):
`osteologicalUnitCount → archaeologicalContextCount`, `datedSpecimenCount →
datedSampleCount`. This endpoint is the **only** data source for the `/bones`
page; it must never be reconstructed via `/api/archaeological-contexts` or
`/api/bones`.

## Skeletons — `/api/skeletons`

| Method | Path | Query params | Response | Caller |
|---|---|---|---|---|
| GET | `/skeletons` | `q`, `individualId`, `skeletonCategory`, `page`, `size`, `sort` | `PaginatedResponse<Skeleton>` | [BOTH] |
| GET | `/skeletons/:id` | — | `Skeleton` | [BOTH] |
| POST/PATCH/DELETE | `…[/:id]` | — | `Skeleton` / `void` | [ADMIN] |

Required create fields: `individualId`, `skeletonCategory`.

## Funerary Contexts — `/api/funerary-contexts`

| Method | Path | Query params | Response | Caller |
|---|---|---|---|---|
| GET | `/funerary-contexts` | `archaeologicalContextId`, `burialContext`, `individualId`, `page`, `size`, `sort` | `PaginatedResponse<FuneraryContext>` | [BOTH] |
| GET | `/funerary-contexts/:id` | — | `FuneraryContext` | [BOTH] |
| POST/PATCH/DELETE | `…[/:id]` | — | `FuneraryContext` / `void` | [ADMIN] |

Create/update accepts `individualIds` for M2M assignment (replace-set on PATCH).

## Dated Samples — `/api/dated-samples`  ·  [BOTH] (public read)

| Method | Path | Query params | Response |
|---|---|---|---|
| GET | `/dated-samples` | `sampleOrigin`, `datingType`, `archaeologicalContextId`, `boneId`, `skeletonId`, `page`, `size`, `sort` | `PaginatedResponse<DatedSample>` |
| GET | `/dated-samples/:id` | — | `DatedSample` |
| POST/PATCH/DELETE | `…[/:id]` | — | `DatedSample` / `void` ([ADMIN]) |

A `DatedSample` links to exactly one of `archaeologicalContext`, `bone`, or
`skeleton`. Each response embeds the full `datingResults[]` payload — the public
Individual detail page consumes this directly and issues no separate
`/dating-results` call.

## Dating Results — `/api/dating-results`  ·  public read

| Method | Path | Query params | Response |
|---|---|---|---|
| GET | `/dating-results` | `datedSampleId`, `datingTechniqueId`, `page`, `size`, `sort` | `PaginatedResponse<DatingResult>` |
| GET | `/dating-results/:id` | — | `DatingResult` |
| POST/PATCH/DELETE | `…[/:id]` | — | `DatingResult` / `void` ([ADMIN]) |

`DatingResult` carries `datedSampleId`, `datingTechnique{datingTechniqueId,
datingTechniqueName}`, `datesBpUncal`, `datesRange`, `notes`. `GET` is **public**
per the `SecurityConfig` allowlist. The backoffice **no longer mutates these
directly** — it uses the transactional admin aggregate below. **No calibrated BP
field exists.**

## Dating Records (admin aggregate) — `/api/admin/dating-records`  ·  [ADMIN]

A *dating record* is one `DatedSample` + its one-or-more `DatingResult`s,
persisted/updated/deleted in a **single transaction** (it is not a new table).
The public `/api/dated-samples` and `/api/dating-results` reads are unchanged.

| Method | Path | Notes |
|---|---|---|
| GET | `/admin/dating-records` | Paginated `DatingRecordResponse`. Filters: `sampleOrigin`, `datingType`, `archaeologicalContextId`, `boneId`, `skeletonId`, `datingTechniqueId`, `page`, `size`, `sort` (`datedSampleId`, `sampleOrigin`, `datingType`, `material`, `updatedAt`). |
| GET | `/admin/dating-records/:datedSampleId` | Single aggregate. |
| POST | `/admin/dating-records` | Create sample + results in one transaction → `201`. |
| PATCH | `/admin/dating-records/:datedSampleId` | `JsonNullable` patch of sample fields; a present `datingResults` **replaces the whole child set** → `200`. |
| DELETE | `/admin/dating-records/:datedSampleId` | Deletes sample + results atomically → `204`. |

`DatingRecordResponse` is **flat**: origin references are bare ids
(`archaeologicalContextId` / `boneId` / `skeletonId`, exactly one non-null
matching `sampleOrigin`), and each result inlines `datingTechniqueId` +
`datingTechniqueName`. Stable validation codes (`422`):
`DATING_RECORD_INVALID_ORIGIN`, `DATING_RECORD_INVALID_TYPE_FOR_ORIGIN`,
`DATING_RECORD_REQUIRES_RESULTS`, `DATING_RECORD_TECHNIQUE_NOT_FOUND` (see
[backend/03-validation-and-errors.md](./backend/03-validation-and-errors.md)).
Backoffice client: `DatingRecordService`.

## Cultures — `/api/cultures`  ·  [BOTH]

| Method | Path | Query params | Response |
|---|---|---|---|
| GET | `/cultures` | `phase` (optional) | `Culture[]` (unpaginated) |
| GET | `/cultures/:id` | — | `Culture` |
| POST/PATCH/DELETE | `…[/:id]` | — | `Culture` / `void` ([ADMIN]) |

`/cultures` returns an unpaginated list (no `q`/`page`/`size`/`sort`). The full
response has `region`, `description`, `features[]`; the embedded form
(`CultureEmbedResponse`) omits them. The public `CultureService` caches
`getAll()` with `shareReplay`.

## References — `/api/references`  ·  [BOTH]

| Method | Path | Query params | Response |
|---|---|---|---|
| GET | `/references` | `q`, `page`, `size`, `sort` | `PaginatedResponse<Reference>` |
| GET | `/references/:id` | — | `Reference` |
| POST/PATCH/DELETE | `…[/:id]` | — | `Reference` / `void` ([ADMIN]) |

Default sort `year,desc` then `bibliographicReferenceId,asc`. The public
`/bibliography` page consumes the paginated envelope — do not type it as a bare
array.

## Bone Catalog — `/api/bone-catalog`  ·  [BOTH]

| Method | Path | Query params | Response |
|---|---|---|---|
| GET | `/bone-catalog` | `q`, `skeletonMain`, `skeletonRegion`, `boneCategory`, `termType`, `parentBoneCatalogId` | `BoneCatalog[]` (unpaginated) |
| GET | `/bone-catalog/:id` | — | `BoneCatalog` |
| POST/PATCH/DELETE | `…[/:id]` | — | `BoneCatalog` / `void` ([ADMIN]) |

The legacy path `/api/bone-names` no longer exists; a dead public-web service
still points at it (see [03-domain-model.md](./03-domain-model.md) and
[frontend/07-roadmap.md](./frontend/07-roadmap.md)).

## Dating Techniques — `/api/dating-techniques`  ·  public read, [ADMIN] writes

| Method | Path | Query params | Response |
|---|---|---|---|
| GET | `/dating-techniques` | `q` (optional) | `DatingTechnique[]` (unpaginated) |
| GET | `/dating-techniques/:id` | — | `DatingTechnique` |
| POST/PATCH/DELETE | `…[/:id]` | — | `DatingTechnique` / `void` ([ADMIN]) |

`GET` is **public** per the `SecurityConfig` allowlist
(`GET /api/dating-techniques/**`), although today only the backoffice consumes
it. Writes require `ROLE_ADMIN`.

## Users — `/api/users`

| Endpoint | Auth | Notes |
| --- | --- | --- |
| `GET /api/users/me` | authenticated | current user profile |
| `POST /api/users/me/password` | authenticated | `PasswordChangeRequest`; `204` |
| `GET /api/users` | `ROLE_ADMIN` | optional `q` filter |
| `GET /api/users/{id}` | `ROLE_ADMIN` | |
| `POST /api/users` | `ROLE_ADMIN` | `UserCreateRequest` |
| `PATCH /api/users/{id}` | `ROLE_ADMIN` | `UserUpdateRequest` |
| `DELETE /api/users/{id}` | `ROLE_ADMIN` | |

`/me` and `/me/password` also enforce `@PreAuthorize("isAuthenticated()")` as
defense in depth. Self-delete / self-disable / self-role-change are rejected
server-side (see
[backend/02-security-and-auth.md](./backend/02-security-and-auth.md)). Field
shapes live in `projects/backoffice/src/app/features/users/model/*.ts`.

## Dataset download — `/api/dataset/download`  ·  public (no JWT)

Returns a ZIP (`application/zip`, `paleohumans-dataset.zip`) of UTF-8 CSV files.
Entries map one-to-one to domain tables (`sites.csv`, `cultures.csv`,
`culture_features.csv`, `archaeological_contexts.csv`, `individuals.csv`,
`bone_catalog.csv`, `bones.csv` (no `individual_id` column),
`bone_individuals.csv` bridge, `skeletons.csv`, `dating_techniques.csv`,
`dated_samples.csv`, `dating_results.csv`, `references.csv`,
`site_references.csv`, `archaeological_context_references.csv`,
`funerary_contexts.csv`, `funerary_context_individuals.csv`). Excluded:
`app_user`, `refresh_token`, the `bone_catalog_component` view, and all v1
tables.

Operational: response cached `app.dataset.export.cache-ttl-seconds` (default
`300`), capped at `app.dataset.export.max-bytes` (default 20 MiB), streamed via
`StreamingResponseBody`, anonymous callers rate-limited, and every download
logged as a `dataset_download` audit event. Frontend usage:
`DatasetService.downloadDataset()` requests with `responseType: 'blob'`, reads
the filename from `Content-Disposition`, and triggers a browser download
(browser-only; never invoked from SSR).

## Admin lightweight projections — `/api/admin/*`  ·  [ADMIN]

The backoffice uses these to avoid hydrating full entity graphs.

| Method | Path | Notes |
|---|---|---|
| GET | `/admin/stats/dashboard` | `AdminDashboardStats` — scalar counts for `sites`, `archaeologicalContexts`, `individuals`, `bones`, `boneCatalog`, `skeletons`, `references`, `cultures`, `funeraryContexts`, `datingTechniques`, `datedSamples`, `datingResults`, `users`. |
| GET | `/admin/sites` | Paginated `AdminSiteListResponse` (`archaeologicalContextCount`, `referenceCount`). Filters `q`, `page`, `size`, `sort`. |
| GET | `/admin/individuals` | Paginated `AdminIndividualListResponse` (`individualType`, `mni`, `mniStatistical`, `archaeologicalContextId`, `boneCount`, `skeletonCount`, `funeraryContextCount`). Filters `q`, `sex`, `ageClassMain`, `individualType`, `archaeologicalContextId`, `page`, `size`, `sort`. |
| GET | `/admin/bones` | Paginated `AdminBoneListResponse` (`individualCount` — distinct associated individuals — plus `specimenName`, `repository`). The `individualId` filter matches through `bone_individual`. Filters `q`, `boneCatalogId`, `individualId`, `page`, `size`, `sort`. |

### Global admin search — `GET /api/admin/search`  ·  [ADMIN]

Powers the backoffice top-bar search via one bounded, lightweight projection
query per searched entity, returning a single flat ranked list.

- `q` — search text, trimmed, **minimum 2 characters**; blank/too-short returns
  `200` with empty `results` (case-insensitive `LIKE %q%`).
- `limit` — optional max total results, default `20`, clamped `[1, 50]`.
- `types` — optional comma-separated `AdminSearchResultType` names (e.g.
  `SITE,REFERENCE`); unknown type → `400`.

Each result is flat: `type`, `id`, `label`, `secondaryLabel` (nullable),
`route`. Searched types and routes: `SITE`, `INDIVIDUAL`,
`ARCHAEOLOGICAL_CONTEXT`, `REFERENCE`, `CULTURE`, `BONE_CATALOG`. Ranking is
deterministic: label-match quality, then entity priority, then label, then id.
Optional domains (Bones, Skeletons, Funerary Contexts, Dating records, Dating
techniques, Users) and enum-facet matching are **not** searched in this version
(tracked in [08-roadmap.md](./08-roadmap.md)).

## DTO conventions

- `*CreateRequest` — Java `record` with Bean Validation.
- `*UpdateRequest` — mutable class with `JsonNullable<T>` fields (absent =
  unchanged, `null` = clear where allowed, value = set).
- `*Response` — `record` with stable JSON keys, using `common/summary/*` records
  to keep nested entities one level deep.
- `*SummaryResponse` — flat one-level views used inside response records.

Summary DTOs: `SiteSummaryResponse`, `ArchaeologicalContextSummaryResponse`,
`IndividualSummaryResponse`, `BoneSummaryResponse`, `SkeletonSummaryResponse`
(in `common/summary/`), plus per-package `FuneraryContextSummaryResponse`,
`DatedSampleSummaryResponse`, `CultureEmbedResponse`,
`DatingTechniqueEmbedResponse`.

## Known absent filters and caveats

Things the frontend would benefit from but that **do not exist** in the current
contract:

- `country` filter on `/api/individuals`.
- `cultureId` filter on `/api/individuals`.
- Calibrated BP range field on any dating DTO (`calBpMin` / `calBpMax`).
- Aggregate count fields on `SiteResponse`.
- `GET /api/individuals/{id}/bundle` or an `?expand=` projection.
- No multi-value filters (each `country`/`siteId`/`cultureId`/
  `archaeologicalContextId`/`individualId` is single-value) and no `OR` across
  filters (everything is `AND`).

See [08-roadmap.md](./08-roadmap.md) for the active API wishlist.

## Security and CORS (summary)

- **JWT:** the custom `role` claim is used directly as a single
  `GrantedAuthority` (the `ROLE_` prefix is applied at issuance).
- **Rate limiting:** `/auth/login`, `/auth/refresh` (per IP / username) and the
  anonymous public endpoints `/bone-site-search`, `/dataset/download`.
- **CORS:** allows `GET, POST, PATCH, DELETE, OPTIONS` (**not `PUT`**); dev
  allows `http://localhost:*` / `http://127.0.0.1:*`, production uses the
  configured origin allowlist.

Full detail in
[backend/02-security-and-auth.md](./backend/02-security-and-auth.md);
caching/compression in
[backend/05-performance-and-pagination.md](./backend/05-performance-and-pagination.md).

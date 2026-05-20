# 05 — API Contract

This document is the catalogue of HTTP endpoints exposed by the
backend. Security details (public vs admin, JWT, rate limits) live in
[06-security-and-auth.md](./06-security-and-auth.md). Validation rules
live in [07-validation-and-errors.md](./07-validation-and-errors.md).

## Base path

`WebConfig.configurePathMatch` prefixes every controller mapping with
`/api`. A controller annotated `@RequestMapping("/sites")` is reachable
at `/api/sites`.

## Controllers

| Controller | URL prefix |
| --- | --- |
| `AuthController` | `/api/auth` |
| `SiteController` | `/api/sites` |
| `AdminSiteController` | `/api/admin/sites` |
| `CultureController` | `/api/cultures` |
| `ArchaeologicalContextController` | `/api/archaeological-contexts` |
| `IndividualController` | `/api/individuals` |
| `AdminIndividualController` | `/api/admin/individuals` |
| `BoneCatalogController` | `/api/bone-catalog` |
| `BoneController` | `/api/bones` |
| `AdminBoneController` | `/api/admin/bones` |
| `SkeletonController` | `/api/skeletons` |
| `DatedSampleController` | `/api/dated-samples` |
| `DatingResultController` | `/api/dating-results` |
| `DatingTechniqueController` | `/api/dating-techniques` |
| `ReferenceController` | `/api/references` |
| `FuneraryContextController` | `/api/funerary-contexts` |
| `StatsController` | `/api/stats` |
| `AdminDashboardStatsController` | `/api/admin/stats` |
| `BoneSiteSearchController` | `/api/bone-site-search` |
| `DatasetController` | `/api/dataset` |
| `UserController` | `/api/users` |

Removed v1 paths — do not document or reintroduce:
`/api/osteological-units`, `/api/specimens`, `/api/dates`,
`/api/burial-groups`. No controller serves these paths anymore.

> Open question (Unverified): `SecurityConfig` still declares
> `permitAll` entries for these legacy paths, and
> `CacheControlInterceptor` still has them in its `matches(...)`
> list. They are dead config because no controller responds to them,
> but the entries should be removed to avoid confusion. Tracked in
> [11-roadmap.md](./11-roadmap.md).

## Standard CRUD shape

Domain controllers expose:

- `GET /resource` — paginated list with filter and sort.
- `GET /resource/{id}` — single detail.
- `POST /resource` — create, returns 201 + body.
- `PATCH /resource/{id}` — partial update via `*UpdateRequest`
  (`JsonNullable<T>`); returns updated body.
- `DELETE /resource/{id}` — returns 204.

Listing endpoints use `PaginatedResponse<T>` (see "Pagination
envelope" below).

## Public collection filters

Filter parameters are validated as the declared type
(`@RequestParam` Jackson coercion). Unknown enum values produce a
`400`.

| Endpoint | Query parameters |
| --- | --- |
| `GET /api/sites` | `q`, `country`, `page`, `size`, `sort` |
| `GET /api/sites/countries` | none — returns `List<String>` of distinct countries |
| `GET /api/cultures` | `phase` — returns an unpaginated `List<CultureResponse>` (no `q`, no `page`/`size`/`sort`). |
| `GET /api/archaeological-contexts` | `q`, `siteId`, `cultureId`, `page`, `size`, `sort` |
| `GET /api/individuals` | `q`, `sex`, `sexCertain`, `ageClassMain`, `individualType`, `archaeologicalContextId`, `siteId`, `page`, `size`, `sort` |
| `GET /api/individuals/list` | same filters as `/api/individuals`; returns a lightweight `IndividualSummaryDto` page (only the columns the public list view renders) |
| `GET /api/bone-catalog` | `q`, `skeletonMain`, `skeletonRegion`, `boneCategory`, `termType`, `parentBoneCatalogId` — returns an unpaginated `List<BoneCatalogResponse>` (no `page`/`size`/`sort`). |
| `GET /api/bones` | `q`, `boneCatalogId`, `individualId`, `page`, `size`, `sort` |
| `GET /api/skeletons` | `q`, `individualId`, `skeletonCategory`, `page`, `size`, `sort` |
| `GET /api/dated-samples` | `sampleOrigin`, `datingType`, `archaeologicalContextId`, `boneId`, `skeletonId`, `page`, `size`, `sort` |
| `GET /api/dating-results` | `datedSampleId`, `datingTechniqueId`, `page`, `size`, `sort` |
| `GET /api/dating-techniques` | `q` — returns an unpaginated `List<DatingTechniqueResponse>` (no `page`/`size`/`sort`). |
| `GET /api/funerary-contexts` | `archaeologicalContextId`, `burialContext`, `page`, `size`, `sort` |
| `GET /api/references` | `q`, `page`, `size`, `sort` |

### Site listing

`GET /api/sites?country=` filters by exact country string.
`GET /api/sites/countries` returns the deduplicated list of countries
to populate dropdowns without paging the entire table.

### Individual listing

`GET /api/individuals` returns the full nested
`IndividualResponse` (with `archaeologicalContext`, `bones`,
`skeletons`, `funeraryContexts`). It is heavy.

`GET /api/individuals/list` returns the same filter surface but
yields `IndividualSummaryDto` — the projection consumed by the
public table view. It includes `ageAtDeathText` as a nullable field
from `individual.age_at_death_text`. Prefer this endpoint for list
views.

## Public stats endpoints

### `GET /api/stats/home`

Compact summary for the public Home page.

```json
{
  "totalSites": 248,
  "totalIndividuals": 804,
  "totalBoneRemains": 6604,
  "totalSkeletons": 105,
  "lastUpdatedAt": "2026-04-22T17:35:00Z"
}
```

| Field | Meaning |
| --- | --- |
| `totalSites` | `COUNT(*)` over `site`. |
| `totalIndividuals` | **`SUM(individual.mni_statistical)`** — curated minimum number of distinct individuals, not the row count of `individual`. |
| `totalBoneRemains` | `SUM(bone.bone_quantity_min)`. |
| `totalSkeletons` | `COUNT(*)` over `skeleton`. |
| `lastUpdatedAt` | Maximum `updated_at` across the public domain tables (site, culture, archaeological_context, individual, bone, skeleton, bibliographic_reference). Null when all relevant tables are empty. The public Home page hides the "Database last updated" line when null. |

The `totalIndividuals` semantics are intentional: a single
`Individual` row may represent `MIXED_INDIVIDUALS` or
`UNASSIGNED_REMAINS` with `mni > 1`. Summing `mni_statistical`
captures the curated minimum number of distinct individuals.

### `GET /api/stats/map-timeline`

Normalized read model for the public Map and Timeline pages.

```json
{
  "sites": [
    {
      "siteId": 1,
      "siteName": "Arene Candide",
      "country": "Italy",
      "region": "Liguria",
      "municipality": "Finale Ligure",
      "latitude": 44.17,
      "longitude": 8.34,
      "totalMni": 5,
      "individualCount": 3,
      "boneCount": 2,
      "skeletonCount": 1,
      "datedSampleCount": 1,
      "dominantCultureId": 100,
      "cultureIds": [100, 200]
    }
  ],
  "units": [
    {
      "archaeologicalContextId": 10,
      "siteId": 1,
      "cultureId": 100,
      "individualCount": 3,
      "mniTotal": 5,
      "boneCount": 2,
      "skeletonCount": 1,
      "datedSampleCount": 1
    }
  ],
  "cultures": [
    {
      "cultureId": 100,
      "cultureName": "Aurignacian",
      "phase": "EARLY_UPPER_PALAEOLITHIC",
      "startBp": 43000,
      "endBp": 33000,
      "color": "#d34d31",
      "colorRgb": "211,77,49"
    }
  ]
}
```

The list element type is named `units` for backward compatibility
with the frontend — it represents archaeological contexts in v2. The
`archaeologicalContextId` field on each row makes the canonical name
explicit.

Aggregation semantics:

| Field | Meaning |
| --- | --- |
| Site `totalMni` | sum of `mniTotal` over the site's archaeological contexts (itself `SUM(individual.mni_statistical)` per context). |
| Site `individualCount` | sum of distinct `Individual` rows per context. |
| Site `boneCount` | sum of `Bone` rows per context. |
| Site `skeletonCount` | sum of `Skeleton` rows per context. |
| Site `datedSampleCount` | distinct `DatedSample` rows linked through the context, its bones, or its skeletons. |
| Site `dominantCultureId` | culture id with the most archaeological contexts at the site. |
| Site `cultureIds` | culture ids represented at the site. |
| Unit `mniTotal` | `SUM(individual.mni_statistical)` for the context. |

## Bone-site search

`GET /api/bone-site-search` — JPQL constructor projection over
`Bone JOIN BoneCatalog JOIN Individual JOIN ArchaeologicalContext
JOIN Site LEFT JOIN Culture` plus `SIZE(b.datedSamples)` for the
per-bone dated-sample count.

Filters (all optional):

`q`, `boneCatalogId`, `skeletonMain`, `skeletonRegion`,
`boneCategory`, `laterality`, `toothType`, `toothVerticalPosition`,
`toothNumber`, `vertebraType`, `vertebraNumber`, `ribNumber`,
`phalanxType`, `phalanxNumber`, `handFootBoneSegment`, `country`,
`cultureId`, `datingAvailable`.

`datingAvailable=true` is implemented as `SIZE(b.datedSamples) > 0`.

Pagination: `page`, `size`, `sort`. Allowed sort fields:
`siteName`, `country`, `minimumBoneRemainCount`,
`matchingBoneRecordCount`, `individualCount`. Default sort:
`minimumBoneRemainCount,desc` then `siteName,asc`.

Anonymous callers (no `Authorization` header) pass through
`PublicEndpointRateLimiter` keyed on
`bone-site-search:<remote-addr>`. Repository-level
`setMaxResults(app.search.bone-site.max-match-rows)` caps the broad
match budget; controller-level page size is capped by
`app.search.bone-site.max-page-size`.

Per-site response example:

```json
{
  "siteId": 1,
  "siteName": "Arene Candide",
  "matchingBoneRecordCount": 2,
  "minimumBoneRemainCount": 5,
  "individualCount": 2,
  "archaeologicalContextCount": 2,
  "datedSampleCount": 2,
  "cultures": [...],
  "boneGroups": [...],
  "units": [
    {
      "archaeologicalContextId": 10,
      "individualId": 5,
      "individualType": "INDIVIDUAL",
      "mni": "1",
      "mniStatistical": 1,
      "stratigraphicContext": "Layer A",
      "cultureId": 100
    }
  ]
}
```

## Dataset download

`GET /api/dataset/download` — anonymous (no JWT). Returns a ZIP
archive (`application/zip`) named `paleohumans-dataset.zip` with
UTF-8 CSV files.

Entries:

| Entry | Source table |
| --- | --- |
| `sites.csv` | `site` |
| `cultures.csv` | `culture` |
| `culture_features.csv` | `culture_feature` |
| `archaeological_contexts.csv` | `archaeological_context` |
| `individuals.csv` | `individual` |
| `bone_catalog.csv` | `bone_catalog` |
| `bones.csv` | `bone` |
| `skeletons.csv` | `skeleton` |
| `dating_techniques.csv` | `dating_technique` |
| `dated_samples.csv` | `dated_sample` |
| `dating_results.csv` | `dating_result` |
| `references.csv` | `bibliographic_reference` |
| `site_references.csv` | `site_bibliographic_reference` |
| `archaeological_context_references.csv` | `archaeological_context_bibliographic_reference` |
| `funerary_contexts.csv` | `funerary_context` |
| `funerary_context_individuals.csv` | `funerary_context_individual` |

Excluded: `app_user`, `refresh_token`, `bone_catalog_component`
(view), and any v1 tables.

Operational constraints:

- ZIP body cached for `app.dataset.export.cache-ttl-seconds` (default
  `300`).
- Total bytes capped at `app.dataset.export.max-bytes` (default
  `20971520` = 20 MiB).
- Response is streamed via `StreamingResponseBody`.
- Anonymous callers pass through `PublicEndpointRateLimiter` keyed on
  `dataset:<remote-addr>`.
- `SecurityAuditService` logs a `dataset_download` event for both
  anonymous and authenticated downloads.

## Auth endpoints

| Endpoint | Method | Body | Notes |
| --- | --- | --- | --- |
| `/api/auth/login` | POST | `LoginRequest { username, password }` | Public. Returns `{ accessToken, refreshToken }`. Rate-limited per IP and per username. |
| `/api/auth/refresh` | POST | `RefreshRequest { refreshToken }` | Public. Rotates the refresh-token family. |
| `/api/auth/logout` | POST | `LogoutRequest { refreshToken }` | Public. Revokes the submitted family. Returns 204. |

See [06-security-and-auth.md](./06-security-and-auth.md) for the full
JWT and refresh-token contract.

## User endpoints

| Endpoint | Auth | Notes |
| --- | --- | --- |
| `GET /api/users/me` | authenticated | current user profile |
| `POST /api/users/me/password` | authenticated | `PasswordChangeRequest`; returns 204 |
| `GET /api/users` | `ROLE_ADMIN` | optional `q` filter |
| `GET /api/users/{id}` | `ROLE_ADMIN` | |
| `POST /api/users` | `ROLE_ADMIN` | `UserCreateRequest` |
| `PATCH /api/users/{id}` | `ROLE_ADMIN` | `UserUpdateRequest` |
| `DELETE /api/users/{id}` | `ROLE_ADMIN` | |

`/me` and `/me/password` also enforce `@PreAuthorize("isAuthenticated()")`
as defense in depth on top of `SecurityConfig`'s route rules.

## Admin endpoints

### `GET /api/admin/stats/dashboard`

```json
{
  "sites": 248,
  "archaeologicalContexts": 412,
  "individuals": 804,
  "bones": 6604,
  "boneCatalog": 205,
  "skeletons": 105,
  "references": 512,
  "cultures": 44,
  "funeraryContexts": 68,
  "datingTechniques": 7,
  "datedSamples": 180,
  "datingResults": 220,
  "users": 3
}
```

### Admin paginated lists

| Endpoint | Filters |
| --- | --- |
| `GET /api/admin/sites` | `q`, `page`, `size`, `sort` |
| `GET /api/admin/individuals` | `q`, `sex`, `ageClassMain`, `individualType`, `archaeologicalContextId`, `page`, `size`, `sort` |
| `GET /api/admin/bones` | `q`, `boneCatalogId`, `individualId`, `page`, `size`, `sort` |

These return `Admin*ListResponse` records purpose-built for the
backoffice table view:

- `AdminSiteListResponse` exposes `archaeologicalContextCount` (the
  v1 field name `osteologicalUnitCount` is gone).
- `AdminIndividualListResponse` exposes `individualType`, `mni`,
  `mniStatistical`, `archaeologicalContextId`, `boneCount`,
  `skeletonCount`, `funeraryContextCount`.
- `AdminBoneListResponse` exposes `individualId`, `specimenName`,
  `repository`.

## Pagination envelope

```json
{
  "content": [],
  "page": 0,
  "size": 20,
  "totalElements": 0,
  "totalPages": 0,
  "first": true,
  "last": true,
  "hasNext": false,
  "hasPrevious": false,
  "sort": []
}
```

Defaults and limits (see `common/util/PaginationUtils.java`):

- `page` zero-based, default `0`, must be `>= 0`.
- `size` default `20`, max `100`.
- `sort` accepted as repeated `field` or `field,direction`. Allowed
  fields are declared per service; unknown fields return `400`.

The bone-site search has its own per-page cap from
`app.search.bone-site.max-page-size`.

## DTO conventions

- `*CreateRequest` — Java `record`. Bean Validation enforces
  required fields and per-field rules. See
  [07-validation-and-errors.md](./07-validation-and-errors.md).
- `*UpdateRequest` — mutable class with `JsonNullable<T>` fields.
  Field omitted means "do not change". Field present with `null`
  means "clear", when the column allows it. Field present with a
  value means "set".
- `*Response` — `record`. Stable JSON keys. Uses
  `common/summary/*` records to keep nested entities one level
  deep.
- `*SummaryResponse` — flat one-level views used inside response
  records.

Summary DTOs in `common/summary/`:

- `SiteSummaryResponse`
- `ArchaeologicalContextSummaryResponse`
- `IndividualSummaryResponse`
- `BoneSummaryResponse`
- `SkeletonSummaryResponse`
- `SummaryMappers` (helper)

Per-package summaries:

- `funerary_context/dto/FuneraryContextSummaryResponse`
- `dated_sample/dto/DatedSampleSummaryResponse`
- `culture/dto/CultureEmbedResponse`
- `dating_technique/dto/DatingTechniqueEmbedResponse`

## Known absent filters and caveats

- No multi-value filters for the `country`, `siteId`, `cultureId`,
  `archaeologicalContextId`, `individualId` parameters — each is
  single-value.
- No `OR` operator across filters; everything is `AND`.
- Bone-site search `datingAvailable` returns the boolean filter on
  bones only — it does not check context-level dated samples.
- The public read endpoints have no projection-by-fields parameter
  (the `/api/individuals/list` projection is a separate endpoint).
- No OpenAPI document is generated. The catalogue here plus the
  controller signatures is the source of truth.

# 04 — Backend API Contract

This document is the **frontend-facing** view of the Spring Boot REST API. It lists every endpoint the two Angular apps consume, with the query parameters they actually pass, plus important caveats.

> **Source authority.** Endpoints, response shapes, and filters here are derived from the active frontend services in `projects/web/src/app/features/*` and `projects/backoffice/src/app/features/*`. The Spring Boot repository is **not present in this workspace**; behavior that depends on backend internals (security config, exact CORS allowlist, calibration logic, rate-limit window) is marked `Requires code verification`.

## Conventions

- **Base URL:** `http://127.0.0.1:8080/api` in dev (per `environment.ts` in each app). Override in `environment.prod.ts` for deployment.
- **Auth:** stateless JWT bearer. Public endpoints accept no token. Admin endpoints require a token with `ROLE_ADMIN`.
- **Pagination envelope:**

  ```ts
  interface PaginatedResponse<T> {
    content: T[];
    page: number;
    size: number;
    totalElements: number;
    totalPages: number;
    first: boolean;
    last: boolean;
    hasNext: boolean;
    hasPrevious: boolean;
    sort: string[];
  }
  ```

  Sort uses repeated Spring-style `sort=field,direction` query parameters. Multiple `sort` values are accumulated via `params.append('sort', value)`.
- **PATCH semantics (`JsonNullable`):**
  - Key absent → field unchanged.
  - Key = `null` → field cleared.
  - Key = value → field updated.
  Frontend always strips `undefined` keys via a per-service `cleanPatch()` helper before sending.
- **Caller column** — `[PUBLIC]` = called by `projects/web`; `[ADMIN]` = called by `projects/backoffice`; `[BOTH]` = both.

---

## Auth — `/api/auth`

| Method | Path | Body | Response |
|---|---|---|---|
| POST | `/auth/login` | `{ username, password }` | `{ accessToken, refreshToken }` |
| POST | `/auth/refresh` | `{ refreshToken }` | `{ accessToken, refreshToken }` |
| POST | `/auth/logout` | `{ refreshToken }` | `204 No Content` |

- The token response carries `accessToken` and `refreshToken` only. No expiry timestamps in the body.
- Refresh tokens rotate on use (old revoked, new issued).
- Rate limiting is reported on `/auth/login` and `/auth/refresh` (sliding window, IP-based). Exact window and quota live in backend config. `Requires code verification`.

---

## Stats — `/api/stats`

| Method | Path | Query params | Response | Called from |
|---|---|---|---|---|
| GET | `/stats/home` | — | `HomeStats` | [PUBLIC] Home |
| GET | `/stats/map-timeline` | — | `MapTimelineStats` | [PUBLIC] Map, Timeline |

`HomeStats` (`projects/web/src/app/features/stats/model/HomeStats.ts`):

```ts
interface HomeStats {
  totalSites: number;
  totalIndividuals: number;
  totalBoneRemains: number;
  totalSkeletons: number;
  lastUpdatedAt?: string | null;   // optional, ISO-8601; hidden when absent
}
```

The frontend renders `lastUpdatedAt` only when the backend supplies it. As of the last verified probe the field is not populated; the Home page omits the "Database last updated" line in that case rather than fabricating a value. `Requires code verification` for the current backend status.

`MapTimelineStats` (`projects/web/src/app/features/stats/model/MapTimelineStats.ts`):

```ts
interface MapTimelineStats {
  sites:    MapTimelineSite[];
  units:    MapTimelineUnit[];        // keyed by archaeologicalContextId in v2
  cultures: MapTimelineCulture[];
}

interface MapTimelineSite {
  siteId; siteName; country; region; municipality;
  latitude; longitude;
  totalMni; individualCount; boneCount; skeletonCount; datedSampleCount;
  dominantCultureId; cultureIds;
}

interface MapTimelineUnit {
  archaeologicalContextId; siteId; cultureId;
  individualCount; mniTotal; boneCount; skeletonCount; datedSampleCount;
}

interface MapTimelineCulture {
  cultureId; cultureName; phase; startBp; endBp; color; colorRgb;
}
```

The Map page and Timeline page **only** use this endpoint for the core visualization data. They do not load unfiltered `/api/archaeological-contexts`.

---

## Sites — `/api/sites`

| Method | Path | Query params | Response | Called from |
|---|---|---|---|---|
| GET | `/sites` | `q` (String), `country` (String), `page`, `size`, repeated `sort` | `PaginatedResponse<Site>` | [BOTH] |
| GET | `/sites/countries` | — | `string[]` | [PUBLIC] Sites |
| GET | `/sites/:id` | — | `Site` | [BOTH] |
| POST | `/sites` | — | `Site` | [ADMIN] |
| PATCH | `/sites/:id` | — | `Site` | [ADMIN] |
| DELETE | `/sites/:id` | — | `void` | [ADMIN] |

Default sort: `siteName,asc`.

`SiteResponse`: `siteId`, `siteName`, `country`, `region`, `municipality`, `latitude`, `longitude`, `updatedAt`.

`SiteResponse` **does not contain aggregate counts** (individuals, contexts, dated samples). Counts come from `/api/stats/map-timeline` or `/api/bone-site-search`. Earlier docs that referenced an `individualCount` field on `SiteResponse` were wrong.

---

## Archaeological Contexts — `/api/archaeological-contexts`

| Method | Path | Query params | Response | Called from |
|---|---|---|---|---|
| GET | `/archaeological-contexts` | `q`, `siteId`, `cultureId`, `page`, `size`, repeated `sort` | `PaginatedResponse<ArchaeologicalContext>` | [BOTH] |
| GET | `/archaeological-contexts/:id` | — | `ArchaeologicalContext` | [BOTH] |
| POST | `/archaeological-contexts` | — | `ArchaeologicalContext` | [ADMIN] |
| PATCH | `/archaeological-contexts/:id` | — | `ArchaeologicalContext` | [ADMIN] |
| DELETE | `/archaeological-contexts/:id` | — | `void` | [ADMIN] |

Default sort: `archaeologicalContextId,asc`.

Response embeds: `site`, `culture`, `individuals[]`, `funeraryContexts[]`, `references[]`, optionally `datedSamples[]`. See [03-domain-model.md](./03-domain-model.md).

Create/update payload accepts: `siteId`, `stratigraphicContext`, `cultureId`, `referenceIds`.

---

## Individuals — `/api/individuals`

| Method | Path | Query params | Response | Called from |
|---|---|---|---|---|
| GET | `/individuals` | `q`, `sex`, `sexCertain`, `ageClassMain`, `individualType`, `archaeologicalContextId`, `siteId`, `page`, `size`, repeated `sort` | `PaginatedResponse<Individual>` | [BOTH] |
| GET | `/individuals/list` | same as above | `PaginatedResponse<IndividualSummary>` | [PUBLIC] Individuals list |
| GET | `/individuals/:id` | — | `Individual` | [BOTH] |
| POST | `/individuals` | — | `Individual` | [ADMIN] |
| PATCH | `/individuals/:id` | — | `Individual` | [ADMIN] |
| DELETE | `/individuals/:id` | — | `void` | [ADMIN] |

Default sort: `individualId,asc`.

`/individuals` returns the full `Individual` payload (with `archaeologicalContext`, `bones`, `skeletons`, `funeraryContexts`, `datedSamples`). The public list page uses the lightweight `/individuals/list` projection that returns only the columns the table renders.

`sex` is a strict enum (`MALE`, `FEMALE`, `INDETERMINATE`). `sexCertain` is a Boolean filter. Combining them powers the UI options `Male`, `Male?`, `Female`, `Female?`, `Indeterminate`, `Indeterminate?`.

There is **no `cultureId` or `country` filter** on `/individuals`. The current public list shows culture/country for each row from the embedded archaeological context, but cannot filter the global list by culture or country without a backend change.

---

## Bones — `/api/bones`

| Method | Path | Query params | Response | Called from |
|---|---|---|---|---|
| GET | `/bones` | `q`, `boneCatalogId`, `individualId`, `page`, `size`, repeated `sort` | `PaginatedResponse<Bone>` _or_ `Bone[]` _(see note)_ | [PUBLIC] |
| GET | `/bones/:id` | — | `Bone` | [BOTH] |
| POST | `/bones` | — | `Bone` | [ADMIN] |
| PATCH | `/bones/:id` | — | `Bone` | [ADMIN] |
| DELETE | `/bones/:id` | — | `void` | [ADMIN] |

The public web has two helpers:

- `BoneService.search({ q, boneCatalogId, individualId, page, size, sort })` returns `PaginatedResponse<Bone>`.
- `BoneService.findByIndividual(individualId)` issues the same `GET /bones?individualId=X` but **types the response as `Bone[]`** because the public `/bones` endpoint historically returns a plain array shape in this mode. The individual detail page uses this helper for the bone inventory.

> The response shape difference is a known backend quirk. Always call `findByIndividual()` when you need every bone for an individual; never try to read `.content` off a plain array. `Requires code verification` for the canonical contract of `/bones` (paginated vs array) after the next backend pass.

Required create fields: `individualId`, `boneCatalogId`, `boneSource` (non-blank), `boneQuantityMin` (positive).

---

## Bone Site Search — `/api/bone-site-search`

| Method | Path | Query params | Response | Called from |
|---|---|---|---|---|
| GET | `/bone-site-search` | see below | `BoneSiteSearchResponse` | [PUBLIC] Bones |

Supported query params:

```
q, boneCatalogId, skeletonMain, skeletonRegion, boneCategory, laterality,
toothType, toothVerticalPosition, toothNumber,
vertebraType, vertebraNumber, ribNumber,
phalanxType, phalanxNumber, handFootBoneSegment,
country, cultureId, datingAvailable,
page, size, repeated sort
```

Default sort: `minimumBoneRemainCount,desc` then `siteName,asc`.

Top-level envelope: `summary`, `filters`, `content[]` (sites), plus the standard pagination fields.

Important v2 renames inside the response (compared to old docs):

- `osteologicalUnitCount` → `archaeologicalContextCount`
- `datedSpecimenCount` → `datedSampleCount`

This endpoint is the **only** data source the `/bones` page uses. The page must never reconstruct the view by chaining `/api/archaeological-contexts` or `/api/bones`.

---

## Skeletons — `/api/skeletons`

| Method | Path | Query params | Response | Called from |
|---|---|---|---|---|
| GET | `/skeletons` | `q`, `individualId`, `skeletonCategory`, `page`, `size`, repeated `sort` | `PaginatedResponse<Skeleton>` | [BOTH] |
| GET | `/skeletons/:id` | — | `Skeleton` | [BOTH] |
| POST | `/skeletons` | — | `Skeleton` | [ADMIN] |
| PATCH | `/skeletons/:id` | — | `Skeleton` | [ADMIN] |
| DELETE | `/skeletons/:id` | — | `void` | [ADMIN] |

Required create fields: `individualId`, `skeletonCategory`.

---

## Funerary Contexts — `/api/funerary-contexts`

| Method | Path | Query params | Response | Called from |
|---|---|---|---|---|
| GET | `/funerary-contexts` | `archaeologicalContextId`, `burialContext`, `individualId`, `page`, `size`, repeated `sort` | `PaginatedResponse<FuneraryContext>` | [BOTH] |
| GET | `/funerary-contexts/:id` | — | `FuneraryContext` | [BOTH] |
| POST | `/funerary-contexts` | — | `FuneraryContext` | [ADMIN] |
| PATCH | `/funerary-contexts/:id` | — | `FuneraryContext` | [ADMIN] |
| DELETE | `/funerary-contexts/:id` | — | `void` | [ADMIN] |

Create/update payloads accept `individualIds` as a `Set<number>` for M2M assignment.

---

## Dated Samples — `/api/dated-samples`

| Method | Path | Query params | Response | Called from |
|---|---|---|---|---|
| GET | `/dated-samples` | `sampleOrigin`, `datingType`, `archaeologicalContextId`, `boneId`, `skeletonId`, `page`, `size`, repeated `sort` | `PaginatedResponse<DatedSample>` | [BOTH] |
| GET | `/dated-samples/:id` | — | `DatedSample` | [BOTH] |
| POST | `/dated-samples` | — | `DatedSample` | [ADMIN] |
| PATCH | `/dated-samples/:id` | — | `DatedSample` | [ADMIN] |
| DELETE | `/dated-samples/:id` | — | `void` | [ADMIN] |

A `DatedSample` links to exactly one of `archaeologicalContext`, `bone`, or `skeleton` (mutually exclusive at creation).

Each `DatedSample` response embeds the full `datingResults[]` payload. The public Individual detail page consumes this directly and does **not** issue a separate `/dating-results` call.

---

## Dating Results — `/api/dating-results`

| Method | Path | Query params | Response | Called from |
|---|---|---|---|---|
| GET | `/dating-results` | `datedSampleId`, `datingTechniqueId`, `page`, `size`, repeated `sort` | `PaginatedResponse<DatingResult>` | [ADMIN] |
| GET | `/dating-results/:id` | — | `DatingResult` | [ADMIN] |
| POST | `/dating-results` | — | `DatingResult` | [ADMIN] |
| PATCH | `/dating-results/:id` | — | `DatingResult` | [ADMIN] |
| DELETE | `/dating-results/:id` | — | `void` | [ADMIN] |

`DatingResult` carries `datedSampleId`, `datingTechnique{datingTechniqueId, datingTechniqueName}`, `datesBpUncal`, `datesRange`, `notes`. The backoffice has a composite `DatingRecordService` that creates a `DatedSample` + `DatingResult` in one user-facing flow.

**No calibrated BP range field exists.** Anywhere that "calibrated range" was referenced is incorrect.

---

## Cultures — `/api/cultures`

| Method | Path | Query params | Response | Called from |
|---|---|---|---|---|
| GET | `/cultures` | `phase` (optional) | `Culture[]` | [BOTH] |
| GET | `/cultures/:id` | — | `Culture` | [BOTH] |
| POST | `/cultures` | — | `Culture` | [ADMIN] |
| PATCH | `/cultures/:id` | — | `Culture` | [ADMIN] |
| DELETE | `/cultures/:id` | — | `void` | [ADMIN] |

`CultureService` in the public web caches `getAll()` with `shareReplay({ bufferSize: 1, refCount: false })`.

The full `Culture` response has `region`, `description`, `features[]`; the embedded form (`CultureEmbedResponse`) omits them.

---

## References — `/api/references`

| Method | Path | Query params | Response | Called from |
|---|---|---|---|---|
| GET | `/references` | `q`, `page`, `size`, repeated `sort` | `PaginatedResponse<Reference>` | [BOTH] |
| GET | `/references/:id` | — | `Reference` | [BOTH] |
| POST | `/references` | — | `Reference` | [ADMIN] |
| PATCH | `/references/:id` | — | `Reference` | [ADMIN] |
| DELETE | `/references/:id` | — | `void` | [ADMIN] |

Default sort: `year,desc` then `bibliographicReferenceId,asc`.

The public `/bibliography` page consumes the paginated envelope. Do not type the response as a bare array.

---

## Bone Catalog — `/api/bone-catalog`

| Method | Path | Query params | Response | Called from |
|---|---|---|---|---|
| GET | `/bone-catalog` | `q`, `skeletonMain`, `skeletonRegion`, `boneCategory`, `termType`, `parentBoneCatalogId` | `BoneCatalog[]` | [BOTH] |
| GET | `/bone-catalog/:id` | — | `BoneCatalog` | [BOTH] |
| POST | `/bone-catalog` | — | `BoneCatalog` | [ADMIN] |
| PATCH | `/bone-catalog/:id` | — | `BoneCatalog` | [ADMIN] |
| DELETE | `/bone-catalog/:id` | — | `void` | [ADMIN] |

The legacy path `/api/bone-names` no longer exists on the backend. A dead public-web service still points at it (see [03-domain-model.md](./03-domain-model.md) and [10-roadmap.md](./10-roadmap.md)).

---

## Dating Techniques — `/api/dating-techniques`

| Method | Path | Query params | Response | Called from |
|---|---|---|---|---|
| GET | `/dating-techniques` | `q` (optional) | `DatingTechnique[]` | [ADMIN] |
| GET | `/dating-techniques/:id` | — | `DatingTechnique` | [ADMIN] |
| POST | `/dating-techniques` | — | `DatingTechnique` | [ADMIN] |
| PATCH | `/dating-techniques/:id` | — | `DatingTechnique` | [ADMIN] |
| DELETE | `/dating-techniques/:id` | — | `void` | [ADMIN] |

Public access status post-v2: `Requires code verification` against the current Spring `SecurityConfig`.

---

## Admin lightweight projections — `/api/admin/*`

The backoffice uses these endpoints for list pages and the dashboard to avoid hydrating full entity graphs. All require `ROLE_ADMIN`.

| Method | Path | Notes |
|---|---|---|
| GET | `/admin/stats/dashboard` | Returns `AdminDashboardStats` (scalar counts). |
| GET | `/admin/sites` | Paginated `AdminSiteListResponse` (with `archaeologicalContextCount`, `referenceCount`). |
| GET | `/admin/individuals` | Paginated `AdminIndividualListResponse`. |
| GET | `/admin/bones` | Paginated `AdminBoneListResponse`. |

`AdminDashboardStats` (`projects/backoffice/src/app/features/admin-stats/model/AdminDashboardStats.ts`):

```ts
{
  sites, archaeologicalContexts, individuals,
  bones, boneCatalog, skeletons, references, cultures,
  funeraryContexts, datingTechniques, datedSamples, datingResults,
  users
}
```

---

## Users — `/api/users`

Backoffice only. Admin CRUD over admin/user accounts, plus a self-service `/users/me` endpoint for the signed-in admin. Available on the backend since the 2026-04-19 rebuild.

Exact field shapes live in `projects/backoffice/src/app/features/users/model/*.ts`. `Requires code verification` for current public/admin classification of every method.

---

## Dataset download — `/api/dataset`

| Method | Path | Response |
|---|---|---|
| GET | `/dataset/download` | `application/zip` |

Returns a ZIP archive of UTF-8 CSV files containing the full structured dataset. No JWT required.

Frontend usage: `DatasetService.downloadDataset()` requests the file with `responseType: 'blob'` + `observe: 'response'`, reads the filename from `Content-Disposition` (fallback `paleohumans-dataset.zip`), and triggers a browser download via a temporary object URL. Browser-only; never invoked from SSR.

---

## Known absent filters and caveats

Things the frontend would benefit from but **does not exist** in the current backend contract:

- `country` filter on `/api/individuals`.
- `cultureId` filter on `/api/individuals`.
- Calibrated BP range field on any dating DTO (`calBpMin` / `calBpMax`).
- Aggregate count fields on `SiteResponse`.
- `GET /api/individuals/{id}/bundle` or `?expand=` projection that returns the individual with bones, skeletons, dated samples, and context in a single round-trip.

See [10-roadmap.md](./10-roadmap.md) for the active backend wishlist.

---

## Security and CORS

- **JWT:** custom `role` claim is used directly as a `GrantedAuthority`. The `ROLE_` prefix is applied at token issuance.
- **Rate limiting:** applied to `/auth/login` and `/auth/refresh`. Exact algorithm and quota live in backend config. `Requires code verification`.
- **CORS allowlist:** at the time of the v1 documentation only `http://localhost:*` was whitelisted; the production origin was commented out. Current state needs `Requires code verification` against the live backend.

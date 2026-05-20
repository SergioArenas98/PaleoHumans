# 09 — Performance and Pagination

## Pagination contract

Common pagination parameters across paginated `GET` endpoints:

- `page` — zero-based; default `0`; must be `>= 0`.
- `size` — default `20`; hard maximum `100`
  (`common/util/PaginationUtils.MAX_PAGE_SIZE`).
- `sort` — one or more `field` or `field,direction` parameters.
  Allowed sort fields are declared per service; an unknown field
  returns HTTP `400`.

Helper: `common/util/PaginationUtils.createPageable(...)` parses the
inputs, validates the sort allow-list, and returns a `Pageable`.

### Envelope

Returned by every paginated endpoint
(`common/dto/PaginatedResponse`):

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
  "sort": ["field,direction"]
}
```

### Default sorts

| Service | Default sort |
| --- | --- |
| `IndividualService` | `individualId,asc` |
| `DatedSampleService` | `datedSampleId,asc` |
| `BoneSiteSearchService` | `minimumBoneRemainCount,desc` then `siteName,asc` |
| Others | `id,asc` *(Requires code verification per service)* |

### Per-endpoint size caps

The bone-site search clamps page size to
`app.search.bone-site.max-page-size` (default `50` in `application.yml`).
All other paginated endpoints share the global `MAX_PAGE_SIZE = 100`.

## Heavy endpoints

These responses can be large and should be used carefully by clients.

### `GET /api/individuals`

Each row is a full nested `IndividualResponse` with
`archaeologicalContext` (including site, culture, culture features),
`bones` (bone catalog summary, curatorial fields, quantity),
`skeletons`, and `funeraryContexts`. The payload is dominated by the
nested objects, not by the column count of `individual`.

For list views that only need a few columns per row, the public web
should call `GET /api/individuals/list` instead. That endpoint
returns `IndividualSummaryDto` directly from a JPQL constructor
projection — entities are not hydrated.

### `GET /api/stats/map-timeline`

Aggregates site, archaeological-context, and culture rows for the
public Map and Timeline pages. Cached via
`Cache-Control: public, max-age=300, stale-while-revalidate=86400`
(see below). The frontend service also shares the result through
`shareReplay`.

### `GET /api/bone-site-search`

Single JPQL constructor projection plus in-memory aggregation. The
broad-match budget is capped at `app.search.bone-site.max-match-rows`
(default `5000`) before aggregation. Anonymous callers are
rate-limited.

### `GET /api/dataset/download`

A full ZIP of CSVs. Streamed with `StreamingResponseBody`, cached in
memory for `app.dataset.export.cache-ttl-seconds` (default `300`),
and capped at `app.dataset.export.max-bytes` (default `20 MiB`).
Anonymous callers are rate-limited.

## Lightweight projection endpoints

- `GET /api/individuals/list` — `IndividualSummaryDto` projection,
  used by the public individuals table.
- `GET /api/sites/countries` — `List<String>` of distinct countries,
  used to populate the country dropdown without paging the entire
  `site` table.
- `GET /api/stats/home` — single-pass aggregates with
  `lastUpdatedAt`.

## N+1 controls

Repositories use `@EntityGraph` to load nested data in one
follow-up query for detail endpoints:

| Repository | Detail entity graph |
| --- | --- |
| `IndividualRepository.findDetailById` / `findDetailsByIndividualIdIn` | `archaeologicalContext`, `archaeologicalContext.site`, `archaeologicalContext.culture`, `archaeologicalContext.culture.features`, `bones`, `bones.boneCatalog`, `skeletons`, `funeraryContexts`, `funeraryContexts.individuals` |
| `ArchaeologicalContextRepository.findDetailById` | `site`, `culture`, `culture.features`, `references` |
| `DatedSampleRepository.findDetailById` | `archaeologicalContext`, `archaeologicalContext.site`, `archaeologicalContext.culture`, `bone`, `bone.boneCatalog`, `bone.individual`, `skeleton`, `skeleton.individual`, `datingResults`, `datingResults.datingTechnique` |

The wide entity graphs are fine for current data volume but should
be revisited if rows-per-table grow significantly. Inspect actual
SQL counts before optimizing.

`spring.jpa.open-in-view=false` is set globally
(`application.properties`). DTO mapping must happen **inside** the
`@Transactional` service method, not while serializing the response.

## Aggregate / projection queries

Used for stats and search to avoid hydrating entities:

- `IndividualRepository.sumMniStatistical()` — single `SUM` query.
- `BoneRepository.sumBoneQuantityMin()` — single `SUM` query.
- `SiteRepository.findMapTimelineSiteRows()` — JPQL constructor
  projection.
- `ArchaeologicalContextRepository.findMapTimelineUnitRows()` —
  JPQL constructor projection with correlated subqueries for per-
  context counts (individuals, MNI total, bones, skeletons, dated
  samples).
- `CultureRepository.findMapTimelineCultureRows()` — JPQL
  constructor projection.
- `BoneSiteSearchRepository.findMatches(...)` — single JPQL
  constructor projection with `SIZE(b.datedSamples)`.

## Response compression

Enabled in `application.properties`:

```properties
server.compression.enabled=true
server.compression.mime-types=application/json,application/xml,text/plain,text/xml
server.compression.min-response-size=1024
```

Spring Boot negotiates gzip with the client (`Accept-Encoding: gzip,
br`); brotli requires a separate filter or an edge layer. JSON
payloads over 1 KiB are compressed before transmission.

## HTTP cache headers

`CacheControlInterceptor` (`config/CacheControlInterceptor.java`)
sets `Cache-Control` on `GET` responses for the public resources.
This takes precedence over Spring Security's default `no-store`
header.

| Path pattern | `Cache-Control` |
| --- | --- |
| `/api/stats/*` | `public, max-age=300, stale-while-revalidate=86400` |
| `/api/cultures`, `/api/cultures/...` | `public, max-age=3600, stale-while-revalidate=86400` |
| `/api/sites`, `/api/individuals`, `/api/archaeological-contexts`, `/api/dated-samples`, `/api/bones`, `/api/bone-catalog`, `/api/skeletons`, `/api/funerary-contexts`, `/api/dating-techniques`, `/api/dating-results`, `/api/references` | `public, max-age=120, stale-while-revalidate=3600` |
| `/api/auth/*`, `/api/users/me*` | no header set (falls back to Spring Security default `no-store`) |
| anything else (`/api/admin/**`, write methods, non-`GET`) | no header set |

Notes:

- The interceptor only acts on HTTP `GET`. Writes pass through with
  no cache header set.
- The matching uses URL prefixes. `/api/cultures/123` is treated as
  a cultures path; `/api/sites/12/anything` is treated as a sites
  path.
- `/api/dataset/download` is not in the list — it has its own
  application-level cache via `DatasetExportService`.

> Open question (Unverified): the interceptor also matches the
> removed v1 paths `/api/osteological-units`, `/api/specimens`, and
> `/api/burial-groups`. Those entries are dead because no controller
> serves them, but they should be removed alongside the equivalent
> `SecurityConfig` lines. Tracked in
> [11-roadmap.md](./11-roadmap.md).

## Application-level caches

- `DatasetExportService` caches the dataset ZIP bytes for
  `app.dataset.export.cache-ttl-seconds` (default `300`).
- Frontend services (`MapTimelineStatsService`, `CultureService`)
  use `shareReplay` to share results between concurrent subscribers,
  but those are client-side and have no backend impact.

There is no Spring `@Cacheable` in this codebase. A distributed
cache layer (e.g. Redis) is not configured.

## Performance guardrails for future agents

1. Do not turn `spring.jpa.open-in-view` on. Open Session In View
   re-introduces lazy-loading after the transaction closes and
   defeats the entity-graph design.
2. Keep DTO mapping **inside** the `@Transactional` boundary so
   `@EntityGraph` loads stay in one transaction.
3. Avoid widening the existing `@EntityGraph` definitions unless the
   DTO actually needs the data. Wider graphs translate into wider
   joins.
4. Prefer projection endpoints (`/api/individuals/list`,
   `/api/stats/*`, `/api/sites/countries`) over filtering full
   entities client-side. If the public web needs a new aggregated
   read, add a new projection endpoint here rather than walking the
   paged collections.
5. Cap any new public endpoint that performs heavy aggregation
   through `PublicEndpointRateLimiter` (`security/`) and add a
   `Cache-Control` mapping in `CacheControlInterceptor`.
6. Use `setMaxResults(...)` on broad search queries with
   in-memory aggregation; the bone-site search is the existing
   pattern.
7. The `JsonIgnoreProperties` annotations on JPA inverse collections
   exist to break Jackson recursion. Do not remove them. Use the
   summary DTOs in `common/summary/` to represent related entities
   in responses.
8. Compression-friendly JSON: prefer flat shapes; the existing
   summary DTOs gzip very well.

## Known performance caveats

- `IndividualResponse` is heavy (deeply nested). Public list views
  should use `/api/individuals/list`.
- `/api/stats/map-timeline` payload size grows with the number of
  sites × archaeological contexts. Today it is comfortably under
  150 KB uncompressed (≈ 25–35 KB gzipped) but should be monitored
  if site density doubles.
- Wide `@EntityGraph` definitions can fan into large result sets
  when an individual has many bones with dated samples. Re-evaluate
  if a future dataset has individuals with >50 bones.
- The detail endpoints `GET /api/individuals/{id}` and
  `GET /api/archaeological-contexts/{id}` are not cached at the HTTP
  layer beyond `max-age=120`. Stronger caching would require
  versioned URLs or `ETag` support, neither of which is implemented.

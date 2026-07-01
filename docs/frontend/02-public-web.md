# 05 — Public Web (`projects/web`)

The public site at `projects/web` is the research database UI: read-only, fully
public, and currently deployed to Vercel as a prerendered/static browser output
with live client refetches after hydration. Angular still emits an SSR server
bundle, but the public Vercel project currently serves `dist/web/browser` and
does not route requests to that server bundle.

## Route map

| Route | Component | Render mode | Services called |
|---|---|---|---|
| `/` | `HomePage` | Prerender | `HomeStatsService`, `SiteService.getAll({ size: 12 })` |
| `/sites` | `SitesPage` | Prerender | `SiteService.getList`, `SiteService.getCountries` |
| `/sites/:id` | `SiteDetailsPage` | Server config; CSR fallback on current Vercel | `SiteService.getById`, `ArchaeologicalContextService.searchBySite` |
| `/individuals` | `IndividualsPage` | Prerender | `IndividualService.getList` (or `.search`) |
| `/individuals/:id` | `IndividualDetailPageComponent` | Server config; CSR fallback on current Vercel | `IndividualService.getById`, `BoneService.findByIndividual`, `SkeletonService.search`, `ArchaeologicalContextService.getById`, `DatedSampleService.search` |
| `/map` | `MapPage` | Prerender | `MapTimelineStatsService.getMapTimelineStats` |
| `/timeline` | `TimelinePage` | Prerender | `MapTimelineStatsService.getMapTimelineStats`, `CultureService.getAll` |
| `/bibliography` | `BibliographyPageComponent` (lazy) | Prerender | `ReferenceService.getAll` |
| `/bones` | `BonesPage` (lazy) | Prerender | `BoneSiteSearchService.search` |
| `/analysis` | `AnalysisPage` (lazy) | Server config; CSR fallback on current Vercel | `IndividualService.getAllItems({ size: 200 })` (paginated sweep) |
| `/methodology` | `MethodologyPage` | Prerender | — (static content) |
| `/about` | `AboutPage` | Prerender | — (static content) |
| `/dataset` | `DatasetPageComponent` (lazy) | Prerender | `DatasetService.downloadDataset` (on user click) |
| `/explore` | redirect → `/map` | — | — |
| `/chronology` | redirect → `/timeline` | — | — |

Routes that no longer exist on the public web (do not reintroduce without product approval):

- `/units/:id` — removed; archaeological-context data is shown inside parent views.
- `/specimens/*` — removed with the v1 model.
- `/burial-groups/*` — removed with the v1 model; superseded by funerary contexts surfaced through individuals.
- `/cultures/:id` — never implemented on the public web. Open gap.

## Public layout shell

All public pages are children of `PublicShellComponent` (`projects/web/src/app/components/layout/public-shell/public-shell.ts`). The shell renders:

```html
<app-header></app-header>
<router-outlet></router-outlet>
<app-footer></app-footer>
```

The shell is mounted once. `HeaderComponent` and `FooterComponent` persist across route transitions. **Page templates must not include `<app-header>` or `<app-footer>` directly.**

## Header navigation

`HeaderComponent` (`projects/web/src/app/components/header/header.ts`) renders:

```
Primary:  Home · Sites · Individuals · Bones · Map · Timeline · Bibliography
Muted:    Dataset · Methodology · About
```

`Analysis` (`/analysis`) is registered but **not linked** from the nav. It is reachable only by direct URL.

`HeaderComponent` warms the Map page on hover/focus/touch of the nav `Map` link: it dynamically imports the Leaflet chunk, primes the `MapTimelineStats` `shareReplay` cache via `warmMapStats()`, and pre-fetches the initial physical (OpenTopoMap) basemap tiles via `warmPhysicalBasemapTiles()`. The tile warm-up targets the exact z5 tiles the live map requests (it opens at fractional zoom 4.5, which Leaflet rounds to z5). MapLibre / the political vector layer is **not** warmed here — it stays lazy. See [05-performance.md → Map and Timeline loading strategy](./05-performance.md#map-and-timeline-loading-strategy).

## Data loading strategy

- **No route resolvers** anywhere in the public web. Routes activate immediately and components subscribe after activation. The blocking "Loading..." card on slow pages is delayed by the shared `DelayedLoadingState` helper (`projects/web/src/app/utils/delayed-loading-state.ts`, default threshold 300 ms; SSR-safe).
- **HTTP transfer cache is disabled.** `provideClientHydration(withEventReplay(), withNoHttpTransferCache())` is set in `app.config.ts`. The public Vercel project serves prerendered `dist/web/browser` output, so transfer-cached HTTP responses would be build-time snapshots. Disabling the HTTP transfer cache lets the browser refetch live API data after hydration. The backend serves anonymous public reads with `Cache-Control: no-cache`, so those refetches revalidate against origin.
- **Retry interceptor** (`projects/web/src/app/core/interceptors/retry-interceptor.ts`) retries idempotent failed GETs.

## Per-page detail

### `/` — Home

- Sections:
  1. Hero with stats (`totalIndividuals`, `totalBoneRemains`, `totalSkeletons`, `totalSites`) and CTAs to `/sites`, `/individuals`, plus a muted link to `/dataset`.
  2. Site ticker (first 12 sites, duplicated for marquee).
  3. Research access triptych: geographic / biological / chronological cards.
- Optional "Database last updated" line, shown only when the backend supplies `HomeStats.lastUpdatedAt`. The frontend never fabricates a date.
- Stats source of truth: `GET /api/stats/home`. The Home page must **not** reconstruct counts from collection endpoints.
- The site ticker calls `SiteService.getAll(..., { forceRefresh: true })` so the visible homepage does not reuse a warmed site-list cache entry.

### `/sites` — Sites list

- Paginated site card grid (default `pageSize = 20`).
- Backend-supported text search (`q`) and country dropdown.
- Sort: site name only.
- Data flow is backend-driven through `SiteService.getList({ q, country, page, size, sort }, { forceRefresh: true })`.
- The country options come from `SiteService.getCountries({ forceRefresh: true })` (`GET /api/sites/countries`) on page activation.
- `SiteService` keeps a short-lived, query-keyed in-memory cache for warm preloads/helper calls only. Visible `/sites` loads bypass it.

### `/sites/:id` — Site detail

- Hero (left): breadcrumb, site h1, location line, coordinates, KPI strip (Contexts, MNI, Individuals, Dated samples).
- Hero (right): inline Leaflet mini-map with one marker. Always uses the OpenTopoMap physical base layer; the Physical / Political toggle from MapPage is not exposed here. The mount point shows a plain white background while the (external) tiles load, so the panel never flashes Leaflet's default grey and there is no decorative/cartographic placeholder; the live tiles then paint over it. See [05-performance.md](./05-performance.md#map-and-timeline-loading-strategy).
- Body: archaeological-context cards sorted from oldest culture (`culture.startBp` desc) to most recent. Title prefers `stratigraphicContext`, falls back to `culture.cultureName`, then to "Archaeological context". `archaeologicalContextId` is rendered as small metadata under the title.
- References: deduplicated list, each rendered as a formatted citation (journal article, article/chapter in book, or complete book — see [`/bibliography`](#bibliography)).
- Loads only `GET /api/sites/:id` and `GET /api/archaeological-contexts?siteId=`. Does **not** load `/api/stats/map-timeline` or full collection endpoints.

### `/individuals` — Individuals list

- Paginated table with columns: combined `Individual / Site` identity column, `Sex`, `Age class`, `Culture`, `Country`.
- Backend-supported filters: `q`, `sex` (with sex-certainty variants `Male?`, `Female?`, `Indeterminate?`), `ageClassMain`.
- Backend-supported sort: individual name, sex, main age class.
- Active-filter indicator with "Clear all".
- Click row → `/individuals/:id`.
- Public list uses `GET /api/individuals/list` (lightweight projection). The full `Individual` endpoint is reserved for the detail page.
- Visible `/individuals` loads pass `forceRefresh=true`. `IndividualService` keeps a short-lived, query-keyed in-memory cache for warm preloads/helper calls only.

### `/individuals/:id` — Individual detail

Progressive component-local loading after route activation. See [01-frontend-architecture.md](./01-frontend-architecture.md) for the phase model.

Sections (numbered, sticky quick-facts sidebar):

1. Biological profile (sex, certainty, age, age class).
2. Archaeological context (linked site, culture, stratigraphy).
3. Funerary context (burial context/type, boolean flags).
4. Bone inventory (table fed by `BoneService.findByIndividual`). The endpoint returns bones **associated** with this individual; because bone↔individual is M2M (`bone_individual`), a listed bone may also be linked to other individuals — the table does not imply exclusive ownership.
5. Skeletons (table fed by `SkeletonService.search({ individualId })`).
6. Dating (table laid out as **Origin · Dating result · Material · Method**, fed by `DatedSampleService.search`).
7. Bibliography (deduplicated from the full archaeological context).

Loading states:

- Phase 1 shows a full-page card delayed by 300 ms.
- Phases 2/3 use per-section inline spinners.
- Each enrichment branch carries its own error signal (`bonesError`, `skeletonsError`, `datingError`, `referencesError`); a single failed branch never blanks the rest.

### `/map` — Map

- Leaflet map (browser-only, dynamic import). Keyless basemaps with two selectable modes:
  - Physical: OpenTopoMap raster tiles — default.
  - Political: OpenFreeMap vector tiles through MapLibre GL Leaflet.
- Search by site name + country filter + culture multi-select.
- `PaleoTimeControlComponent` toggles a palaeogeographic reconstruction overlay (ice sheets, emerged land by cal BP slice). The tile source for the overlay is external and not present in this repository.
- Click a marker → right-hand site preview panel (totals, culture chips, dominant culture). Click empty map → close panel.
- Marker design is shared with the SiteDetailsPage mini-map via `projects/web/src/app/utils/map-markers.ts`. Markers are neutral — cultures carry no colour anywhere in the public web (filters identify cultures by name/id only).
- Data source: **only** `GET /api/stats/map-timeline`. Do not load unfiltered `/api/archaeological-contexts`.
- `/map` passes `forceRefresh=true` to `MapTimelineStatsService` on activation so a Home/Header warm-up cannot keep a stale aggregate snapshot visible for the cache TTL window.

### `/timeline` — Timeline

- Track: fixed left column (culture name, BP range, filtered site count) + scrollable right (phase headers, time axis, culture bands with site dots; dot size = MNI).
- Culture filter (multi-select). Search + country + reset.
- Zoom slider (100–400 %).
- Click dot → right-side site detail panel; "Open site" link navigates to `/sites/:id`.
- Data source: `GET /api/stats/map-timeline` is the source of truth; `GET /api/cultures` is called only to enrich the secondary culture cards (`description`, `region`, `features`).
- `/timeline` passes `forceRefresh=true` for both reads so culture/site edits show on activation/hydration with no in-memory cache wait.

### `/bones` — Bones (site-first search)

- Sections: search/filter panel, summary metric strip (6 totals), site result cards, expandable inventory per card.
- Filters wired to `GET /api/bone-site-search`. Filter option lists are populated from the `filters` envelope returned by the backend, never hardcoded.
- Sort options: `minimumBoneRemainCount,desc`, `matchingBoneRecordCount,desc`, `individualCount,desc` (renamed from v1 `specimenCount`), `siteName,asc`, `country,asc`.
- The page never falls back to `/api/archaeological-contexts` or `/api/bones`.
- Aggregate counts (`individualCount`, `matchingBoneRecordCount`, `minimumBoneRemainCount`, …) come from the backend and **must not be reconstructed client-side**. Because a bone may be associated with several individuals (M2M `bone_individual`), the backend already deduplicates by `bone_id`; re-summing nested inventory rows would double-count shared bones.

### `/bibliography`

- Paginated reference list via `ReferenceService.getAll({ q, page, size })` → `GET /api/references`.
- Backend `q` search, page-size selector, prev/next.
- Results grouped by exact publication year within the currently loaded page (newest year first; references with no year fall under an `Unknown year` group).
- Lazy-loaded route (`loadComponent`).
- Visible `/bibliography` loads pass `forceRefresh=true`. `ReferenceService` keeps a short-lived, query-keyed in-memory cache for warm preloads/helper calls only.
- Each reference renders as a formatted citation via the local helper
  `features/references/reference-citation.ts`, which selects one of three project
  citation shapes from the `isBook` / `isArticleInBook` flags:
  - **Journal article** (default; also used when both flags are null) —
    `Authors, Year. Title. Journal Volume (Issue), Pages. DOI`
  - **Article/chapter in book** (`isArticleInBook = true`) —
    `Authors, Year. Title, in: Editor(s), Book title, Corporate author, City, Pages.`
  - **Complete book** (`isBook = true`) —
    `Authors, Year. Book title. Corporate author, City, Pages.`
  Missing/blank fields are dropped without leaving empty punctuation (no doubled
  commas/periods, no empty issue parentheses, no dangling `in:`). The DOI renders
  only when present, as a clickable `https://doi.org/…` link.
- **Typographic convention & line layout.** The helper exposes the citation both
  as flat styled `referenceCitationSegments()` (`{ text, kind }` fragments) and,
  for the reading pages, as `referenceCitationLines()` — the same content
  regrouped onto three readable lines (`authorsLine` · `titleLine` ·
  `metadataLine`, plus `doiUrl`) and rendered by the local
  `ReferenceCitationComponent` (`features/references/reference-citation.component.ts`):
  - **Line 1** — authors and year.
  - **Line 2** — the main work title (`title`, or `book` for a complete book), in **bold**.
  - **Line 3** — publication/container metadata: journal + volume/issue/pages for an
    article; `in:` + editor/book/corporate author/city/pages for a chapter;
    corporate author/city/pages for a complete book. The **container title is
    italic** (journal for an article, the book for a chapter; a complete book's
    `book` is the bold title and has no separate container). Authors/year/metadata
    stay normal weight; lines are justified where they wrap.
  The required citation order, punctuation and plain-text output are unchanged
  (`referenceCitationParts` / `formatReferenceCitation` derive from the same flat
  segments, and the line grouping reuses those building blocks). Absent fields
  collapse to empty lines that the template skips — no blank lines or stray
  punctuation. The **DOI link is preserved** on each page (clickable
  `https://doi.org/…`). The same component is reused on `/bibliography`,
  `/sites/:id` and `/individuals/:id`, so the layout is consistent across every
  public reading page; it carries its own scoped `rc-*` styles and never touches
  the DEEP TIME `--dt-*` tokens. The split is purely visual and trivially
  revertible (it does not alter the citation string or any data).
- The public `Reference` model (`features/references/model/Reference.ts`) mirrors
  the backend contract: `authors` and `year` are the only universally required
  fields; `title` and `journal` are nullable; complete books store their title in
  `book`. Fields: `authors`, `year`, `title`, `editor`, `book`, `corporateAuthor`,
  `city`, `journal`, `volume`, `issue`, `pages`, `doi`, `isBook`, `isArticleInBook`.

### `/analysis`

- The most complex page in the app. Multi-select chip filters (country, culture phase, culture, sex, age class, individual type, burial type), chart config panel, Chart.js charts (bar/timeline/heatmap/scatter), inferential stats (χ², Cramér's V, Pearson r, descriptive stats), CSV export.
- Data flow: `IndividualService.getAllItems({ size: 200 })` (paginated sweep) → derives site, culture, individual type, and funerary data from each individual's embedded summaries.
- **Uses legacy `*ngIf`/`*ngFor`.** Other pages use `@if`/`@for`. See [07-roadmap.md](./07-roadmap.md).
- Render mode: configured as `Server`, but current Vercel static hosting rewrites `/analysis` to `/index.csr.html`. It is therefore client-rendered in production until the public project is changed to deploy the Angular SSR server function.

### `/dataset`

- Single export action panel. Button calls `DatasetService.downloadDataset()` and triggers a browser download.
- The downloaded ZIP bundles the scientific CSV tables, a `README.md` (citation + license), and a `manifest.csv` (file list with row counts); the page copy mentions these.
- Browser-only; SSR-safe.
- Linked from the muted header, footer, and a small secondary Home link.

### `/methodology` and `/about`

- Pure static content. No API calls.
- About hero hardcodes `804 individuals / 6,604 bones / 248 sites / 20 countries` and Methodology section 5 hardcodes `M: 98 / F: 68 / Indet: 624`. Both numbers come from the published article snapshot, not live data. See [07-roadmap.md](./07-roadmap.md).

## Shared components

| Component | Path | Purpose |
|---|---|---|
| `PublicShellComponent` | `components/layout/public-shell/public-shell.ts` | Top-level layout for all public routes |
| `HeaderComponent` | `components/header/header.ts` | Top nav, mobile drawer, muted items |
| `FooterComponent` | `components/footer/footer.ts` | Site footer |
| `PaleoTimeControlComponent` | `components/paleotime-control/paleotime-control.ts` | Palaeogeographic overlay toggle |
| `PaleoTimeSlice` | `components/paleotime-control/paleotime-slice.ts` | Slice model |
| `BoolFlagComponent` | `components/boolean-flag-display/boolean-flag-display.ts` | Yes/No/Unknown display for Boolean flags |
| `DateRangeDisplayComponent` | `components/date-range-display/date-range-display.ts` | BP range display |
| `DataPanelComponent` | `components/data-panel/data-panel.ts` | Generic data display panel |
| `TemporalPositionBarComponent` | `components/temporal-position-bar/temporal-position-bar.ts` | Visual chronological position bar |
| `TerrainMapComponent` | `components/terrain-map/terrain-map.ts` | Embedded Leaflet map component |
| `DelayedLoadingState` | `utils/delayed-loading-state.ts` | Shared delayed visibility helper (not a rendered component) |

## SSR / browser-only rules

- Every page that uses Leaflet, `window`, `document`, `localStorage`, or `IntersectionObserver` must guard with `isPlatformBrowser(PLATFORM_ID)`.
- Leaflet/MapLibre structural CSS lives in the `styleUrls` of `MapPage` and `SiteDetailsPage` only. Do not add map library CSS to `projects/web/src/styles.css`.
- The Material Symbols `<link>` is loaded once from `index.html`.
- The home page hero JPG and the logo PNG live under `projects/web/public/`. Production renames assets via `outputHashing: 'all'`.

## Current functionality (snapshot)

The public web supports, today:

- Home with live stats + ticker.
- Site list with country filter + pagination.
- Site detail with archaeological contexts and a mini-map.
- Individuals list with backend filters.
- Individual detail with progressive enrichment (biological profile, archaeological context, funerary context, bones, skeletons, dating, bibliography).
- Map with OpenTopoMap/OpenFreeMap base layers, culture filter, palaeogeographic overlay, neutral markers.
- Timeline with culture filter, zoom, site detail panel.
- Bibliography with paginated search.
- Site-first bones search at `/bones`.
- Statistical analysis at `/analysis` (direct URL).
- Dataset download at `/dataset`.
- Methodology and About static pages.

## Known gaps

Items that are confirmed missing on the public web:

- **No `/cultures/:id` page.** Culture editorial content only appears as cards on `/timeline`.
- **No standalone funerary-context / multi-burial page.**
- **Hardcoded stats** on About and Methodology pages.
- **Citation year drift:** Home page citation may show 2026 while About shows 2024. `Requires code verification`.
- **`Export RIS` button** on Home is disabled with no explanation.

See [07-roadmap.md](./07-roadmap.md) for the active improvement backlog.

## Safe-to-modify guidance for future agents

When a future agent works on the public web, the following changes are safe (use existing endpoints, no backend coordination required):

- Visual design, typography, color, spacing, layout, mobile breakpoints.
- New routes that re-use existing data (e.g. `/cultures/:id` via `CultureService.getById`).
- Filter UX (chips, drawers, drawers, facets) that maps cleanly to existing query params.
- Replacing hardcoded stats with `GET /api/stats/home` values.
- Refactoring `/analysis` templates to `@if`/`@for`.
- Moving Chart.js from CDN injection to an npm import.

Changes that need backend coordination:

- New filter parameters (`country` / `cultureId` on `/individuals`, calibrated BP fields).
- New aggregate endpoints (e.g. `GET /api/stats/analysis`, `GET /api/individuals/{id}/bundle`).
- Cache-Control header changes (must come from Spring or the deployment edge).
- Adding `lastUpdatedAt` to `HomeStats`.

Changes that are out of scope for the public web entirely:

- Anything in `projects/backoffice`.
- The shared models in `projects/backoffice/src/app/features/*` (they are not imported by the public web).
- Database schema.

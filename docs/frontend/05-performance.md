# 08 — Performance

This document records the **current performance shape** of the public web, including the loading strategies that are already in place and the constraints that future agents must respect.

> All measurements quoted here come from a 2026-05-10 audit against the live API. Numbers shift over time; treat them as orientation, not contract. Re-measure before claiming a regression.

## Architectural posture

- **SSG/SSR via `@angular/build:application`**, `outputMode: server`.
- **Per-route render mode** declared in `app.routes.server.ts`. Most routes are prerendered; `/sites/:id`, `/individuals/:id`, and `/analysis` are `RenderMode.Server`. See [01-frontend-architecture.md](./01-frontend-architecture.md).
- **Hydration transfer cache enabled.** `provideClientHydration(withEventReplay())` is set; the older `withNoHttpTransferCache()` opt-out was removed. SSR/SSG-fetched payloads survive hydration when the API returns the correct `Cache-Control`.
- **Retry interceptor** for idempotent GETs in `projects/web/src/app/core/interceptors/retry-interceptor.ts`.
- **Service-level in-memory caches:** `MapTimelineStatsService.getMapTimelineStats()` and `CultureService.getAll()` use `shareReplay({ bufferSize: 1, refCount: false })`. `SiteService` (after the 2026-05-12 audit) maintains a bounded **query-keyed cache** of `getAll` / `getList` results plus a single-shot `getCountries()` cache, so a navigation from `/` to `/sites` that was pre-warmed by HomePage hits the cache synchronously and skips the blocking loader.

## Heavy endpoints (current)

| Endpoint | Size on the wire | Frontend usage |
|---|---|---|
| `GET /api/stats/map-timeline` | ~130 KB JSON (uncompressed) | `/map` and `/timeline` main render |
| `GET /api/individuals?page=0&size=200` | ~2.5 MB JSON | `/analysis` (paginates 4 times for 773 individuals) |
| `GET /api/individuals?page=0&size=20` | ~119 KB JSON | `/individuals` list (deeply nested per row) |
| `GET /api/bone-site-search?page=0&size=10` | ~110 KB JSON | `/bones` |

The `/individuals/list` lightweight projection exists specifically to shrink the public list payload. The full `/api/individuals` endpoint should not be used by list views.

## Lightweight stats endpoints

These should always be preferred over reconstructing counts client-side:

- `GET /api/stats/home` — drives the Home hero stats.
- `GET /api/stats/map-timeline` — drives the Map and Timeline visualisations and provides per-site aggregate counts.
- `GET /api/admin/stats/dashboard` — drives the backoffice dashboard.
- `GET /api/bone-site-search` — provides per-site aggregate counts and the bones-page summary metrics.

Reconstructing these values from collection endpoints (full sweeps of `/api/sites`, `/api/individuals`, `/api/bones`) is **prohibited** for the relevant pages. Past regressions came from re-introducing such sweeps when the stats endpoints were missing fields.

## Pagination rules

- Public list pages use backend pagination (no client-side pagination over a full sweep) **whenever the backend supports the necessary filters**.
- The `/sites` page falls back to a cached full sweep only when a country filter is active and the backend country filter is unavailable. With the now-shipped `country` query param and `/api/sites/countries` endpoint, the fallback can be retired.
- `/analysis` is a documented exception: it sweeps `/api/individuals` four times to compute statistics. This is acknowledged technical debt (see [07-roadmap.md](./07-roadmap.md)).

## HomePage hero loading strategy

- **Hero asset is responsive AVIF / WebP / JPG**, not a CSS background. The source is `projects/web/public/hero-image.jpg` (2400×1350); a one-shot `scripts/optimize-hero.mjs` regenerates `hero-image-{1280,1920,2400}.{avif,webp,jpg}` variants — AVIF lands at ~55 KB / 105 KB / 160 KB, down from the original 562 KB JPG (~80–90 % reduction).
- **`<picture>` markup** in `pages/home-page/home-page.html` serves AVIF first, WebP next, and the smallest JPG as fallback. The element uses `width=2400 height=1350` (no layout shift), `fetchpriority="high"`, `loading="eager"`, and `decoding="async"` so the browser treats it as the LCP candidate.
- **`<link rel="preload" as="image">` in `index.html`** points at `hero-image-1920.avif` with `imagesrcset` / `imagesizes`, so the AVIF download starts in parallel with the JS chunk.
- **DEEP TIME-tinted fallback backdrop** (`.hp-hero__bg`) is always visible underneath the picture; the optimised image fades in via a CSS transition once the `<img>` fires its `load` event (`onHeroImageLoad`). The hero never shows an empty gradient or "broken" state — even before any image bytes have arrived the surface reads as an intentional dark tableau. `prefers-reduced-motion: reduce` disables the fade-in.

## Sites list cache and preload

- **HomePage warms the same query SitesPage will issue.** `HomePage.warmSitesPagePreload()` calls `SiteService.warmSitesFirstPage()` (page 0, size 20, `siteName,asc` — the exact SitesPage default) plus `SiteService.getCountries()`. Both share an in-memory `shareReplay` cache.
- **SitesPage skips the blocking loader on a warm cache hit.** `loadSites()` calls `SiteService.hasCachedQuery(...)` before starting the `DelayedLoadingState`; on a cache hit the data swap is synchronous and the 300 ms loader never enters the DOM.
- **Country dropdown is hydrated synchronously** from `SiteService.getCachedCountries()` when warm, so the dropdown is populated on first paint.
- **Search / pagination / sorting / country filters all flow through the same cache key** (`q | country | page | size | sort`). Re-visiting a page already viewed in the session is therefore free, while filter / search changes still issue a fresh request.
- The cache is bounded to 24 entries (a recent-history LRU); past that the oldest entry is evicted, so a long browsing session does not retain hundreds of stale subscriptions.

## Map and Timeline loading strategy

> **Diagnosis (2026-06-27, HTTP-trace + Chromium/CDP audit).** The dominant perceived-performance cost on `/map` and `/sites/:id` is **external OpenTopoMap tile latency**, not the API or database. `GET /api/stats/map-timeline` returns in ~80-140 ms locally (~15 KB gzip) and is **not** the bottleneck; it remains the single core data source for `/map`. Individual cold OpenTopoMap tiles were observed taking 1.3-15 s each, and the previous UI tied the markers to the basemap `load` event — so one slow tile could leave the map dark and the markers hidden for 3-9+ s. The fixes below make the map useful immediately and decouple markers from tile latency, but they cannot remove the provider latency itself (see "Backend / deployment blockers" #5).

- **No route resolvers.** Both routes activate immediately. The blocking page loader is gated by `DelayedLoadingState` (300 ms threshold).
- **Leaflet is dynamically imported** in browser context only.
- **Static-first map shell.** `/map` (`.mp-el`) and the `/sites/:id` mini-map (`.sdp-hero__map-el`) paint a local, prerendered cartographic SVG (`projects/web/public/map-shell.svg` — graticule + abstract landmasses + contour hints, ~4 KB, DEEP TIME palette) as their CSS background. It is part of the prerendered HTML, so a meaningful map surface is visible at first paint regardless of tile latency. The Leaflet container mounted inside is transparent (`app-map-page .leaflet-container { background: transparent }`), so the shell shows through until external tiles fade in over it, then sits subordinate behind them. This **replaced** the former dark `.mp-terrain-veil` (a delayed striped gradient): the shell is visible immediately and reads as an intentional map rather than a loading state. The dark `--dt-map-bg` background on `.mp-map-wrap` / `.mp-el` / the mini-map element is kept as the ultimate fallback behind the shell, so the surface never flashes Leaflet's default grey.
- **MapPage** initialises the Leaflet shell (map instance, base tile, controls, marker layer) as soon as the view mounts, then waits for `/api/stats/map-timeline` to render markers. The status indicator is a small non-blocking chip.
- **Markers no longer wait for tiles.** `MapPage.tryRenderMarkers()` renders the marker layer as soon as the Leaflet shell **and** the stats data are ready (`shellInited && dataReady`). It no longer waits for the basemap's first `load` event. Because the static shell already provides geographic context, markers are never shown over a blank canvas, and a single slow OpenTopoMap tile can no longer hold them hostage. Markers are positioned in Leaflet's coordinate space, so they align with the live tiles once those fade in. `tilesReady` now only drives the map region's `aria-busy` hint.
- **Basemap fade-in.** Physical uses OpenTopoMap raster tiles; Political uses OpenFreeMap vector tiles through MapLibre GL Leaflet. New base layers start visually hidden and fade in after the Leaflet tile event or MapLibre GL load/error event fires, with a 2.5 s safety timeout. Raster tiles set `updateWhenZooming: false` (skips throwaway intermediate-zoom requests during the fractional-zoom animation) in `utils/base-map-layer.ts`; provider, URL, and attribution unchanged. `keepBuffer` is intentionally left at Leaflet's default of 2 — a browser smoke test confirmed it does not change the initial tile-request count (it only governs off-screen tile retention while panning, in `GridLayer._pruneTiles`), so raising it adds memory pressure with no cold-load benefit.
- **Warm-up targets the real initial tile zoom.** The live map opens at fractional zoom 4.5; Leaflet rounds tile requests to `Math.round(4.5) = z5`. The warm-up previously fetched **z4** tiles the live map never requested, so a "warm" cold-open still paid full tile latency. `EUROPE_INITIAL_CENTER` / `EUROPE_INITIAL_ZOOM` / `EUROPE_INITIAL_TILE_ZOOM` now live in `projects/web/src/app/utils/map-viewport.ts` and are the single source of truth for both the live map and the warm-up, so the warm-up primes the exact z5 tile block the map paints first.
- **Nav-intent warm-up.** Hovering / focusing / touching the `Map` nav link triggers `warmLeafletChunk()`, `warmMapStats(...)`, and `warmPhysicalBasemapTiles()` (`projects/web/src/app/utils/map-preload.ts`); the last warms a small z5 OpenTopoMap block around the initial view. All are idempotent, browser-only, and respect Save-Data / slow-2g. **MapLibre / the political vector layer stays lazy** — it is only pre-warmed by HomePage during idle (`warmBasemaps()`), never on the header hover path or the default physical render. `index.html` preconnects all three OpenTopoMap subdomains (`a`/`b`/`c`, loaded in parallel) and keeps OpenFreeMap on a lighter `dns-prefetch`.
- **MapTimelineStatsService caches via `shareReplay({ bufferSize: 1, refCount: false })`.** Live until page reload. `/sites/:id` does **not** request `/api/stats/map-timeline`.

> **Pre-change baseline (CDP audit, local):** cold `/map` — Leaflet container ~1.38 s, first tile request ~1.38 s, first tile painted ~2.54 s, markers ~3.37 s (one outlier pass: no tiles/markers after 9 s). Cold `/sites/1` mini-map — first tile ~3.38 s.
>
> **Post-change measurement (2026-06-27, headless Chrome via Playwright against the production build served by the local SSR server; map data supplied to the browser so markers render).** `/map`: meaningful map surface (shell painted) ~20-30 ms, FCP ~85-110 ms, markers rendered ~0.2-0.3 s, first OpenTopoMap tile painted ~0.7-1.2 s, basemap "ready" (`aria-busy` cleared) ~2.3-5.3 s depending on live tile latency. **Markers stayed at ~0.3 s even on a run where basemap-ready was 5.3 s**, confirming markers are decoupled from tile load. Initial OpenTopoMap tile requests: **32** on `/map` (1280×800), **9** on the `/sites/1` mini-map — identical for `keepBuffer` 1, 2, and 4, confirming `keepBuffer` does not affect cold-load tile pressure. `Note:` numbers are headless/local and vary with external OpenTopoMap latency; treat as orientation, re-measure on real devices/networks for field data.

## Individual detail loading strategy

Documented progressively-loading pattern:

1. **Phase 1.** `IndividualService.getById(id)` renders the shell.
2. **Phase 2.** Parallel `forkJoin`: `BoneService.findByIndividual(id)`, `SkeletonService.search({ individualId })`, `ArchaeologicalContextService.getById(contextId)`. The `/bones?individualId=` filter still works but now resolves through the `bone_individual` bridge, so a returned bone may also be linked to other individuals — the inventory does not imply exclusive ownership.
3. **Phase 3.** `DatedSampleService.search({ archaeologicalContextId | boneId | skeletonId })` fanned out per linked entity and deduped by `datedSampleId`. `datingResults[]` already arrives embedded; no separate `/dating-results` call.

> The Phase 3 fan-out is the page's biggest cost. For an individual with `N` bones, Phase 3 issues at least `N + linked-skeleton + 1` requests. Collapsing this into a `GET /api/individuals/{id}/bundle` is the highest-impact future backend change (see [07-roadmap.md](./07-roadmap.md)).

## Practical warnings for future agents

1. **Do not reintroduce route resolvers.** The public web is intentionally resolver-free. Loading is component-local and progressive, gated by `DelayedLoadingState`.
2. **Do not import global map library CSS.** Leaflet/MapLibre structural CSS lives in the `styleUrls` of `MapPage` and `SiteDetailsPage` only. Adding map CSS back to `styles.css` regresses every non-map route.
3. **Do not reconstruct stats from collections.** Use the relevant `/api/stats/*` or `/api/bone-site-search` endpoint. If a needed field is missing, raise it in [07-roadmap.md](./07-roadmap.md), do not paper over it with a sweep. In particular, bone totals join through `bone_individual`, so a bone shared by several individuals must be counted once (`COUNT(DISTINCT bone_id)`) — never sum per-individual bone lists.
4. **Do not fabricate `lastUpdatedAt`.** If the backend does not supply it, the line is hidden. Do not derive it from a partial collection scan.
5. **Do not type `/bones?individualId=` as `PaginatedResponse`.** Use `BoneService.findByIndividual()`, which expects a `Bone[]` array shape.
6. **Do not block first paint on heavy collections.** Stats endpoints first, enrichment after.
7. **Respect `RenderMode.Server` on parametric routes.** Switching `/sites/:id` and `/individuals/:id` to `Prerender` requires either `getPrerenderParams` for the full set or a build-time API contract that can serve every id. Either is a deliberate decision, not a flip.
8. **Service caches are in-memory.** `shareReplay` does not persist across reloads. Adding `localStorage` / `sessionStorage` / Service Worker caches would alter SSR semantics.

## Known regressions to watch for

When any of these patterns reappear, treat it as a regression:

- A page that re-fetches a value already SSG-baked into its HTML on first paint.
- A non-map page importing map library structural CSS.
- A page calling `IndividualService.getAllItems({ size: 200 })` outside `/analysis`.
- A page deriving Home stats from `/api/sites` and `/api/individuals` instead of `/api/stats/home`.
- A page calling unfiltered `/api/archaeological-contexts` from Map or Timeline.
- A blocking loader that is not gated by `DelayedLoadingState`.
- `/map` markers re-gated behind the basemap `load` event (the static shell makes that unnecessary and reintroduces the 3-9 s marker delay under slow tiles).
- The map warm-up warming a zoom that differs from `Math.round(EUROPE_INITIAL_ZOOM)`, or `map-preload.ts` / `map-page.ts` re-introducing duplicated centre/zoom constants instead of importing `utils/map-viewport.ts`.
- MapLibre eagerly loaded on the default physical `/map` render or on the header hover path (it must stay lazy / HomePage-idle-only).

## Asset and font loading

- **Hero variants** are produced offline by `scripts/optimize-hero.mjs` (uses `sharp`, not a runtime dep). The generated AVIF / WebP / JPG files live alongside `hero-image.jpg` in `projects/web/public/` and are served via `<picture>` with `srcset`.
- **Logo variants** are produced by `scripts/optimize-logo.mjs`. The header `<picture>` serves `logo-64.webp` (≈2.5 KB) / `logo-128.webp` (≈5.8 KB) with `<img>` fallback to the matching PNGs. The original `logo.png` (≈237 KB) is kept only as the legacy filename in case external referrers depend on it.
- **Material Symbols** is subsetted via Google Fonts' `icon_names=ac_unit,info,public,water` parameter — the only icons rendered (inside `paleotime-control`). The stylesheet is loaded non-render-blocking via `media="print" onload="this.media='all'"`.
- **Space Grotesk** is loaded with the default Google `display=swap`; text remains visible during font fetch.

## Backend / deployment blockers (frontend-visible)

These items are out of scope for `projects/web` but their absence has a direct, measurable impact on perceived performance. They are tracked here so future agents do not chase them in the frontend.

1. **No `Cache-Control: public, max-age=…` on read-only endpoints.** Without it the hydration transfer cache cannot survive a hard reload and edge caches cannot help. Largest visible cost: `GET /api/stats/map-timeline` (~130 KB) re-downloads on every map visit.
2. **No `Content-Encoding: br` / `gzip` on JSON.** The map/timeline stats payload (~130 KB), bone-site-search (~110 KB), and the `/analysis` sweep (~2.5 MB total) all ship uncompressed. Compression would cut wire bytes by 5-10x.
3. **No `GET /api/individuals/{id}/bundle`.** Phase-3 fan-out on individual detail still issues `N + linked-skeleton + 1` `dated-samples` requests in parallel. Collapsing into a single bundle endpoint is the single highest-impact backend change for that page.
4. **No `GET /api/stats/analysis`.** `/analysis` still sweeps `/api/individuals` four times (≈2.5 MB total) to compute statistics. A per-chart aggregate endpoint would shrink this to one small request.
5. **External basemap latency.** OpenTopoMap raster tiles (default physical layer) and OpenFreeMap vector tiles (lazy political layer) are external dependencies and the dominant perceived-performance cost on `/map` and `/sites/:id` — individual cold tiles were observed taking 1.3-15 s each. Frontend mitigations now in place (see "Map and Timeline loading strategy"): the static-first map shell, decoupling markers from tile load, a tile-zoom-correct nav-intent warm-up, and all-subdomain preconnects. These hide the wait and remove the marker-gating regression, but cannot remove provider latency. The only structural fix is self-hosting raster tiles / PMTiles or moving to a faster provider — tracked as future work in [07-roadmap.md](./07-roadmap.md) and [../08-roadmap.md](../08-roadmap.md).

## Out of scope here

- Backend response compression, `Cache-Control` strategy, and SSR edge caching live in the `backend/` service and the deployment platform (Vercel / Railway). They are flagged in the "Backend / deployment blockers" section above because they directly affect the frontend, but they are not modifiable from `projects/web`. See [../backend/05-performance-and-pagination.md](../backend/05-performance-and-pagination.md).
- Lighthouse / WebPageTest field metrics. The audit referenced above is HTTP-trace-based and code-based, not field-measured.

# 08 — Performance

This document records the **current performance shape** of the public web, including the loading strategies that are already in place and the constraints that future agents must respect.

> All measurements quoted here come from a 2026-05-10 audit against the live API. Numbers shift over time; treat them as orientation, not contract. Re-measure before claiming a regression.

## Architectural posture

- **SSG/SSR via `@angular/build:application`**, `outputMode: server`.
- **Per-route render mode** declared in `app.routes.server.ts`. Most routes are prerendered; `/sites/:id`, `/individuals/:id`, and `/analysis` are `RenderMode.Server`. See [02-architecture.md](./02-architecture.md).
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
- `/analysis` is a documented exception: it sweeps `/api/individuals` four times to compute statistics. This is acknowledged technical debt (see [10-roadmap.md](./10-roadmap.md)).

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

- **No route resolvers.** Both routes activate immediately. The blocking page loader is gated by `DelayedLoadingState` (300 ms threshold).
- **Leaflet is dynamically imported** in browser context only.
- **MapPage** initialises the Leaflet shell (map instance, base tile, controls, marker layer) as soon as the view mounts, then waits for `/api/stats/map-timeline` to render markers. The status indicator is a small non-blocking chip.
- **Basemap fade-in.** Physical uses OpenTopoMap raster tiles; Political uses OpenFreeMap vector tiles through MapLibre GL Leaflet. New base layers start visually hidden and fade in after the Leaflet tile event or MapLibre GL load/error event fires, with a 2.5 s safety timeout.
- **Terrain loading veil.** A soft `.mp-terrain-veil` (dark radial gradient with low-contrast striations) sits above the Leaflet container and below markers / status chip until the active basemap layer reports ready (or its safety timeout fires). Combined with the hard-pinned `#0d0d0d` background on `.mp-map-wrap` and `.leaflet-container`, this guarantees the map surface never appears white at any moment — even in light theme, even on cold loads while raster tiles or the vector style are still on the wire.
- **Markers wait for `tilesReady`.** `MapPage.tryRenderMarkers()` no longer renders the marker layer as soon as the shell + data are ready. It also waits for the basemap's first `load` event (or its safety timeout). Markers therefore never appear over a blank canvas.
- **Nav-intent warm-up.** Hovering / focusing / touching the `Map` nav link triggers `warmLeafletChunk()` and `warmMapStats(...)` (`projects/web/src/app/utils/map-preload.ts`). Both are idempotent and browser-only. `index.html` adds restrained resource hints for OpenTopoMap and OpenFreeMap hosts.
- **MapTimelineStatsService caches via `shareReplay({ bufferSize: 1, refCount: false })`.** Live until page reload.

## Individual detail loading strategy

Documented progressively-loading pattern:

1. **Phase 1.** `IndividualService.getById(id)` renders the shell.
2. **Phase 2.** Parallel `forkJoin`: `BoneService.findByIndividual(id)`, `SkeletonService.search({ individualId })`, `ArchaeologicalContextService.getById(contextId)`.
3. **Phase 3.** `DatedSampleService.search({ archaeologicalContextId | boneId | skeletonId })` fanned out per linked entity and deduped by `datedSampleId`. `datingResults[]` already arrives embedded; no separate `/dating-results` call.

> The Phase 3 fan-out is the page's biggest cost. For an individual with `N` bones, Phase 3 issues at least `N + linked-skeleton + 1` requests. Collapsing this into a `GET /api/individuals/{id}/bundle` is the highest-impact future backend change (see [10-roadmap.md](./10-roadmap.md)).

## Practical warnings for future agents

1. **Do not reintroduce route resolvers.** The public web is intentionally resolver-free. Loading is component-local and progressive, gated by `DelayedLoadingState`.
2. **Do not import global map library CSS.** Leaflet/MapLibre structural CSS lives in the `styleUrls` of `MapPage` and `SiteDetailsPage` only. Adding map CSS back to `styles.css` regresses every non-map route.
3. **Do not reconstruct stats from collections.** Use the relevant `/api/stats/*` or `/api/bone-site-search` endpoint. If a needed field is missing, raise it in [10-roadmap.md](./10-roadmap.md), do not paper over it with a sweep.
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
5. **External basemap latency.** OpenTopoMap raster tiles and OpenFreeMap vector tiles are external dependencies. Frontend control is limited to restrained resource hints in `index.html` plus the basemap fade-in / loading veil that masks the wait.

## Out of scope here

- Backend response compression, `Cache-Control` strategy, and SSR edge caching live in the Spring repo and the deployment platform (Vercel / Railway). They are flagged in the "Backend / deployment blockers" section above because they directly affect the frontend, but they are not modifiable from `projects/web`.
- Lighthouse / WebPageTest field metrics. The audit referenced above is HTTP-trace-based and code-based, not field-measured.

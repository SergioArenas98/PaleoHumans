# 08 — Timeline Page UX/UI Audit & Redesign Proposal

> **Status: investigation + proposal only. No source code has been changed.**
> Scope is `projects/web/src/app/pages/timeline-page/**` and adjacent frontend
> docs. This document does not touch backend code, the DB schema, the API
> response shapes, or the backoffice. It proposes work; it does not perform it.

Audit date: 2026-06-29. Verified against running stack (backend `:8080` → `200`,
web dev server `:4200` → `200`) plus a static read of the prerendered output in
`frontend/dist/web/browser/timeline/index.html`.

---

## 1. Executive summary

The `/timeline` page renders a single Gantt-style scatter chart: one fixed
left column of culture rows, and a horizontally scrollable right track with
phase headers, a cal-BP axis, neutral culture bands, and ~270 site dots sized by
MNI. It is functional on a wide desktop, but its information architecture does
not scale down, and several core encodings are either broken or misleading.

The five most consequential findings:

1. **The time axis domain is narrower than the data.** The axis is hard-coded to
   `45,300 → 11,700 cal BP`, but real cultures extend older (e.g. *Neronian*
   57k–52k, *Early Upper Palaeolithic* 55k–35k). `pct()` clamps everything to
   `[0,100]`, so cultures older than 45,300 collapse against the left edge —
   *Neronian* becomes a zero-width band with a dot stuck on the frame. The chart
   silently misrepresents part of the record. **(Correctness)**

2. **Mobile hides the only labels.** At ≤640 px the left culture column is
   `display:none`, leaving anonymous rows of dots with no way to tell which
   culture each row is. The page is effectively unusable below tablet width.
   **(Mobile / a11y)**

3. **Dot size barely encodes MNI.** `r = max(4, min(12, √mni·1.5))`. Every site
   with MNI ≤ 7 floors to an 8 px dot, and everything ≥ 64 caps at 24 px. The
   bulk of the record (low MNI) is visually identical, so the legend "Dot size =
   MNI" promises a signal the chart does not deliver. **(Encoding)**

4. **Double scrolling + extreme height.** The track is ≥ ~1,340 px tall (21
   rows × 64 px) before the full culture-card reference section beneath it, while
   the chart itself scrolls horizontally. Users scroll the page vertically *and*
   the track horizontally, and the axis/phase headers are **not sticky**, so
   orientation is lost on both axes. **(Layout)**

5. **Filters under-deliver.** The culture multi-select **closes after every
   pick** (so multi-selecting means reopening the dropdown N times); there is
   **no reset/clear-all button** in the live template (the `clearFilters()`
   method exists but is unwired); and there is **no results count** ("showing X
   of Y sites"). Search has **no debounce**. **(Discoverability)**

Accessibility is the weakest dimension: ~270 sequential dot buttons in the tab
order with no skip, 8 px hit targets (well under the 24 px WCAG 2.5.8 minimum),
infinite animations with **no `prefers-reduced-motion` guard**, 9 px low-contrast
axis labels, no focus management on the detail panel, and no Esc-to-close.

The good news: the page is **SSR-safe today** (no direct `window`/`document`
use; the timeline is genuinely prerendered with data) and **already neutral**
(no per-culture colour). A redesign can be **almost entirely frontend-only**
using the two existing endpoints. The recommended MVP keeps the desktop Gantt
metaphor but (a) fits the axis to the data, (b) makes phase headers + axis
sticky, (c) adds an overview navigator, (d) fixes the filters, legend, and
reset, and (e) replaces the mobile chart with a culture-first list using the
existing `TemporalPositionBarComponent`.

---

## 2. Current implementation map (file paths)

| Concern | File |
|---|---|
| Component logic | `frontend/projects/web/src/app/pages/timeline-page/timeline-page.ts` |
| Template | `frontend/projects/web/src/app/pages/timeline-page/timeline-page.html` |
| Styles | `frontend/projects/web/src/app/pages/timeline-page/timeline-page.css` |
| Primary data model | `frontend/projects/web/src/app/features/stats/model/MapTimelineStats.ts` |
| Primary data service | `frontend/projects/web/src/app/features/stats/services/map-timeline-stats-service.ts` |
| Enrichment model | `frontend/projects/web/src/app/features/cultures/model/Culture.ts` |
| Enrichment service | `frontend/projects/web/src/app/features/cultures/services/culture-service.ts` |
| Phase enum + labels | `frontend/projects/web/src/app/features/cultures/model/CulturePhase.ts` |
| Loading helper (SSR-safe) | `frontend/projects/web/src/app/utils/delayed-loading-state.ts` |
| **Reuse candidate** (mini bar) | `frontend/projects/web/src/app/components/temporal-position-bar/temporal-position-bar.ts` |
| Route registration | `frontend/projects/web/src/app/app.routes.ts` (`/timeline`) |
| Render mode | `frontend/projects/web/src/app/app.routes.server.ts` (`RenderMode.Prerender`) |
| Design tokens | `frontend/projects/web/src/styles/dark-theme.css` (`--dt-*`) |
| **No test exists** | `timeline-page.spec.ts` — *absent* |

Key constants in `timeline-page.ts`:

- `TL_START = 45300`, `TL_END = 11700`, `TL_SPAN = 33600` — the fixed axis domain.
- `PHASE_FULL_START = 34500`, `PHASE_FINAL_START = 20200` — phase boundaries.
- `BAND_H = 64` — per-culture row height.
- `BASE_WIDTH = 840`, `ZOOM_MIN = 1`, `ZOOM_MAX = 5`, `ZOOM_STEP = 0.5` — track
  width = `840 × zoom`. (The 100–400 % in the spec is now 100–500 % in code.)
- `NEUTRAL_BAND_RGB = '158, 154, 148'` — single neutral stone for every band/dot.
- `pct(bp)` clamps to `[0,100]`; `h(n,s)` is a deterministic pseudo-random
  spreader for dot jitter.

---

## 3. Current data flow

### 3.1 What arrives from `GET /api/stats/map-timeline`

`MapTimelineStats = { sites[], units[], cultures[] }`:

- **`sites[]`** (`MapTimelineSite`): `siteId, siteName, country, region,
  municipality, latitude, longitude, totalMni, individualCount, boneCount,
  skeletonCount, datedSampleCount, dominantCultureId, cultureIds[]`.
- **`units[]`** (`MapTimelineUnit`, keyed by `archaeologicalContextId`):
  `archaeologicalContextId, siteId, cultureId, individualCount, mniTotal,
  boneCount, skeletonCount, datedSampleCount`.
- **`cultures[]`** (`MapTimelineCulture`): `cultureId, cultureName, phase,
  startBp, endBp`.

This single payload is **the source of truth** for cultures, sites, bands, and
dots. It is `shareReplay`-cached in the service (shared with `/map`).

### 3.2 What arrives from `GET /api/cultures`

`Culture[]` with `region, description, features[]`. Used **only** by
`enrichCultureDetails()` to fill the secondary culture cards. It is a
*non-blocking* second fetch fired after the timeline is already built; its
failure is intentionally swallowed.

### 3.3 How the component derives its view

1. `buildCultures(stats.cultures)` → `CultureDef[]`, sorted oldest-first by
   `startBP`. Each culture gets `phase` (via `mapPhase`) and `{leftPct, widthPct}`
   (via `cultureBounds` → `pct`).
2. `buildData(stats)`:
   - Indexes sites by id; iterates `units`, **skipping** units whose `cultureId`
     is null or not in `cultureMap`.
   - Builds `siteMap: Map<siteId, SiteDetail>` with one `entries[]` per distinct
     `archaeologicalContextId`.
   - Emits one **dot per (site × culture)**: `mni` summed over that culture's
     entries; `r` from the MNI formula; `leftPct` placed *inside the culture
     band* with `h()` jitter; `topPx` a vertical `h()` jitter within the 64 px row.
   - `allSiteCountMap` = distinct sites per culture.
   - `pruneCulturesWithoutSites()` drops cultures with 0 sites.
   - Derives the sorted `countries[]` for the country filter.
3. `applyFilters()` filters `allDots` by culture set, country, and search
   (site/country substring), rebuilds `dotsByCulture`, recomputes `siteCountMap`,
   and `refreshPhaseGroups()` for the cards.
4. `enrichCultureDetails()` later patches `region/description/features` onto the
   existing `CultureDef[]`.

### 3.4 How filtering affects the view

- **Site dots:** `filteredDots` drives `dotsByCulture` → only matching dots render.
- **Culture rows / bands:** governed by `isCultureActive(key)` = *no selection*
  **or** key is selected. So with an active culture selection, non-selected rows
  are hidden entirely (both the left label and the band).
- **Left-column site count:** `getSiteCount(key)` reads the **filtered**
  `siteCountMap`, so it reflects the active country/search.
- **Culture cards:** `visibleCulturesForCards()` mirrors the selection.

### 3.5 How zoom affects layout

`trackWidthPx = round(840 × zoom)`; only the track's `min-width` changes. Bands
and dots are percentage-positioned, so zoom is a pure horizontal stretch. It
does **not** change row height, density, or what is in view — it only lengthens
the horizontal scroll.

### 3.6 Redundant / avoidable computation

- **`getBandBg(c)`** builds a gradient string for every band on every change
  detection pass (called in the `[style.background]` binding). The value is
  static per culture and could be precomputed once.
- **`isCultureActive(c.key)`** is evaluated twice per culture per CD (labels
  column + band loop).
- **No `trackBy` issue**, but ~270 dot buttons each carry 6 inline style
  bindings re-evaluated whenever `markForCheck()` fires (e.g. selecting a dot
  re-evaluates all dots). `selectDot()` triggers a full template re-eval.
- **Search has no debounce:** every keystroke runs the full `applyFilters()`
  pipeline + map rebuilds + CD.
- `getTotalSiteCount()` / the legacy `tl-detail` block / `clearFilters()` /
  several CSS classes are **dead code** (see §10).

---

## 4. Visual evaluation (screenshots)

Captured with headless Chrome against the live dev server. **Note:** the
captures render in the **light theme** (headless defaults to
`prefers-color-scheme: light`), which itself is a useful finding — the timeline
must hold up in *both* DEEP TIME (dark) and the light override, and several
contrast problems are worse in light.

> Captured states: **default** at 1440×900, 1024×768, 390×844.
> *Not* captured programmatically (no Playwright/driver installed; one-shot
> headless cannot script clicks): filtered, zoomed, panel-open, empty, and error
> states. These are analysed below from the template/CSS and the data model.
> If live captures of the interactive states are required, the cleanest path is
> the `e2e-runner` agent (Playwright) — see §13.

### 4.1 Desktop 1440×900 (default)
- Controls row: **Search · All countries · All cultures** only — no reset, no
  result count.
- Legend ("Dot size = MNI", "Click a dot for details") + zoom control on one
  hairline-bordered row.
- Left labels column (150 px): culture name, BP range, site count.
- Right track: 3 phase headers, axis ticks 45k…12k BP, neutral bands + dots.
- **Visible defects:** *Neronian* (57k–52k) collapses to a sliver with its dot
  pinned to the left frame; the phase-named catch-all rows (*Early/Full/Final
  Upper Palaeolithic*, *Uncertain 45k–12k, 4 sites*) render as huge bands that
  duplicate the phase headers; most dots are the same small size regardless of MNI.

### 4.2 Tablet 1024×768 (default)
- Header collapses to a hamburger.
- The 840 px track no longer fits; *Final Upper Palaeolithic* is clipped at the
  right and requires horizontal scroll. The fixed left column eats ~15 % of width.

### 4.3 Mobile 390×844 (default)
- Controls stack full-width; legend + zoom stack.
- **The left labels column is gone** (`display:none` ≤640 px). Rows of dots have
  no culture identity. Only ~46 % of the time span is visible (track min-width
  840 in a 390 viewport); the rest needs horizontal scroll. This is the single
  worst experience on the page.

---

## 5. Main UX/UI problems

| # | Problem | Evidence | Severity |
|---|---|---|---|
| P1 | Axis domain (45,300–11,700) narrower than data; older cultures clamp/collapse | `TL_START/TL_END`, `pct()` clamp; *Neronian* sliver in screenshot | **Critical** |
| P2 | Mobile drops all culture labels | `@media (max-width:640px){ .tl-labels-col{display:none} }` | **Critical** |
| P3 | Dot-size→MNI encoding flat for MNI ≤ 7, capped at ≥ 64 | `r = max(4,min(12,√mni·1.5))` | High |
| P4 | Double scroll + non-sticky axis/phase headers; very tall page | track ≥1,340 px; axis inside scroll-inner, page scrolls past it | High |
| P5 | Culture multi-select closes after each pick | `selectCulture()` sets `cultureDropdownOpen=false` | High |
| P6 | No reset/clear-all in UI; no results count | template has no reset button / no `.tlf-meta`; `clearFilters()` unwired | High |
| P7 | Phase-named catch-all cultures create giant duplicate bands | data has cultures named "…Upper Palaeolithic"/"Uncertain"; bands span the axis | Medium |
| P8 | Dot vertical/horizontal position is meaningless jitter; dots overlap and occlude each other | `h()` spread + `topPx` jitter, no collision avoidance | Medium |
| P9 | No hover/focus tooltip with MNI/BP/culture/country | only native `title="name · country"` | Medium |
| P10 | Zoom only stretches width; no fit-to-data, no overview/minimap, no jump | `trackWidthPx` is the only effect | Medium |
| P11 | Search not debounced | `(ngModelChange)="onFilterChange()"` → `applyFilters()` | Low |
| P12 | Detail panel overlays the right of the track (`position:absolute; right:0`), covering data while open | `.tl-site-panel` | Medium |
| P13 | Secondary culture-card section duplicates the whole culture list again, doubling page length | `.tl-cultures-section` repeats every culture | Low |
| P14 | `TemporalPositionBarComponent` uses a *different* axis (40k–10k) than the timeline (45.3k–11.7k) | inconsistent chronological scale across the app | Low |

---

## 6. Accessibility findings

| # | Finding | WCAG ref | Severity |
|---|---|---|---|
| A1 | ~270 dot `<button>`s in sequential tab order, no skip / no grouping; keyboard users must tab through every dot | 2.4.3, 2.1.1 | **Critical** |
| A2 | Dot hit targets 8 px (MNI ≤ 7) — far below 24 px; overlapping dots compound it | 2.5.8 (AA) / 2.5.5 (AAA) | **Critical** |
| A3 | Infinite/entry animations (`ring-pulse`, `tl-panel-slide-in`, `slide-up`, `spin`) with **no `prefers-reduced-motion`** in `timeline-page.css` | 2.3.3 | High |
| A4 | 9 px axis labels / 9 px meta in `--dt-text-3` (alpha 0.35) — fails contrast & minimum size, worse in light theme | 1.4.3, 1.4.11 | High |
| A5 | Detail panel: focus is not moved into it on open, not restored on close; no focus trap; **no Esc-to-close** (no keydown handler) | 2.4.3, 2.1.2 | High |
| A6 | Timeline conveys chronology only visually; axis/phases are `aria-hidden`. Screen-reader users get a flat list of "Open site detail: X" buttons with no temporal/culture context, and on mobile no labels at all | 1.3.1 | High |
| A7 | Dot `aria-label` says "Open site detail" but the action opens an *inline panel*, not navigation — label/role mismatch | 4.1.2 | Low |
| A8 | Neutral dots (stone on light cream / stone on near-black) are low-contrast against bands in both themes | 1.4.11 | Medium |
| A9 | Custom listbox dropdowns: items are `<button>`, not `role="option"` with roving focus/typeahead; `aria-selected` present but no arrow-key navigation | 4.1.2, 2.1.1 | Medium |

---

## 7. Performance findings

- **Initial payload:** the prerendered `/timeline/index.html` is ~372 KB with
  ~270 dot buttons inlined. Heavy first paint; acceptable but notable.
- **Change detection over ~270 dots:** each carries 6 inline style bindings;
  `selectDot()`/filter changes re-evaluate all of them. On mobile this risks
  jank. **Medium risk.**
- **`getBandBg()` rebuilt per CD** for every band — pure waste; precompute once.
- **No search debounce** — full filter + map rebuild per keystroke (small data,
  so low real-world cost, but trivially fixable).
- **Positive:** the `shareReplay` cache means the second visit (and the shared
  `/map` page) hits no network. Pruning + map indexing are O(n) one-time.
- **SSR:** no resolvers; data is fetched during prerender and survives via the
  transfer cache. No regression risk if we keep computations synchronous and
  browser-API-free.

---

## 8. Proposed improvements (ranked by impact ÷ effort)

Each item: **Problem · Benefit · Approach · Files · Complexity · Risk · FE-only?**

### Tier 1 — high impact, low/medium effort (the MVP)

**I1. Fit the axis to the data (or extend the fixed domain).**
- *Problem:* P1 — older cultures clamp to the frame.
- *Benefit:* every culture is positioned truthfully; the chart stops lying.
- *Approach:* derive `TL_START`/`TL_END` from `min(startBp)`/`max(endBp)` across
  `stats.cultures` (with a small padding and "nice" rounding for ticks), instead
  of hard-coding. Recompute `pct()`, phase boundaries stay as data values.
  Provide a "Fit to data" vs "Fit selected" toggle later (I12).
- *Files:* `timeline-page.ts` (constants → computed), minor `.html` (axis ticks
  generated from the domain).
- *Complexity:* Medium · *Risk:* Low · *FE-only:* Yes.

**I2. Make phase headers + axis sticky.**
- *Problem:* P4 — orientation lost when scrolling.
- *Benefit:* the scale is always visible while scanning rows and panning.
- *Approach:* `position: sticky; top: 0` on the phase/axis block within the
  scroll container; keep them inside the horizontally-scrolled inner so they
  track the pan; ensure they also stick vertically relative to the page. Pure
  CSS — **no scroll listeners, no JS, SSR-safe.**
- *Files:* `timeline-page.css` (+ minor DOM grouping in `.html`).
- *Complexity:* Medium · *Risk:* Low · *FE-only:* Yes.

**I3. Fix the culture multi-select.**
- *Problem:* P5 — closes after each pick.
- *Benefit:* real multi-select; far fewer clicks.
- *Approach:* in `selectCulture()`, stop setting `cultureDropdownOpen=false`;
  keep the panel open, toggle a checkmark, close only on backdrop/Esc. Add
  arrow-key roving focus + typeahead and `role="option"` (also fixes A9).
- *Files:* `timeline-page.ts`, `timeline-page.html`.
- *Complexity:* Low · *Risk:* Low · *FE-only:* Yes.

**I4. Add reset + results count (and wire the existing `clearFilters()`).**
- *Problem:* P6 — no reset, no feedback.
- *Benefit:* discoverable recovery + orientation ("Showing 84 of 248 sites").
- *Approach:* add a "Reset" button to the controls bound to `clearFilters()`;
  add a meta line using existing getters `filteredSiteCount` / `totalSiteCount`
  / `hasActiveFilters`. Reuse the `.tlf-meta` styles already in the CSS.
- *Files:* `timeline-page.html` (+ tiny `.ts` if a label getter helps).
- *Complexity:* Low · *Risk:* Low · *FE-only:* Yes.

**I5. Honest, useful legend + reduced-motion + bigger hit targets.**
- *Problem:* P3, A2, A3, A4.
- *Benefit:* the legend matches reality; motion-sensitive users are safe; dots
  are tappable.
- *Approach:* (a) widen the dot scale and label the legend with real anchors
  (e.g. "MNI 1 · 10 · 50+") or switch to graduated buckets; raise the minimum
  dot to ~12 px and add a transparent ≥24 px hit area via padding/`::before`.
  (b) Wrap all animations in `@media (prefers-reduced-motion: reduce){…}` and
  disable `ring-pulse`/slide/spin. (c) Bump axis labels off `--dt-text-3` to a
  readable token/size.
- *Files:* `timeline-page.css`, small `.ts` (dot radius formula).
- *Complexity:* Low · *Risk:* Low · *FE-only:* Yes.

**I6. Hover/focus tooltip with site · culture · BP range · MNI · country.**
- *Problem:* P9.
- *Benefit:* identify a point without opening the panel; works on keyboard focus.
- *Approach:* a small absolutely-positioned tooltip element driven by the
  hovered/focused dot (CSS-positioned from the dot's `leftPct/topPx`); content
  from the existing `Dot`/`SiteDetail`. No portal, no library.
- *Files:* `timeline-page.html`, `.css`, small `.ts` state.
- *Complexity:* Medium · *Risk:* Low · *FE-only:* Yes.

**I7. Mobile: culture-first list with mini temporal bars.**
- *Problem:* P2 (and A1/A2 on mobile).
- *Benefit:* a genuinely usable mobile view instead of a labelless scatter.
- *Approach:* below a breakpoint, render a vertical list of culture rows: name,
  BP range, phase, site count, and a **`TemporalPositionBarComponent`** mini-bar
  (reuse the existing component; reconcile its axis to the page domain — see
  I14). Tapping a culture expands its sites (name, country, MNI) as a list, each
  linking to `/sites/:id`. The big horizontal chart is hidden on mobile.
- *Files:* `timeline-page.html`, `.css`, `.ts`; reuse
  `components/temporal-position-bar/*`.
- *Complexity:* Medium · *Risk:* Low · *FE-only:* Yes.

### Tier 2 — high impact, higher effort

**I8. Overview navigator (mini-map of time) above the track.**
- *Problem:* P10 — zoom doesn't help you find things.
- *Benefit:* see the whole span at a glance; drag a viewport window to pan/jump.
- *Approach:* a thin full-width strip showing all cultures as tick marks across
  the full domain, with a draggable "current window" rectangle that maps to the
  track's `scrollLeft`. Browser-only interaction — **must guard with
  `isPlatformBrowser`** and only attach listeners after hydration.
- *Files:* `timeline-page.html`, `.css`, `.ts` (+ maybe a small child component).
- *Complexity:* High · *Risk:* Medium (SSR/scroll sync) · *FE-only:* Yes.

**I9. Density modes: Overview / Standard / Detailed.**
- *Problem:* P4/P8 — one fixed 64 px row, no way to compress.
- *Benefit:* fit more cultures on screen; reduce scroll.
- *Approach:* a segmented control switching `BAND_H` (e.g. 28 / 64 / 96) and dot
  rendering (Overview = collapsed strip per culture; Detailed = current).
- *Files:* `timeline-page.ts`, `.css`, `.html`.
- *Complexity:* Medium · *Risk:* Low · *FE-only:* Yes.

**I10. Cluster overlapping dots.**
- *Problem:* P8 — dots overlap and occlude (some unclickable).
- *Benefit:* legible density; clickable clusters that expand.
- *Approach:* bucket dots per culture row by x-proximity; render a cluster
  marker with a count when N dots fall within a pixel threshold at the current
  zoom; expand on click/focus into a small list. Recompute buckets on zoom.
- *Files:* `timeline-page.ts` (bucketing), `.html`, `.css`.
- *Complexity:* High · *Risk:* Medium · *FE-only:* Yes.

**I11. Sortable culture list + quick filters.**
- *Problem:* finding a specific culture; no fast slices.
- *Benefit:* "oldest first / newest / most sites / most MNI"; one-tap
  "Early UP / Full UP / Final UP / high-MNI / dated only".
- *Approach:* sort `cultures` by the chosen key; quick-filter chips set the
  culture selection / a derived predicate. "Dated only" uses
  `datedSampleCount > 0` which **is present** in the payload (no backend work).
  Phase chips reuse `mapPhase`.
- *Files:* `timeline-page.ts`, `.html`.
- *Complexity:* Medium · *Risk:* Low · *FE-only:* Yes.

**I12. Fit-to-data / Fit-selected / Reset view controls.**
- *Problem:* P1/P10 — no way to frame the data or a selection.
- *Benefit:* recompute the visible domain to the data or the selected cultures.
- *Approach:* extends I1; "Fit selected" sets the domain to the min/max BP of the
  selected culture set and resets scroll/zoom.
- *Files:* `timeline-page.ts`, `.html`.
- *Complexity:* Medium · *Risk:* Low · *FE-only:* Yes.

### Tier 3 — polish / contextual

**I13. Reframe the detail panel as contextual reading, not a competing sidebar.**
- *Problem:* P12 — the panel overlays the right of the track, hiding data.
- *Benefit:* read context without losing the chart; better focus management.
- *Approach:* on desktop, dock the panel *beside* the track (grid column) or as a
  bottom sheet that doesn't occlude the scatter; on open, move focus to the
  panel heading and trap focus; Esc + restore focus on close (fixes A5).
- *Files:* `timeline-page.html`, `.css`, `.ts`.
- *Complexity:* Medium · *Risk:* Low · *FE-only:* Yes.

**I14. Reconcile `TemporalPositionBarComponent` axis with the page domain.**
- *Problem:* P14 — two different chronological scales in the app.
- *Benefit:* consistent BP positioning everywhere it's used.
- *Approach:* make the component's axis configurable via inputs (default to the
  fitted domain) instead of the hard-coded 40k–10k.
- *Files:* `components/temporal-position-bar/temporal-position-bar.ts` (and any
  other consumer — verify usages first).
- *Complexity:* Low · *Risk:* Low (shared component — check consumers) · *FE-only:* Yes.

**I15. Collapse cultures by phase (expandable phase groups) in the chart.**
- *Problem:* P7/P4 — phase-named catch-alls + length.
- *Benefit:* a tidy, scannable hierarchy; collapse noisy catch-alls.
- *Approach:* group rows under collapsible phase sections in the track (mirrors
  the cards section); optionally de-emphasise phase-named/"Uncertain" cultures.
- *Files:* `timeline-page.ts`, `.html`, `.css`.
- *Complexity:* Medium · *Risk:* Low · *FE-only:* Yes.

### Separate track (NOT part of the timeline redesign)

**I16. A dedicated `/cultures/:id` route.**
- *Status:* This is a **new route**, an existing known gap
  ([02-public-web.md](./02-public-web.md) "Known gaps"), not a Timeline change.
- *Benefit:* a real home for culture editorial content currently crammed into
  the cards section; the timeline could then **link** culture names there and
  drop the giant duplicate cards section (P13).
- *Approach:* new lazy route using `CultureService.getById`. Frontend-only.
- *Recommendation:* propose separately; out of scope for the MVP here.

---

## 9. Recommended MVP redesign

Goal: fix correctness, orientation, filters, mobile, and a11y **without**
abandoning the desktop Gantt or introducing SSR/scroll-sync risk.

Include: **I1, I2, I3, I4, I5, I6, I7** (and the focus/Esc parts of I13 for the
existing panel; defer the docking layout).

Resulting experience:
- Desktop: a Gantt whose axis fits the data, with sticky phase headers + axis, a
  working multi-select, a reset + "showing X of Y" line, an honest legend with
  reduced-motion support and tappable dots, and hover/focus tooltips.
- Mobile: a culture-first list with mini temporal bars (reusing
  `TemporalPositionBarComponent`) and expandable site lists — no labelless
  scatter, no forced horizontal chart.
- A11y: reduced-motion respected, ≥24 px hit targets, Esc-to-close + focus
  management on the panel.

Deliberately deferred to phase 2: overview navigator (I8), clustering (I10),
density modes (I9), sortable/quick filters (I11), fit-selected (I12), panel
docking (I13 layout), phase collapsing (I15), `/cultures/:id` (I16).

---

## 10. Optional second-phase enhancements

I8 overview navigator · I9 density modes · I10 clustering · I11 sort + quick
filters · I12 fit-to-data/selected/reset-view · I13 panel docking · I14 shared
mini-bar axis · I15 phase collapsing · I16 `/cultures/:id` (new route, separate
approval).

---

## 11. Changes explicitly NOT recommended

- **Reintroducing per-culture colour** — prohibited by DEEP TIME; differentiate
  by text/structure/ordering/density only.
- **Reconstructing the timeline client-side from `/api/archaeological-contexts`**
  or any endpoint other than `/api/stats/map-timeline` — violates the data rule.
- **Backend/API/schema/backoffice changes** — out of scope; everything in the
  MVP is achievable with the two existing endpoints.
- **A charting library (D3/Chart.js) for the main track** — adds bundle weight
  and SSR/hydration risk for a layout that CSS percentage positioning already
  handles. (`/analysis` already owns Chart.js; don't spread it here.)
- **Encoding meaning into dot vertical position** — keep vertical jitter cosmetic
  or remove it; do not imply a second data dimension that doesn't exist.
- **Removing prerender / adding route resolvers** — keep `RenderMode.Prerender`
  and component-local loading; don't regress SSR.
- **Deleting the culture-cards section before `/cultures/:id` exists** — it is the
  only home for that editorial content today.

---

## 12. Implementation plan for the MVP

Phased, each phase independently shippable and verifiable.

1. **Cleanup pass (no behaviour change).** Remove dead code: the
   `legacyDetailPanelEnabled` block + `.tl-detail*` CSS, unused
   `tl-cult-swatch`/`tl-cult-count`/`tl-axis__line`/`tl-phase-div`/
   `tl-phase-header__sub` classes, and wire/remove `clearFilters()`/
   `getTotalSiteCount()`. Add a `timeline-page.spec.ts` harness first (see §13).
2. **I1 — fit axis to data.** Convert `TL_START/TL_END/TL_SPAN` and `AXIS_TICKS`
   to values computed from `stats.cultures`; verify *Neronian* now positions
   correctly. Snapshot the geometry in a unit test.
3. **I5 — legend honesty + reduced-motion + hit targets.** CSS-only + dot-radius
   tweak. Add `prefers-reduced-motion` block.
4. **I3 + I4 — filters.** Keep dropdown open on multi-pick; add roving
   focus/`role=option`; add Reset button + results-count meta.
5. **I2 — sticky phase/axis.** CSS `position: sticky`; verify horizontal pan and
   vertical scroll keep the scale visible; check both themes.
6. **I6 — tooltip.** Hover + focus, keyboard-reachable, content from `Dot`.
7. **I13 (focus subset) — panel a11y.** Move focus on open, Esc + restore on
   close, focus trap.
8. **I7 — mobile culture-first list.** Reuse `TemporalPositionBarComponent`;
   hide the chart below the breakpoint; expandable site lists.
9. **Verify** across the matrix in §13; update
   [02-public-web.md](./02-public-web.md) `/timeline` row and
   [04-design-system.md](./04-design-system.md) if any pattern is added.

Estimated size: a focused but non-trivial PR (or 2–3 small PRs by phase). All
frontend-only.

---

## 13. Test plan

There is **no `timeline-page.spec.ts` today** — step 0 is to add one.

**Unit (Vitest, `ng test --project=web`):**
- `pct()` / axis fitting: oldest and youngest cultures map to 0 % / 100 % after
  I1; a culture older than the previous fixed start is no longer clamped.
- Dot radius mapping (post-I5): boundary MNI values produce distinct, tappable
  sizes; minimum ≥ chosen floor.
- `applyFilters()`: culture set, country, and search combine correctly;
  `filteredSiteCount`/`siteCountMap` match; selection survives a
  still-matching dot and clears on a no-longer-matching one.
- Multi-select (post-I3): selecting two cultures keeps both; dropdown stays open.
- `clearFilters()` resets all three filters and the panel.
- Phase grouping: a culture maps to the expected phase via `mapPhase`/chronology.

**Component / DOM:**
- Default render lists all cultures with site counts; results-count line present.
- Reduced-motion: with `prefers-reduced-motion`, no animation classes apply.
- Panel: opens with focus moved to the heading; Esc closes and restores focus.
- Mobile breakpoint: culture-first list renders, chart hidden, each culture
  expands to sites linking to `/sites/:id`.

**Accessibility:**
- Axe (or manual) on default + panel-open: hit-target size, contrast on axis
  labels/dots, listbox semantics, no `aria-hidden` focusable traps.
- Keyboard walkthrough: reach filters → cultures → a dot → open panel → Esc,
  without a mouse; confirm the dot tab-order is bounded/skippable.

**SSR / render:**
- `ng build --project=web` prerenders `/timeline` with data (no runtime errors);
  confirm no `window`/`document` access outside `isPlatformBrowser` guards.
- Hydration: no mismatch warnings; transfer cache reused (no refetch on load).

**Visual / E2E (defer to `e2e-runner`, Playwright):**
- Capture the interactive states this audit could not script (filtered, zoomed,
  panel-open, empty/no-results, error) at 1440×900 / 1024×768 / 390×844, in both
  themes. Add a no-results assertion (search with no matches) and an error-state
  assertion (stub a failed `/api/stats/map-timeline`).

---

## 14. Constraint compliance checklist

- [x] No backend/API/schema/backoffice changes proposed in the MVP.
- [x] `/api/stats/map-timeline` remains the source of truth; `/api/cultures`
      only enriches cards. No unfiltered `/api/archaeological-contexts`.
- [x] SSR preserved (`RenderMode.Prerender`, no resolvers); any browser-only
      work (I8 navigator, tooltip positioning) is flagged to guard with
      `isPlatformBrowser`.
- [x] DEEP TIME `--dt-*` tokens only; no new hard-coded chroma.
- [x] No per-culture colour reintroduced.
- [x] Accessibility centred (keyboard, focus, ARIA, reduced motion, contrast,
      zoom, hit targets, mobile).
- [x] No source files edited; nothing committed. This is a proposal awaiting
      approval.

---

## 15. Phase 1 implementation status (2026-06-29)

Implemented (frontend-only, inside `pages/timeline-page/**`):

- **I1 — Fit axis to data.** New pure module `timeline-axis.ts`
  (`computeAxisDomain`/`bpToPct`/`cultureBounds`/`computeAxisTicks`/
  `computePhaseDefs`/`dotRadius`). The component derives the visible window from
  `stats.cultures` (defensive fallback to the legacy 45,300–11,700 window). Verified
  in the real prerender: the axis now runs **58,000 → 10,000 cal BP** and the
  Neronian (57k–52k) renders as a real band with its dot inside it instead of
  clamped to the frame.
- **I2 — Sticky phase headers + axis.** The chart became a single bounded
  frozen-panes scroll viewport (`.tl-scroll-outer` `overflow:auto` +
  `max-height: clamp(420px,72vh,820px)`): phase row + axis are `position:sticky;
  top:0` (`.tl-track-head`), the left labels column is sticky `left:0`, the
  corner sticks to both. Pure CSS, SSR-safe.
- **I4 — Reset + result count.** Visible **Reset** button wired to the existing
  `clearFilters()` (disabled when no filter is active) and an `aria-live`
  "Showing X of Y sites" line.
- **I5 — Honest legend + reduced motion + hit targets.** Legend shows real
  graduated MNI anchors (1 · 10 · 50+) from the new `dotRadius`, plus an
  older→recent direction cue; a `prefers-reduced-motion` block disables the
  pulsing ring / panel slide / spinner / dot scaling; an invisible `::before`
  gives every dot a ≥24 px hit target. The MNI radius scale was reworked so low
  MNI (1 vs 10) is now visually distinguishable (the old formula floored
  everything ≤ 7 to one size).
- **Panel a11y subset.** Detail panel is now `role="dialog"`,
  `aria-labelledby`, `tabindex="-1"`; focus moves into it on open, **Escape**
  closes it, and focus returns to the triggering dot. All focus/`setTimeout`
  work is guarded by `isPlatformBrowser`.

Tests: `timeline-axis.spec.ts` (16) + `timeline-page.spec.ts` (7) added; full
web suite **111/111 green**. Build: `ng build --project=web` prerenders all 10
routes with no AOT errors. Smoke test: `/timeline` renders correctly at
1440/1024/390 with no console errors; `/map` (shared service) unaffected.

**Deferred to Phase 2** (per the approved scope): I3 culture multi-select that
stays open + keyboard nav; I6 hover/focus tooltips; I7 mobile culture-first
layout reusing `TemporalPositionBarComponent`. Phase 1 leaves the mobile
labels-hidden behaviour unchanged — I7 addresses it.

Committed as `feat(web): improve timeline axis and accessibility`.

---

## 16. Phase 2 implementation status (2026-06-29)

Implemented (frontend-only, inside `pages/timeline-page/**`):

- **I3 — Culture multi-select.** `selectCulture`/`selectAllCultures` no longer
  close the panel, so several cultures can be toggled in one pass. Options are
  `role="option"` with `aria-selected`; a shared listbox keyboard handler adds
  Arrow/Home/End navigation and **Escape** (closes + returns focus to the
  trigger); `ArrowDown` on the trigger opens and moves focus into the list. The
  same handler is wired to the country panel. Visual language and `--dt-*`
  tokens unchanged.
- **I6 — Hover/focus tooltips.** A single `.tl-tooltip` shows site name, culture,
  culture BP range, MNI and country on `mouseenter`/`focus` (cleared on
  `mouseleave`/`blur`), so it works for mouse *and* keyboard. Position is derived
  from a precomputed `cultureRowIndex` (no `getBoundingClientRect`, no layout
  thrash); the tooltip is `aria-hidden` (dots already carry a full `aria-label`).
  SSR-safe: only rendered when `hoveredDot` is set, which never happens during
  prerender.
- **I7 — Mobile culture-first layout.** Below 640 px the desktop chart and its
  chart-only controls (zoom, MNI/direction legend) are hidden and a
  `.tl-mobile` section takes over: one card per visible culture with the
  **culture name**, phase, BP range, site count, a **mini temporal bar**, and a
  tappable list of its sites (country · MNI) that opens the detail panel (now a
  viewport-fixed bottom sheet). `TemporalPositionBarComponent` is **not**
  reused — its hard-coded 40k–10k axis conflicts with the fitted ~58k–10k
  domain and it is a shared component; instead the mini-bars render directly
  from `CultureDef.leftPct/widthPct` (the same `timeline-axis` math), so there
  is exactly one consistent axis and no `pages → components` dependency.

Tests: `timeline-page.spec.ts` grew to 15 (multi-select stays open, option
roles, ArrowDown focus move, Escape + focus return, tooltip content + position,
mobile cards + mini-bar on fitted axis, mobile site row opens panel). Full web
suite **119/119 green**. Build: `ng build --project=web` prerenders all 10
routes, no AOT errors. Smoke test: mobile renders the culture-first list with
visible names + fitted mini-bars; desktop/tablet rest state unchanged; `/map`
(shared service) unaffected; no console errors.

Not committed yet (awaiting review). To be committed as a separate logical
change covering only `timeline-page.{ts,html,css,spec.ts}` + this doc.
```

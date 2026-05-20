# 07 — Design System

The two applications use **two separate, isolated design systems**. They do not share tokens, components, or fonts.

| App | Token namespace | Theme | Source files |
|---|---|---|---|
| `projects/web` | `--dt-*` | DEEP TIME (dark) + Stone Journal (light overrides) | `projects/web/src/styles.css`, `projects/web/src/styles/dark-theme.css`, `projects/web/src/styles/light-theme.css` |
| `projects/backoffice` | `--ph-*`, `--t-*` | Editorial light, parchment + ochre | `projects/backoffice/src/styles.css` |

> **Never mix tokens across the two apps.** The redesign of the public web targets `--dt-*` only; the backoffice keeps `--ph-*` / `--t-*` untouched.

---

## Public web — DEEP TIME

### Design intent

DEEP TIME is the active visual direction for `projects/web`. The principles, in order of priority:

1. **Darkness as fidelity.** The near-void background (`--dt-bg: #080808`) is not decorative dark mode; it is a statement that the documented humans lived 11,700–45,300 years before electricity. The dark environment is the correct backdrop for the data.
2. **Culture colors as the only chromatic vocabulary.** Roughly 19 archaeological culture colors (from each `Culture.color` and `Culture.colorRgb`) are the exclusive source of chrominance. UI surfaces, navigation, and controls are neutral. New chromatic decisions must resolve to either a neutral token or `var(--culture-color)`.
3. **Data is a precision instrument.** Numeric and identifier data uses a monospaced font (the codebase declares `JetBrains Mono` as the intended `--dt-font-data`, though `dark-theme.css` currently aliases all four font slots to `Space Grotesk`; see "Font reality" below).
4. **Temporal depth is always present.** Where possible, pages surface chronological orientation: KPI strips, culture chips, and the dedicated `/timeline` view.
5. **Each page has one job.** Page composition follows the page's communication purpose rather than a homogeneous template.
6. **Seriousness through precision, not decoration.** Exact dates (`12,345 ± 120 BP (uncal)`), exact citations, exact terminology. No academic ornamentation.

### Token map (excerpt)

Full declaration: `projects/web/src/styles/dark-theme.css`.

| Token | Value | Purpose |
|---|---|---|
| `--dt-bg` | `#080808` | Page background, void |
| `--dt-surface-1` | `#111111` | Cards, nav bar, header |
| `--dt-surface-2` | `#1A1A1A` | Elevated cards, code blocks |
| `--dt-surface-3` | `#222222` | Inputs, select backgrounds |
| `--dt-surface-4` | `#2A2A2A` | Hover state for surface-3 |
| `--dt-glass` | `rgba(8,8,8,0.75)` | Floating panels |
| `--dt-glass-blur` | `16px` | `backdrop-filter` value |
| `--dt-border` | `rgba(255,255,255,0.06)` | Default border |
| `--dt-border-strong` | `rgba(255,255,255,0.14)` | Hover / active |
| `--dt-border-focus` | `rgba(255,255,255,0.35)` | Focus ring |
| `--dt-text` | `#F0EDE6` | Primary text |
| `--dt-text-2` | `#9E9A94` | Secondary text |
| `--dt-text-3` | `rgba(240,237,230,0.35)` | Tertiary / decorative |
| `--dt-text-data` | `#E8E4DC` | Numeric / monospaced data |
| `--dt-action` | `#E8E4DC` | Primary button text |
| `--dt-action-hover` | `#FFFFFF` | Hover |
| `--dt-action-muted` | `rgba(240,237,230,0.55)` | Disabled / ghost |
| `--dt-error` / `--dt-error-bg` | `#FF6B6B` / `rgba(255,107,107,0.08)` | Error |
| `--dt-success` / `--dt-success-bg` | `#4CAF82` / `rgba(76,175,130,0.08)` | Success |
| `--dt-warning` / `--dt-warning-bg` | `#F5A623` / `rgba(245,166,35,0.08)` | Warning |

### Culture color binding

Culture color must come from the backend, set per component via inline CSS variables:

```html
<div [style.--culture-color]="culture.color"
     [style.--culture-rgb]="culture.colorRgb"
     class="ctx-card">
  ...
</div>
```

Derived tokens then resolve correctly:

```css
--dt-culture-color:       var(--culture-color, #888888);
--dt-culture-wash:        rgba(var(--culture-rgb, 136,136,136), 0.06);
--dt-culture-wash-strong: rgba(var(--culture-rgb, 136,136,136), 0.25);
--dt-culture-glow:        rgba(var(--culture-rgb, 136,136,136), 0.20);
```

Markers on the public Map and the SiteDetailsPage mini-map are **neutral** (near-black outer disc, muted stone centre). Culture color is reserved for chips, strips, and timeline accents — never the marker itself.

### Typography

Intended font roles (as specified in the chosen-direction document):

- `--dt-font-display`: Space Grotesk
- `--dt-font-body`: Inter
- `--dt-font-data`: JetBrains Mono

**Font reality (verified 2026-05).** `projects/web/src/styles/dark-theme.css` currently aliases all four `--dt-font-*` slots to `Space Grotesk`:

```css
--dt-font-header:  'Space Grotesk', system-ui, sans-serif;
--dt-font-display: var(--dt-font-header);
--dt-font-body:    var(--dt-font-header);
--dt-font-data:    var(--dt-font-header);
```

`projects/web/src/index.html` loads Space Grotesk via `<link>`. Inter and JetBrains Mono are not currently loaded in production CSS. Future work to honor the original "data is precision" principle would (a) load Inter + JetBrains Mono, (b) un-alias `--dt-font-body` and `--dt-font-data`, (c) audit numeric components to use `--dt-font-data`. Tracked in [10-roadmap.md](./10-roadmap.md).

### Layout primitives

These structural patterns recur in the public web and should be preserved across redesign work:

- **Numbered sections** (`01`, `02`, `03`) on long record pages (Site Detail, Individual Detail) for orientation.
- **KPI strip** under page heroes (e.g. Contexts / MNI / Individuals / Dated samples).
- **Sticky sidebar quick-facts** on long detail pages.
- **Culture chips** for quick culture identification, colored via `var(--dt-culture-color)`.
- **Mini-timeline strip** intended for record pages (declared as a DEEP TIME idea; partial implementation. `Requires code verification` for current state on each page).
- **Inline mini-map** on SiteDetailsPage, framed with a neutral DEEP TIME border + matting + hairline + drop shadow.

### Components and patterns

- Public shell: persistent header + footer; routed pages mount inside.
- Header: 7 primary items (Home, Sites, Individuals, Bones, Map, Timeline, Bibliography) + 3 muted items (Dataset, Methodology, About). Mobile drawer below 1024 px.
- Filter bars: chip-style multi-select, accessibility-first, keyboard-navigable.
- Tables: dense scientific tables scroll inside their own wrappers; long labels wrap rather than widening the page.
- Loaders: blocking page loaders gated by `DelayedLoadingState` (300 ms threshold, SSR-safe). Inline section loaders remain immediate.

### UX principles for the public web

- Show real values from real endpoints. Never fabricate (e.g. `lastUpdatedAt`).
- Show "—" or hide for unknown values; do not pretend zero.
- Preserve exact terminology and abbreviations from the original article (MNI, MNBR, cal BP, AMS, OSL, AAR).
- Always link records back to their bibliographic source.
- Avoid emoji and consumer-app icons. Material Symbols are used today but constrained; long-term direction is a more scholarly icon set.

---

## Backoffice — TERRAIN (light)

`projects/backoffice/src/styles.css` defines an editorial **light** theme for long admin sessions: parchment neutrals, ochre accents, near-black warm text. Token namespace is `--ph-*`.

| Token | Value | Purpose |
|---|---|---|
| `--ph-bg` | `#f4f1ea` | Soft parchment background |
| `--ph-bg-raised` | `#ffffff` | Cards / panels |
| `--ph-bg-sunken` | `#ece8df` | Input fields |
| `--ph-border` | `#d7d0c0` | Default border |
| `--ph-border-strong` | `#b9ae95` | Strong border |
| `--ph-focus-ring` | `#8c5d1f` | Focus ring (ochre) |
| `--ph-text` | `#2a2418` | Near-black, warm cast |
| `--ph-text-muted` | `#5a5343` | Muted text |
| `--ph-text-dim` | `#7a7261` | Dim text |
| (additional ochre accent tokens follow) | — | — |

The backoffice imports Leaflet CSS globally (`angular.json` config) because the admin app may render maps for editorial preview workflows. The public web does not import Leaflet CSS globally — see [02-architecture.md](./02-architecture.md).

---

## Archived design directions

The earlier `docs/paleohumans-design/` folder contained four design exploration documents (DEEP TIME, TERRAIN, STRATA, DEEP FIELD), a comparative review, a rebuild plan, a page-reinvention document, and the design-system spec. Their durable content has been absorbed into this document. Specifically:

- **Chosen direction:** DEEP TIME (this document).
- **Rejected directions:** STRATA (correct fallback if accessibility issues arise), DEEP FIELD (rejected for SSR risk and color-competition with culture palette).
- **Adapted direction:** TERRAIN structural patterns (numbered sections, KPI strips, sticky sidebars, culture chips, MNI bars) kept and restyled.

The exploration sources are no longer part of the active documentation. The implementation specification lives in:

- `projects/web/src/styles/dark-theme.css` — the canonical token file.
- `projects/web/src/styles.css` — global imports and shared component class hooks.
- Component-level CSS in each `pages/<page>/<page>.css` and `components/<component>/<component>.css`.

If a future redesign wants to revisit the rejected directions, recover them from git history rather than from this folder.

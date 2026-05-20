# 09 — Development Workflow

## Prerequisites

- **Node.js** matching the engines declared in `package.json` (see file). Use the same version as CI/CD where possible.
- **npm** (the repository's `package.json` declares `"packageManager": "npm"` via `angular.json`).
- A reachable Spring Boot backend on `http://127.0.0.1:8080/api` for local development, or a production URL configured in `environment.prod.ts` for builds intended for deploy.

## Install

```bash
npm install
```

## Commands

All commands run from the workspace root.

### Public web (`projects/web`)

```bash
ng serve --project=web              # http://localhost:4200 (dev)
ng build --project=web              # production build
ng test  --project=web              # Vitest unit tests
ng lint  --project=web              # if a lint target is configured
```

### Backoffice (`projects/backoffice`)

```bash
ng serve --project=backoffice       # http://localhost:4200 (dev, alternative project)
ng build --project=backoffice
ng test  --project=backoffice
```

### Generate

```bash
ng generate component my-component --project=web
ng generate component my-admin     --project=backoffice
```

## Environment configuration

Each project owns its `environment.ts` / `environment.prod.ts`:

| File | Purpose |
|---|---|
| `projects/web/src/app/environments/environment.ts` | Public web dev settings: `apiBaseUrl`. |
| `projects/web/src/app/environments/environment.prod.ts` | Public web production overrides. Replaced via `fileReplacements` in `angular.json`. |
| `projects/backoffice/src/app/environments/environment.ts` | Backoffice dev settings: `apiBaseUrl`, `sessionStorageKey`, etc. |
| `projects/backoffice/src/app/environments/environment.prod.ts` | Backoffice production overrides. |

**Do not hardcode secrets** anywhere. The public web basemaps are keyless; environment files should not contain browser-side map provider tokens.

## SSR safety

Any code that touches a browser-only API (`window`, `document`, `localStorage`, `IntersectionObserver`, Leaflet, dynamic `<script>` injection) must be guarded with `isPlatformBrowser(PLATFORM_ID)`. The build will compile but the SSR runtime will crash if a top-level call to `window` runs server-side.

Common patterns:

```ts
import { PLATFORM_ID, inject } from '@angular/core';
import { isPlatformBrowser } from '@angular/common';

private readonly platformId = inject(PLATFORM_ID);
private readonly isBrowser = isPlatformBrowser(this.platformId);

ngOnInit() {
  if (!this.isBrowser) return;
  // browser-only work here
}
```

Dynamic-import patterns for Leaflet should mirror what `MapPage` already does:

```ts
const L = await import('leaflet');
```

## Angular conventions

- **Standalone components only.** No NgModules.
- **`inject()` over constructor injection** in services and component fields.
- **Control flow:** `@if`, `@for`, `@switch`. (`/analysis` still uses `*ngIf`/`*ngFor`; do not propagate.)
- **OnPush change detection** where helpful, especially in the backoffice list pages and shared dialogs.
- **Per-component CSS** (no global CSS framework). The two apps each have their own design-token files; never reach across the `projects/web` ↔ `projects/backoffice` boundary.
- **Default exports forbidden.** All components and services are named exports.

## Naming conventions

- Files: `kebab-case.ts` (`site-details-page.ts`, `bone-service.ts`).
- Classes: `PascalCase` with a role suffix (`SiteDetailsPage`, `IndividualService`, `AdminAuthStore`).
- Routes: lowercase kebab-case (`archaeological-contexts`, `funerary-contexts`, `bone-catalog`).
- Backend response models: live under `features/<entity>/model/*.ts` with `PascalCase` interface names matching the entity (`Site`, `Individual`, `ArchaeologicalContext`).
- Service methods: `getById`, `search`, `getAll`, `getAllItems`, `create`, `update`, `delete`.

## Verification before declaring a task done

Run the smallest useful check first, broaden only if needed.

1. **Type check:** `ng build --project=<app>` — surfaces broken imports and missing fields.
2. **Targeted tests:** `ng test --project=<app>` (Vitest). Use focused tests during development.
3. **Manual smoke test in a browser** for any UI change — there is no good substitute for clicking through the affected route.
4. **SSR smoke test** for routes that render server-side (`/sites/:id`, `/individuals/:id`, `/analysis`).

For UI or frontend changes, **starting the dev server and exercising the feature in a browser** is mandatory before claiming completion. Type-check and tests verify code correctness, not feature correctness.

## Documentation update rules

When code changes affect documentation:

1. Update the **single source-of-truth document** for the topic. Cross-link from other documents that briefly mention the same fact.
2. Do not duplicate explanations. If two places need the same paragraph, the explanation lives in the more authoritative document, and the other links to it.
3. If a fact cannot be verified from the repository, mark it `Unknown`, `Unverified`, or `Requires code verification`.
4. Do not preserve contradictions. Replace the older version.
5. Do not create new top-level files under `/docs/` without explicit need. The numbered files cover the durable knowledge.

## AI agent working rules (specific to this repo)

- **Plan first** for tasks that touch architecture, multiple files, or public APIs.
- **Match the v2 model.** Use `ArchaeologicalContext`, `Individual`, `Bone`, `Skeleton`, `FuneraryContext`, `DatedSample`, `DatingResult`. Reject any new code that introduces `OsteologicalUnit`, `Specimen`, `BurialGroup`, or a top-level `Date` entity.
- **Never invent backend endpoints, fields, or filters.** If the code does not show it and the backend repository is not present, mark the assumption explicitly.
- **Public web and backoffice are separate.** Never import from the other app, even if the type "looks the same."
- **Never modify `docs/database/`** content except for renames clearly required for documentation clarity.
- **Run the dev server in a browser** before claiming a UI change is done.
- **Read the relevant numbered doc(s) before editing.** The numbered file is the source of truth.

## Where to look first

| Need | Path |
|---|---|
| Public route table | `projects/web/src/app/app.routes.ts` |
| Public SSR mode | `projects/web/src/app/app.routes.server.ts` |
| Public bootstrap | `projects/web/src/app/app.config.ts` |
| Public-page implementation | `projects/web/src/app/pages/<name>/` |
| Public domain models | `projects/web/src/app/features/<entity>/model/*.ts` |
| Public services | `projects/web/src/app/features/<entity>/services/*.ts` |
| Public design tokens | `projects/web/src/styles/dark-theme.css`, `projects/web/src/styles.css` |
| Admin route table | `projects/backoffice/src/app/app.routes.ts` |
| Admin auth | `projects/backoffice/src/app/core/auth/*` |
| Admin shared utilities | `projects/backoffice/src/app/features/shared/utils/*.ts` |
| Admin design tokens | `projects/backoffice/src/styles.css` |
| API contract | `docs/04-backend-api-contract.md` |
| Domain model | `docs/03-domain-model.md` |
| Architecture | `docs/02-architecture.md` |

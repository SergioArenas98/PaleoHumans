# Frontend Documentation

This folder is the knowledge base for the **PaleoHumans Angular workspace**
(`frontend/`) — the public web (`projects/web`) and the admin backoffice
(`projects/backoffice`). It is written for AI coding agents and human
contributors.

> **Scope.** This folder describes **only** the frontend workspace. Platform-wide
> topics — the project overview, system architecture, the scientific domain
> model, the database schema, and the REST API contract — are shared with the
> backend and live at the `docs/` root. See [../README.md](../README.md) for the
> full map.

## Shared (platform-wide) references

The frontend consumes the backend over HTTP; the backend lives in `backend/` of
this monorepo and its contract is documented at the root, not here:

| Topic | Document |
|---|---|
| What PaleoHumans is | [../01-project-overview.md](../01-project-overview.md) |
| How the components fit together | [../02-system-architecture.md](../02-system-architecture.md) |
| Scientific/domain model (v2 entities) | [../03-domain-model.md](../03-domain-model.md) |
| Database schema | [../04-database.md](../04-database.md) · raw DDL: [../database/schema.sql](../database/schema.sql) |
| REST API contract | [../05-api-contract.md](../05-api-contract.md) |
| Scientific source | [../references/Article.pdf](../references/Article.pdf) |

## Reading order (frontend workspace)

| # | Document | Purpose |
|---|---|---|
| 0 | [README.md](./README.md) | This file. Index and scope. |
| 1 | [01-frontend-architecture.md](./01-frontend-architecture.md) | Workspace layout, SSR, the two apps, auth boundaries, data flow. |
| 2 | [02-public-web.md](./02-public-web.md) | Public site pages, services, data loading, SSR rules. |
| 3 | [03-backoffice.md](./03-backoffice.md) | Admin app: routes, auth, CRUD, relation editing. |
| 4 | [04-design-system.md](./04-design-system.md) | Design tokens, typography, colour, layout patterns. |
| 5 | [05-performance.md](./05-performance.md) | Loading strategies and performance constraints. |
| 6 | [06-frontend-workflow.md](./06-frontend-workflow.md) | How to run, build, test; Angular conventions for agents. |
| 7 | [07-roadmap.md](./07-roadmap.md) | Frontend-specific gaps and speculative ideas. |

For source-code orientation, the canonical entry points are:

- `frontend/projects/web/src/app/app.routes.ts` — public web routes
- `frontend/projects/backoffice/src/app/app.routes.ts` — admin routes
- `frontend/projects/web/src/app/features/*` and
  `frontend/projects/backoffice/src/app/features/*` — domain models + services

## Source-of-truth rules

1. **Code wins.** If a doc says X and the code does Y, the code is correct —
   update the doc.
2. **The backend contract is fixed for the frontend.** The frontend cannot
   rename endpoints, fields, or filters; those changes originate in `backend/`.
   See [../05-api-contract.md](../05-api-contract.md).
3. **The v2 model is current.** Use `Site`, `ArchaeologicalContext`,
   `Individual`, `Bone`, `Skeleton`, `FuneraryContext`, `DatedSample`,
   `DatingResult`. The v1 names `OsteologicalUnit`, `Specimen`, `BurialGroup`,
   and a top-level `Date` are deprecated — see
   [../03-domain-model.md](../03-domain-model.md).
4. **Public web and backoffice are separate apps.** They share `node_modules`,
   not models. A change to one must not silently affect the other.

When a fact cannot be verified from the repository, the document marks it
`Unverified`, `Unknown`, or `Requires code verification`.

## Warnings

- **Do not edit the database schema from the frontend.** The schema is owned by
  the backend; changes originate there (see [../04-database.md](../04-database.md)).
- **Do not change application behaviour** while editing docs.
- **Do not invent endpoints, fields, filters, or routes** that are not in the
  code. Planned-but-unimplemented work goes in [07-roadmap.md](./07-roadmap.md).
- **Do not preserve contradictions.** Keep the resolved version; remove the
  conflicting one.

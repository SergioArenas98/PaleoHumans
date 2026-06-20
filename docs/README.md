# PaleoHumans Documentation

This is the canonical knowledge base for the **PaleoHumans monorepo** — the
Angular frontend (`frontend/`), the Spring Boot backend (`backend/`), and the
shared database and reference artifacts. It is written for AI coding agents and
human contributors who need a precise, non-ambiguous picture of the project
before changing anything.

## How this folder is organised

The documentation mirrors the monorepo: **platform-wide topics live at the
`docs/` root; workspace-specific topics live in `docs/frontend/` and
`docs/backend/`.**

```text
docs/
  README.md                     ← this index
  01-project-overview.md        ← what PaleoHumans is (platform-wide)
  02-system-architecture.md     ← the 3 components, how they connect, repo layout
  03-domain-model.md            ← canonical v2 scientific model (shared)
  04-database.md                ← canonical PostgreSQL schema reference (shared)
  05-api-contract.md            ← canonical REST contract (shared)
  06-local-development.md       ← run the whole stack locally
  07-deployment.md              ← deployment + environment variables
  08-roadmap.md                 ← cross-cutting roadmap + links to workspace roadmaps

  frontend/                     ← Angular workspace (web + backoffice) only
    README.md
    01-frontend-architecture.md
    02-public-web.md
    03-backoffice.md
    04-design-system.md
    05-performance.md
    06-frontend-workflow.md
    07-roadmap.md

  backend/                      ← Spring Boot service only
    README.md
    01-backend-architecture.md
    02-security-and-auth.md
    03-validation-and-errors.md
    04-services-and-business-rules.md
    05-performance-and-pagination.md
    06-backend-workflow.md
    07-roadmap.md

  database/
    schema.sql                  ← PostgreSQL schema reference copy
  references/
    Article.pdf                 ← Arenas del Amo et al. (2024), the scientific source
```

### What belongs in root `docs/`

Anything that describes the **whole platform** or a contract **shared** by the
frontend and backend: the project overview, the system architecture, the
scientific domain model, the database schema, the REST API contract, how to run
and deploy the stack, and the cross-cutting roadmap.

### What belongs in `docs/frontend/`

Anything that describes **only** the Angular workspace: the workspace
architecture and SSR model, the public web pages, the backoffice admin app, the
design systems, frontend performance, the Angular development workflow, and
frontend-specific gaps.

### What belongs in `docs/backend/`

Anything that describes **only** the Spring Boot service: the backend
architecture, security and authentication, validation and error handling,
services and business rules, backend performance and pagination, the backend
development workflow, and backend-specific gaps.

## Source of truth for common topics

| Topic | Canonical document |
|---|---|
| What the platform is | [01-project-overview.md](./01-project-overview.md) |
| How the components fit together | [02-system-architecture.md](./02-system-architecture.md) |
| Scientific/domain model (v2 entities) | [03-domain-model.md](./03-domain-model.md) |
| PostgreSQL schema (tables, enums, triggers) | [04-database.md](./04-database.md) |
| REST API (endpoints, filters, DTOs) | [05-api-contract.md](./05-api-contract.md) |
| Running the stack locally | [06-local-development.md](./06-local-development.md) |
| Deployment + environment variables | [07-deployment.md](./07-deployment.md) |
| Roadmap (cross-cutting) | [08-roadmap.md](./08-roadmap.md) |

The **raw database schema** reference copy lives at
[database/schema.sql](./database/schema.sql); the authoritative copy is
`backend/src/main/resources/db/database.sql` (see
[04-database.md](./04-database.md) for the relationship between the two). The
primary **scientific source** is [references/Article.pdf](./references/Article.pdf).

## Source-of-truth rules

When two documents disagree:

1. **Code wins.** If a doc says X and the code does Y, the code is correct —
   update the doc. The order of authority is: live backend response → SQL schema
   → frontend models → narrative docs.
2. **One canonical document per topic.** Common topics are owned by the root
   docs above; workspace docs link to them rather than restating them.
3. **The v2 model is current.** The entities are `Site`,
   `ArchaeologicalContext`, `Individual`, `Bone`, `Skeleton`,
   `FuneraryContext`, `DatedSample`, `DatingResult` (plus `Culture`,
   `BoneCatalog`, `Reference`, `DatingTechnique`). The v1 names
   `OsteologicalUnit`, `Specimen`, `BurialGroup`, and a top-level `Date` are
   deprecated and must not be reintroduced. See
   [03-domain-model.md](./03-domain-model.md).
4. **Public web and backoffice are separate apps.** They share `node_modules`,
   not models; a change to one must not silently affect the other.
5. When a fact cannot be verified from the repository, the document marks it
   `Unknown`, `Unverified`, or `Requires code verification`.

## What this docs folder is NOT

- It is not a changelog. Historical implementation logs were absorbed into stable
  sections and removed; the git history is the change log.
- It is not a duplicate of `CLAUDE.md` / `AGENTS.md`. Those provide quick agent
  rules; this folder provides depth.
- It does not invent endpoints, fields, filters, routes, commands, or database
  facts. Planned-but-unimplemented work is marked in the roadmaps.

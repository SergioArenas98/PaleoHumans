# 02 — System Architecture

> **Scope.** This is the platform-level view: the deployable components, how they
> connect, the monorepo layout, and the cross-cutting contracts that bind them.
> The internal architecture of each workspace lives in
> [frontend/01-frontend-architecture.md](./frontend/01-frontend-architecture.md)
> (Angular workspace, SSR, data flow) and
> [backend/01-backend-architecture.md](./backend/01-backend-architecture.md)
> (Spring layers, packages, request lifecycle).

## Components

The platform has three runtime components plus one datastore:

```text
[ Angular public web ]      [ Angular backoffice ]
  frontend/projects/web       frontend/projects/backoffice
          \                          /
           \  REST over HTTPS       / (JWT)
            \                      /
             v                    v
              [ paleohumans-api ]            backend/
                      |
                      v
               [ PostgreSQL ]
```

- The **public web** consumes anonymous `GET` endpoints to render the site
  catalogue, individuals, bones, dated samples, the map/timeline, and
  bibliography. It does not authenticate.
- The **backoffice** authenticates as `ROLE_ADMIN` to perform CRUD on every
  domain entity.
- The **backend** (`paleohumans-api`) is the only component with database
  access. It serves the public reads and the admin writes, owns all validation
  and business rules, and exposes the dataset export.
- The **database** (PostgreSQL) is owned by the backend. No frontend has direct
  database access; anything that looks like ad-hoc SQL on a frontend is a bug.

All three components live in this single repository (monorepo). Each workspace
is built with its own toolchain and is **deployed independently**, but they
share this repository and the `docs/` knowledge base.

## Repository structure

```text
Paleohumans_project/
├── frontend/                # Angular 21 workspace (public web + admin backoffice)
│   ├── angular.json         # project definitions for both apps
│   ├── package.json         # shared dependencies / npm scripts
│   ├── vercel.json          # frontend deployment config
│   └── projects/
│       ├── web/             # Public scientific website (SSR)
│       └── backoffice/      # Admin/curator app (JWT, /admin/*) (SSR)
├── backend/                 # Spring Boot 3.5.6 / Java 17 REST API
│   ├── pom.xml
│   ├── mvnw / mvnw.cmd      # Maven wrapper
│   ├── run-backend.ps1      # Windows convenience launcher
│   └── src/main/resources/db/
│       └── database.sql     # Authoritative PostgreSQL schema (applied manually)
├── docs/                    # Canonical knowledge base (this folder)
│   ├── 01..08-*.md          # Platform-wide docs
│   ├── frontend/            # Frontend-workspace docs
│   ├── backend/             # Backend-service docs
│   ├── database/schema.sql  # PostgreSQL schema reference copy
│   └── references/          # Source material (Article.pdf)
├── .github/                 # CI / tooling hooks
├── .gitattributes
└── .gitignore
```

## Cross-cutting contracts

These are the agreements that span the frontend ↔ backend boundary. Each has a
canonical home:

| Contract | Canonical document |
|---|---|
| Scientific/domain model (v2 entities, relationships, invariants) | [03-domain-model.md](./03-domain-model.md) |
| PostgreSQL schema (tables, enums, triggers, indexes) | [04-database.md](./04-database.md) |
| REST API (endpoints, filters, DTO shapes, pagination) | [05-api-contract.md](./05-api-contract.md) |

Two conventions that recur throughout the stack:

- **Pagination envelope.** Every paginated `GET` returns the same envelope
  (`content`, `page`, `size`, `totalElements`, `totalPages`, `first`, `last`,
  `hasNext`, `hasPrevious`, `sort`). Sort uses repeated `sort=field,direction`
  parameters. See [05-api-contract.md](./05-api-contract.md).
- **PATCH semantics (`JsonNullable`).** Absent key = no change; `null` = clear
  (where the column allows it); value = set. The frontend strips `undefined`
  keys before sending; the backend models the same three states with
  `JsonNullable<T>` update DTOs.

## Authentication boundary

- All `GET` endpoints on the public domain resources are anonymous.
- All writes (`POST` / `PATCH` / `DELETE`) and the entire `/api/admin/**`
  namespace require a JWT with `ROLE_ADMIN`.
- Authentication is **stateless JWT** (Spring Security OAuth2 Resource Server)
  with short-lived access tokens and rotating refresh tokens stored hashed in
  the database.
- The backend CORS policy allows `GET, POST, PATCH, DELETE, OPTIONS` — **not
  `PUT`**. Use `POST`/`PATCH` for mutations.

The full security map (route allowlist, JWT contract, refresh-token rotation,
rate limiting, CORS, security headers) lives in
[backend/02-security-and-auth.md](./backend/02-security-and-auth.md). The
frontend consumption side (token store, interceptor, guards) lives in
[frontend/03-backoffice.md](./frontend/03-backoffice.md).

## Deployment topology

- **Frontend** deploys to Vercel (`frontend/vercel.json`); the two Angular apps
  build independently.
- **Backend** runs under the `production` profile against a managed PostgreSQL
  instance, with a required `JWT_SECRET` and fail-fast startup validation.

The full deployment and environment-variable reference is in
[07-deployment.md](./07-deployment.md). To run the whole stack locally, see
[06-local-development.md](./06-local-development.md).

## Architectural invariants (platform-wide)

| Invariant | Why |
|---|---|
| The backend is the only database client. | The schema assumes a single writer; direct client DB access would require Row Level Security design first. |
| The v2 model is current; v1 names are forbidden. | `OsteologicalUnit`, `Specimen`, `BurialGroup`, top-level `Date` are removed. See [03-domain-model.md](./03-domain-model.md). |
| Public web and backoffice are isolated apps. | They share `node_modules`, not models. Domain types are duplicated on purpose; do not refactor shared types across the two apps. |
| The API contract is fixed for the frontend. | The frontend cannot rename endpoints, fields, or filters; those changes originate in the backend. |
| Raw SQL is never inlined into Markdown. | The schema lives in `.sql` files; docs link to them. See [04-database.md](./04-database.md). |

# 06 — Local Development

> **Scope.** How to run the whole stack locally: PostgreSQL → backend → frontend.
> Workspace-specific commands, conventions, and verification steps live in
> [frontend/06-frontend-workflow.md](./frontend/06-frontend-workflow.md) and
> [backend/06-backend-workflow.md](./backend/06-backend-workflow.md).
> Environment variables and deployment are in [07-deployment.md](./07-deployment.md).

## Prerequisites

- **Java 17 (JDK)** and **Maven** via the bundled wrapper (`backend/mvnw`, or
  `backend/mvnw.cmd` / `backend/run-backend.ps1` on Windows).
- **PostgreSQL 13+** for the backend `dev` profile. (The backend `test` profile
  uses in-process H2 and needs no PostgreSQL.)
- **Node.js** matching the engines in `frontend/package.json`, plus **npm**.

## 1. Database

The backend `dev` profile expects a local PostgreSQL instance on
**`localhost:5433`** with database **`paleohumans`**. There is no migration
tool; the schema is applied by hand from the authoritative
`backend/src/main/resources/db/database.sql` (see [04-database.md](./04-database.md)).

```bash
# Create the database
psql -h localhost -p 5433 -U postgres -c 'CREATE DATABASE paleohumans;'

# Apply the schema
psql -h localhost -p 5433 -U postgres -d paleohumans \
  -f backend/src/main/resources/db/database.sql
```

Hibernate runs with `ddl-auto=validate` in `dev`/`production`, so the
application refuses to start if the entity model and live schema disagree.

## 2. Backend (`http://127.0.0.1:8080`)

Run from the `backend/` directory. Export the required environment variables
first (see [07-deployment.md](./07-deployment.md) for the full list — at minimum
`SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`,
`SPRING_DATASOURCE_PASSWORD`, `JWT_SECRET`, and `SPRING_PROFILES_ACTIVE=dev`).

```bash
cd backend

# Build (compile + run tests + package)
./mvnw clean package          # Windows: .\mvnw.cmd clean package

# Run (default profile = dev; requires local PostgreSQL)
./mvnw spring-boot:run        # Windows: .\mvnw.cmd spring-boot:run  /  .\run-backend.ps1

# Run all tests (test profile + in-memory H2 — no external DB)
./mvnw test

# Run a single test class / method
./mvnw test -Dtest=ClassName
./mvnw test -Dtest=ClassName#methodName
```

The API listens on port **8080** by default (`PORT` to override). The `dev`
profile is **not** active by default — set `SPRING_PROFILES_ACTIVE=dev`
explicitly.

## 3. Frontend (`http://localhost:4200`)

An Angular CLI multi-project workspace (Angular 21, SSR) with two apps sharing
one `node_modules`. Run from the `frontend/` directory. The default dev
configuration points at `http://127.0.0.1:8080/api`, so the backend must be
running for data to load.

```bash
cd frontend
npm install

# Public web
ng serve --project=web              # http://localhost:4200

# Admin backoffice
ng serve --project=backoffice       # http://localhost:4200
```

Builds, tests, scaffolding, and the SSR runtime:

```bash
ng build --project=web              # production build
ng build --project=backoffice
ng test  --project=web              # Vitest unit tests
ng test  --project=backoffice
ng generate component my-component --project=web

npm run build:web                   # ng build --project=web --configuration production
npm run serve:ssr:web               # node dist/web/server/server.mjs (SSR runtime)
npm run serve:ssr:backoffice
```

## Useful commands

### Frontend (`cd frontend`)

| Command | Purpose |
|---|---|
| `npm install` | Install dependencies |
| `ng serve --project=web` | Serve the public web app (dev) |
| `ng serve --project=backoffice` | Serve the admin backoffice (dev) |
| `ng build --project=web` | Build the public web app |
| `ng test --project=web` | Run web unit tests (Vitest) |
| `npm run build:web` | Production web build |
| `npm run serve:ssr:web` | Run the SSR server bundle |

### Backend (`cd backend`)

| Command (Windows: `.\mvnw.cmd …`) | Purpose |
|---|---|
| `./mvnw clean package` | Compile, test, and package |
| `./mvnw spring-boot:run` | Run the API (dev profile) |
| `./mvnw test` | Run the full test suite (H2) |
| `./mvnw test -Dtest=ClassName` | Run a single test class |
| `./mvnw -DskipTests compile` | Compile only |

## Verification before declaring a task done

Run the smallest useful check first, broaden only if needed.

- **Backend:** `./mvnw test` for the affected package, then the full suite. See
  [backend/06-backend-workflow.md](./backend/06-backend-workflow.md).
- **Frontend:** `ng build --project=<app>` (type check) → `ng test
  --project=<app>` (Vitest) → a manual browser smoke test of the affected
  route, including an SSR smoke test for server-rendered routes
  (`/sites/:id`, `/individuals/:id`, `/analysis`). See
  [frontend/06-frontend-workflow.md](./frontend/06-frontend-workflow.md).

## Documentation update rules

When code changes affect documentation:

1. Update the **single source-of-truth document** for the topic, then cross-link
   from docs that briefly mention the same fact. Common-topic source-of-truth
   docs live at the `docs/` root; workspace-specific docs live under
   `docs/frontend/` and `docs/backend/` (see [README.md](./README.md)).
2. Do not duplicate explanations — the canonical document owns the prose; others
   link to it.
3. If a fact cannot be verified from the repository, mark it `Unknown`,
   `Unverified`, or `Requires code verification`.
4. Do not preserve contradictions; replace the older version.
5. Schema changes update `backend/src/main/resources/db/database.sql` and the
   reference copy `docs/database/schema.sql` together, then
   [04-database.md](./04-database.md). Raw SQL never goes inline into Markdown.

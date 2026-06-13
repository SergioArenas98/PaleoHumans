# 07 — Deployment and Environment

> **Scope.** Deployment targets and the full environment-variable reference for
> both workspaces. To run the stack locally, see
> [06-local-development.md](./06-local-development.md). Backend runtime tuning
> keys are detailed in
> [backend/06-backend-workflow.md](./backend/06-backend-workflow.md);
> the security/CORS model is in
> [backend/02-security-and-auth.md](./backend/02-security-and-auth.md).

> **Never hardcode secrets.** Provide credentials and JWT secrets via
> environment variables or a secret manager, validated at startup.

## Frontend deployment

- Deployed to **Vercel** (`frontend/vercel.json`): framework `angular`, install
  with `npm install`, build with `npm run build:web`.
- The production site redirects `www.paleohumans.org` → `paleohumans.org`.
- Each app owns its environment files; configuration is **build-time, not
  secret-bearing** (the public web basemaps are keyless — environment files
  should contain no browser-side map provider tokens).

| File | Purpose |
|---|---|
| `frontend/projects/web/src/app/environments/environment.ts` | Public web dev settings (e.g. `apiBaseUrl`) |
| `frontend/projects/web/src/app/environments/environment.prod.ts` | Public web production overrides (via `fileReplacements` in `angular.json`) |
| `frontend/projects/backoffice/src/app/environments/environment.ts` | Backoffice dev settings (`apiBaseUrl`, `sessionStorageKey`, …) |
| `frontend/projects/backoffice/src/app/environments/environment.prod.ts` | Backoffice production overrides |

## Backend deployment

The backend runs under the `production` profile against a managed PostgreSQL
instance. `ProductionStartupValidator` **fails fast** if any of
`spring.datasource.url`, `spring.datasource.username`,
`spring.datasource.password`, or `app.security.jwt.secret` is missing — there is
no committed fallback JWT secret. The exact hosting target is _to be confirmed_;
the production datasource uses Railway-style `PG*` variables with
`sslmode=require`.

| Profile | Datasource | `ddl-auto` | Notes |
| --- | --- | --- | --- |
| `dev` | `SPRING_DATASOURCE_*` (local PostgreSQL) | `validate` | `show-sql=true`; CORS allows `http://localhost:*` / `http://127.0.0.1:*` |
| `production` | `PG*` vars, `sslmode=require` | `validate` | fail-fast startup validation; SQL logging off |
| `test` | H2 (PostgreSQL compatibility) | `none` | used by `./mvnw test` |

The active profile is selected by `SPRING_PROFILES_ACTIVE` (defaults to `dev` in
local runs but is not set automatically — set it explicitly).

## Backend environment variables

**Required for `dev`:**

- `SPRING_DATASOURCE_URL` — e.g. `jdbc:postgresql://localhost:5433/paleohumans`
- `SPRING_DATASOURCE_USERNAME`
- `SPRING_DATASOURCE_PASSWORD`
- `JWT_SECRET` — symmetric HS256 secret

**Required for `production`:**

- `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER`, `PGPASSWORD`
- `JWT_SECRET` (no fallback — startup fails fast if missing)

**Optional:**

- `PORT` — server port (default `8080`).
- `BOOTSTRAP_ADMIN_ENABLED` (default `false`), `BOOTSTRAP_ADMIN_USERNAME`,
  `BOOTSTRAP_ADMIN_PASSWORD` — one-shot admin seeder. If the flag is enabled but
  the credentials are missing, startup throws; after first use, disable the flag.

Additional server-side caps and tuning live in
`backend/src/main/resources/application.yml` (`app.dataset.export.*`,
`app.search.bone-site.*`, `app.security.cors.production.*`,
`app.security.jwt.*`, `app.security.public-rate-limit.*`,
`app.security.rate-limit.*`). See
[backend/06-backend-workflow.md](./backend/06-backend-workflow.md) for the full
list.

## CORS and origins

- `dev`: `allowedOriginPatterns = http://localhost:*`, `http://127.0.0.1:*`.
- non-dev: `app.security.cors.production.allowed-origins` (comma-separated) and
  `allowed-origin-patterns` (supports e.g. `https://*.vercel.app`). The default
  production origins are `https://paleohumans.org`, `https://www.paleohumans.org`,
  `https://admin.paleohumans.com`.

Allowed methods: `GET, POST, PATCH, DELETE, OPTIONS` (**not `PUT`**). Full CORS
and security-header detail in
[backend/02-security-and-auth.md](./backend/02-security-and-auth.md).

## Edge / deployment-layer responsibilities

Some performance concerns are intentionally left to the deployment edge rather
than the application:

- HTTP response compression beyond Spring Boot's built-in gzip (e.g. Brotli).
- `Cache-Control` enforcement / CDN edge caching for the public read endpoints
  (the backend emits the headers; the edge must honour them).
- A distributed rate limiter for multi-instance deployments (the in-app limiters
  are JVM-local).

These are tracked in [08-roadmap.md](./08-roadmap.md) and the workspace roadmaps.

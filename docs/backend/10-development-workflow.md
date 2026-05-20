# 10 — Development Workflow

## Prerequisites

- Java 17 (JDK).
- Maven via the included wrapper (`./mvnw` on macOS/Linux,
  `mvnw.cmd` on Windows).
- PostgreSQL 13+ for the `dev` and `production` profiles. The
  `test` profile uses an in-process H2 database — no PostgreSQL
  required.
- An `.env` or shell that exports the required environment
  variables before starting the app (see below).

## Commands

```bash
# Build (compile + run tests + package)
./mvnw clean package

# Compile only
./mvnw -DskipTests compile

# Run all tests (uses the test profile + H2)
./mvnw test

# Run a single test class
./mvnw test -Dtest=ClassName

# Run a single test method
./mvnw test -Dtest=ClassName#methodName

# Start the application (default profile = dev)
./mvnw spring-boot:run

# Start with an explicit profile
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

Windows convenience script: `run-backend.ps1`.

## Profiles

| Profile | Datasource | Hibernate `ddl-auto` | JWT secret | Notes |
| --- | --- | --- | --- | --- |
| `dev` | env (`SPRING_DATASOURCE_*`) — local PostgreSQL | `validate` | `${JWT_SECRET}` | `spring.jpa.show-sql=true`; CORS allows `http://localhost:*` and `http://127.0.0.1:*`. |
| `production` | Railway-style env (`PG*` vars) with `sslmode=require` | `validate` | `${JWT_SECRET}` (required) | `ProductionStartupValidator` fails fast if any required env is missing. |
| `test` | H2 in PostgreSQL compatibility mode | `none` | fixed value | Used by `./mvnw test` automatically. |

Active profile is controlled by `SPRING_PROFILES_ACTIVE`. The
`dev` profile is **not** active by default — set the env var
explicitly when running locally:

```bash
export SPRING_PROFILES_ACTIVE=dev
```

## Environment variables

Required for `dev`:

- `SPRING_DATASOURCE_URL` — e.g. `jdbc:postgresql://localhost:5433/paleohumans`
- `SPRING_DATASOURCE_USERNAME`
- `SPRING_DATASOURCE_PASSWORD`
- `JWT_SECRET` — symmetric HS256 secret

Required for `production`:

- `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER`, `PGPASSWORD`
- `JWT_SECRET`

Optional:

- `BOOTSTRAP_ADMIN_ENABLED` (default `false`) — set to `true` to
  enable the one-shot admin seeder.
- `BOOTSTRAP_ADMIN_USERNAME`, `BOOTSTRAP_ADMIN_PASSWORD` — required
  if the flag is enabled.
- `PORT` — server port (default `8080`).

Server-side caps and tuning live in `application.yml`:

- `app.dataset.export.cache-ttl-seconds`,
  `app.dataset.export.max-bytes`
- `app.search.bone-site.max-match-rows`,
  `app.search.bone-site.max-page-size`
- `app.security.cors.production.allowed-origins`,
  `app.security.cors.production.allowed-origin-patterns`
- `app.security.jwt.issuer`,
  `app.security.jwt.access-token-minutes`,
  `app.security.jwt.refresh-token-days`,
  `app.security.jwt.refresh-token-idle-days`
- `app.security.public-rate-limit.*`,
  `app.security.rate-limit.*`

## Local database setup

The PostgreSQL schema lives at `src/main/resources/db/database.sql`
(byte-identical mirror at `docs/database/schema.sql`).

There is no migration tool; the schema is applied by hand. A typical
local bootstrap:

```bash
# Create the database (uses the LC_COLLATE defined in the schema)
psql -h localhost -p 5433 -U postgres -c 'CREATE DATABASE paleohumans;'

# Apply the schema
psql -h localhost -p 5433 -U postgres -d paleohumans \
  -f src/main/resources/db/database.sql
```

After the schema is in place, run the application; Hibernate's
`ddl-auto=validate` will verify the entity model against the live
schema and refuse to start on mismatch.

## Tests

Test configuration lives in
`src/main/resources/application-test.properties`:

```properties
spring.datasource.url=jdbc:h2:mem:paleohumans-test;MODE=PostgreSQL;DATABASE_TO_LOWER=TRUE;DB_CLOSE_DELAY=-1;NON_KEYWORDS=YEAR
spring.datasource.driver-class-name=org.h2.Driver
spring.jpa.hibernate.ddl-auto=none
```

`ddl-auto=none` is intentional: the production schema uses
PostgreSQL-specific named enum types, a `STORED` generated column,
and the `bone_catalog_component` view, which H2 cannot represent
faithfully. Tests that need that fidelity should run against a real
PostgreSQL instance (Testcontainers).

Existing test coverage (see `src/test/java/...`):

- `PaleohumansApiApplicationTests` — context boot.
- `config/ApplicationDefaultsTests`,
  `config/BootstrapAdminInitializerTests`,
  `config/ProductionStartupValidatorTests`.
- `security/SecurityConfigTests`,
  `security/AuthorizationIntegrationTests`,
  `security/PublicAllowlistV2SecurityTests`.
- `auth/AuthControllerTests`, `auth/AuthRateLimiterTests`,
  `user/UserServiceTests`.
- `site/SiteServiceTests`, `site/SiteControllerTests`.
- `individual/IndividualServiceTests`,
  `individual/IndividualControllerTests`.
- `reference/ReferenceServiceTests`,
  `reference/ReferenceControllerTests`,
  `reference/ReferenceSecurityTests`.
- `bone/BoneMapperTests`.
- `bone_site_search/BoneSiteSearchServiceTests`,
  `bone_site_search/BoneSiteSearchControllerTests`,
  `bone_site_search/BoneSiteSearchSecurityTests`.
- `stats/StatsServiceTests`, `stats/StatsControllerTests`,
  `stats/StatsSecurityTests`,
  `stats/AdminDashboardStatsServiceTests`,
  `stats/AdminDashboardStatsControllerTests`,
  `stats/AdminDashboardStatsSecurityTests`.
- `dataset/DatasetExportServiceTests`,
  `dataset/DatasetControllerTests`,
  `dataset/DatasetSecurityTests`.

Coverage gaps and recommended next test work are listed in
[11-roadmap.md](./11-roadmap.md).

## Security tooling

Configured but not yet wired into CI:

- `org.owasp:dependency-check-maven` via the Maven `security-scan`
  profile.
- `org.cyclonedx:cyclonedx-maven-plugin` via the same profile (SBOM
  generation).
- `.gitleaks.toml` for secret scanning.

Typical invocations:

```bash
./mvnw -Psecurity-scan verify
gitleaks detect --config .gitleaks.toml --source .
```

`.github/dependabot.yml` is present for dependency updates.

## Coding conventions

- One public top-level type per Java file.
- Domain code is organized by package (`<domain>/...`); cross-cutting
  utilities live under `common/`.
- DTOs are records for create/response and mutable classes with
  `JsonNullable<T>` for updates (see
  [02-architecture.md](./02-architecture.md) and
  [07-validation-and-errors.md](./07-validation-and-errors.md)).
- Validation happens at the controller (Bean Validation) and the
  service layer (business invariants).
- Controllers do not call repositories directly.
- Mapping is manual via static `*Mapper` classes (no MapStruct).
- Entity inverse collections are `@JsonIgnore`d or
  `@JsonIgnoreProperties`d to break serialization cycles; the
  response DTOs use flat `common/summary/*` records instead.
- New endpoints that perform heavy aggregation should be guarded by
  `PublicEndpointRateLimiter` and given a cache rule in
  `CacheControlInterceptor`.

## AI-agent workflow rules

When working on this repository:

1. **Read the code first.** When the docs disagree with the code,
   the code wins. Update the docs after verifying the code.
2. **Do not recreate v1 entities.** Use the canonical v2 names
   listed in [docs/README.md](./README.md).
3. **Schema changes** require updating
   `src/main/resources/db/database.sql` **and**
   `docs/database/schema.sql` in lockstep (they are byte-identical
   mirrors), then updating [04-database-model.md](./04-database-model.md).
4. **API changes** require updating
   [05-api-contract.md](./05-api-contract.md) and, if security
   boundaries change, [06-security-and-auth.md](./06-security-and-auth.md).
5. **New validation rules** belong in the service layer; record
   them in [07-validation-and-errors.md](./07-validation-and-errors.md)
   and [08-services-and-business-rules.md](./08-services-and-business-rules.md).
6. **Do not add new audit/history Markdown files.** Update the
   relevant numbered document instead.
7. **Raw SQL** never goes inline into Markdown. Keep it in
   `docs/database/` and link to it.
8. **Run the smallest useful verification first.** For backend
   changes that is `./mvnw test` for the affected package, then the
   full suite.

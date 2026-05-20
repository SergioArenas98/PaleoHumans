# 02 — Architecture

## Stack

- Java 17
- Spring Boot 3.5.6
  - `spring-boot-starter-web`
  - `spring-boot-starter-data-jpa` (Hibernate 6)
  - `spring-boot-starter-security`
  - `spring-boot-starter-oauth2-resource-server` + `spring-security-oauth2-jose`
  - `spring-boot-starter-validation`
- PostgreSQL 13+ in `dev` and `production`
- H2 in PostgreSQL compatibility mode in `test`
- Lombok for entity boilerplate
- `jackson-databind-nullable` for PATCH semantics

Build: Maven via the wrapper (`./mvnw`).

## Source layout

```text
src/
  main/
    java/com/sergio/paleohumans/paleohumans_api/
      PaleohumansApiApplication.java   <- @SpringBootApplication entry point
      archaeological_context/
      auth/
      bone/
      bone_catalog/
      bone_site_search/
      common/
        dto/
        summary/
        util/
      config/
      culture/
      culture_feature/
      dataset/
      dated_sample/
      dating_result/
      dating_technique/
      exception/
      funerary_context/
      individual/
      reference/
      refresh_token/
      security/
      site/
      skeleton/
      stats/
      user/
    resources/
      application.properties           <- shared defaults + compression
      application.yml                  <- shared app.* config
      application-dev.properties
      application-production.properties
      application-test.properties
      db/database.sql                  <- canonical PostgreSQL schema
  test/
    java/com/sergio/paleohumans/paleohumans_api/...   <- JUnit 5 tests
```

The package root is `com.sergio.paleohumans.paleohumans_api`. The
underscore in `paleohumans_api` is intentional — see `HELP.md`.

## Package convention

Each domain package follows the same internal layout:

```text
<domain>/
  <Domain>.java            <- JPA entity (or aggregate root)
  <Domain>Repository.java  <- Spring Data JPA repository
  <Domain>Service.java     <- transactional service / business rules
  <Domain>Controller.java  <- thin HTTP adapter (public)
  Admin<Domain>Controller.java  <- admin-only listing controller, where present
  Admin<Domain>Service.java     <- admin listing service, where present
  <Domain>Mapper.java      <- static entity <-> DTO mapping (no MapStruct)
  dto/
    <Domain>CreateRequest.java
    <Domain>UpdateRequest.java
    <Domain>Response.java
    Admin<Domain>ListResponse.java   <- where present
  <Enum>.java              <- domain enums bound to PostgreSQL named enums
```

Cross-cutting:

- `common/dto/PaginatedResponse.java` — pagination envelope.
- `common/summary/*` — flat summary DTOs (`SiteSummaryResponse`,
  `ArchaeologicalContextSummaryResponse`, `IndividualSummaryResponse`,
  `BoneSummaryResponse`, `SkeletonSummaryResponse`) plus
  `SummaryMappers`.
- `common/util/PaginationUtils.java` — page/size/sort parsing with an
  allow-list of sortable fields.
- `common/util/MapperUtils.java` — string normalization and
  `JsonNullable<T>` checks (`isDefined`, `requireNonBlank`).
- `common/util/EntityFetchUtils.java` — small fetch helpers.
- `exception/` — `GlobalExceptionHandler` and domain exceptions
  (`NotFoundException`, `UnauthorizedException`,
  `TooManyRequestsException`).
- `config/` — `WebConfig` (path prefix + interceptor),
  `JacksonConfig` (registers `JsonNullableModule`),
  `CacheControlInterceptor`, `BootstrapAdminInitializer`,
  `ProductionStartupValidator`.

## Layers

### Controllers

Controllers are thin HTTP adapters. They:

- declare the URL and HTTP method;
- bind `@RequestParam`, `@PathVariable`, `@RequestBody`;
- run Bean Validation via `@Valid`;
- delegate to a service;
- return a DTO record (or a `PaginatedResponse<T>` for lists).

Controllers do not call repositories directly and do not contain
persistence logic. Every controller class is mapped under `/api`
because `WebConfig` adds that prefix globally (see *Request lifecycle*
below).

### Services

Services own the transaction boundary (`@Transactional`) and the
business rules. They:

- look up related entities by id;
- enforce schema-aligned invariants before writes;
- replace many-to-many association sets on PATCH where the request
  carries an id list;
- call repositories;
- map entities to response DTOs while the JPA transaction is still
  open so lazy associations can be resolved through `@EntityGraph`
  detail queries.

### Repositories

Repositories extend `JpaRepository<T, ID>` and define query methods,
JPQL queries, or native queries. Detail endpoints use `@EntityGraph`
to load nested data in a single query. Aggregate endpoints
(`/api/stats/home`, `/api/stats/map-timeline`) use constructor
projections or `SUM(...)` queries directly instead of hydrating
entities.

### Mappers

Mappers are static utility classes (no MapStruct). They:

- map entities to response DTOs using `common/summary/SummaryMappers`
  to keep the JSON graph flat;
- apply `*UpdateRequest` PATCH semantics:
  - field omitted → do not change;
  - field present with non-null → update;
  - field present with `null` → clear (when the column allows null).

### DTOs

| DTO type | Form | Notes |
| --- | --- | --- |
| `*CreateRequest` | Java `record` | Bean Validation annotations (`@NotNull`, `@Min`, etc.). |
| `*UpdateRequest` | mutable class with `JsonNullable<T>` fields | distinguishes "omitted" from "explicit null". |
| `*Response` | Java `record` | flat shape, uses `*SummaryResponse` for related entities. |
| `*SummaryResponse` | Java `record` | shallow view used inside response DTOs to avoid recursive nesting. |

`@JsonIgnoreProperties` is applied on JPA collection sides to break
cyclic serialization; response DTOs deliberately do not expose entity
inverse collections.

## Request lifecycle

1. The servlet container (embedded Tomcat) receives the request.
2. Spring Security's filter chain runs:
   - CORS preflight handling (`SecurityConfig.corsConfigurationSource`);
   - OAuth2 resource-server JWT validation, when the path requires
     authentication;
   - method-level `@PreAuthorize` checks where present (e.g.
     `/api/users/**`);
   - route-level authorization rules from `SecurityConfig`.
3. `WebConfig.configurePathMatch` prefixes every controller mapping
   with `/api`, so a controller annotated `@RequestMapping("/sites")`
   actually responds at `/api/sites`.
4. `CacheControlInterceptor.preHandle` sets `Cache-Control` on `GET`
   responses for the public read-only paths (see
   [09-performance-and-pagination.md](./09-performance-and-pagination.md)).
5. Bean Validation runs on `@Valid` arguments.
6. The controller method runs and delegates to a `@Transactional`
   service.
7. The mapper produces a response DTO while the transaction is open.
8. Jackson serializes the DTO. `JsonNullableModule` is registered so
   PATCH semantics round-trip safely.
9. `GlobalExceptionHandler` catches domain and framework exceptions
   and returns RFC 7807 `ProblemDetail` bodies.

## Public vs admin flows

- **Public flow.** Anonymous requests use only `GET` on the whitelist
  declared in `SecurityConfig.filterChain`. They never carry a JWT
  and they never write data. `CacheControlInterceptor` adds
  `Cache-Control: public, max-age=..., stale-while-revalidate=...`
  responses for cache friendliness.
- **Admin flow.** Backoffice clients authenticate at
  `POST /api/auth/login`, then send the access token as
  `Authorization: Bearer <jwt>`. All write operations and all
  `/api/admin/**` routes require `ROLE_ADMIN`. Refresh tokens rotate
  at `POST /api/auth/refresh` and revoke at `POST /api/auth/logout`.

See [06-security-and-auth.md](./06-security-and-auth.md) for the full
security map.

## Configuration files

- `application.properties` — shared defaults: server port from `PORT`,
  `spring.jpa.open-in-view=false`, HTTP response compression enabled
  for JSON/XML/text at `min-response-size=1024`.
- `application.yml` — `app.*` configuration: dataset cache TTL and
  max bytes, bone-site search budgets, bootstrap admin toggles, CORS
  origins, JWT settings, public and auth rate-limit settings.
- `application-dev.properties` — PostgreSQL via env vars
  (`SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`,
  `SPRING_DATASOURCE_PASSWORD`), `ddl-auto=validate`, SQL logging on,
  JWT secret from `JWT_SECRET`.
- `application-production.properties` — PostgreSQL via Railway-style
  env vars (`PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER`, `PGPASSWORD`)
  with `sslmode=require`, `ddl-auto=validate`, SQL logging off.
- `application-test.properties` — H2 in PostgreSQL compatibility mode,
  `ddl-auto=none`, fixed test JWT secret, very high rate-limit cap.

`ddl-auto=validate` in `dev` and `production` means Hibernate
validates the schema against the entity model on startup but never
mutates it. Schema management is manual via `docs/database/schema.sql`.

`ProductionStartupValidator` fails fast in the `production` profile
if `spring.datasource.url`, `spring.datasource.username`,
`spring.datasource.password`, or `app.security.jwt.secret` are missing.

## External dependencies

| Dependency | Purpose |
| --- | --- |
| Spring Boot 3.5.6 | application platform |
| Hibernate 6 (transitive) | JPA implementation |
| Spring Security + OAuth2 Resource Server + Nimbus JOSE | JWT validation |
| PostgreSQL JDBC driver | runtime DB driver |
| H2 | test profile only |
| Lombok | entity getters/setters |
| `jackson-databind-nullable` | `JsonNullable<T>` PATCH semantics |
| `org.eclipse.jdt:org.eclipse.jdt.annotation` | nullability annotations |

Optional Maven profile `security-scan` enables OWASP Dependency-Check
and the CycloneDX SBOM plugin.

## Architectural constraints

The backend must not:

- recreate the v1 entities `OsteologicalUnit`, `Specimen`, top-level
  `Date`, or `BurialGroup`, nor any of the v1 bridge tables;
- introduce a name like `ArchaeologicalUnit` — the canonical name
  is `ArchaeologicalContext`;
- merge `Bone` and `Skeleton` into one entity — they are distinct
  scientific records (anatomical observation vs. preservation
  assessment) and share only a few curatorial fields by design;
- collapse `BoneCatalog` (controlled vocabulary) into `Bone` (observed
  data);
- treat `bone_catalog_component` as anything other than a read-only
  PostgreSQL view; do not add table constraints to it, do not write to
  it, and do not regenerate `bone_catalog_id` values that would
  invalidate it (see [04-database-model.md](./04-database-model.md));
- relax the v2 invariants enforced both in DB constraints and in
  service-layer validation (see [07-validation-and-errors.md](./07-validation-and-errors.md)
  and [08-services-and-business-rules.md](./08-services-and-business-rules.md)).

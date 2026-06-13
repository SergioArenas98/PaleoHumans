# 11 — Roadmap

This document tracks confirmed gaps, technical debt, and speculative
ideas. It is not a backlog — it captures non-obvious work so future
agents do not have to rediscover it. Items are grouped by confidence.

## Confirmed gaps (verified against the current code)

### Stale legacy paths in security and cache config

`security/SecurityConfig.java` still declares `permitAll` for
removed v1 paths: `GET /api/osteological-units/**`,
`GET /api/specimens/**`, `GET /api/dates/**`,
`GET /api/burial-groups/**`. No controller responds to these paths,
so they are dead config rather than an attack surface, but they
should be removed.

`config/CacheControlInterceptor.java` similarly references the dead
paths `/api/osteological-units`, `/api/specimens`,
`/api/burial-groups` inside `matches(...)`.

Cleanup scope: remove the matching lines from both files; no
behavior change for clients.

### No database migration tool

The PostgreSQL schema is applied by hand from
`src/main/resources/db/database.sql` (mirror at
`docs/database/schema.sql`). Adding Flyway or Liquibase would let
the test suite spin up real PostgreSQL via Testcontainers with
deterministic schema state and would track schema changes as
versioned migrations rather than ad-hoc edits.

### No OpenAPI document

There is no generated OpenAPI/Swagger document. Clients rely on
[../05-api-contract.md](../05-api-contract.md) plus controller
signatures. springdoc-openapi would publish a live spec without
much code churn.

### Rate limiting is JVM-local

`AuthRateLimiter` and `PublicEndpointRateLimiter` use in-memory
maps. They reset on restart and do not coordinate across instances.
A multi-instance production deployment needs a shared limiter
(Redis, edge gateway, or reverse proxy).

### No password reset flow

`SecurityAuditService` deliberately omits a `password_reset` event
because there is no reset flow. Implementing one (email/OTP) would
require a new endpoint and new audit event.

### Single role

Authorization has one role: `ROLE_ADMIN`. Finer permissions
(vocabulary editor, user manager, specimen editor, dataset reviewer)
would benefit the backoffice but are not modelled.

### Test coverage gaps

PostgreSQL-specific behavior is not covered by the H2 test profile:

- Named enum binding through `@JdbcTypeCode(SqlTypes.NAMED_ENUM)`.
- The generated `mni_statistical` column.
- The `bone_catalog_component` view.
- DB-level CHECK constraints
  (`chk_individual_type_mni_consistency`,
  `chk_dating_type_origin_consistency`, `chk_funerary_context_type`,
  `chk_mni_format`, `chk_age_range`,
  `chk_bone_quantity_min_positive`).

Highest-value test additions:

1. Testcontainers PostgreSQL tests for the items above.
2. MockMvc tests for `/api/bone-catalog`, `/api/bones`,
   `/api/funerary-contexts`, `/api/archaeological-contexts`,
   `/api/dated-samples`, `/api/dating-results`,
   `/api/dating-techniques`, `/api/skeletons`.
3. Service tests for many-to-many set replacement on
   `FuneraryContextService.update` (members),
   `SiteService.update` (references),
   `ArchaeologicalContextService.update` (references).
4. Service tests for `DatedSampleService.validate` exhaustively
   covering all origin/dating-type combinations.
5. Auth tests for refresh-token rotation replay attempts (the
   atomic-consume path), family revocation on replay, and
   `RefreshTokenCleanupJob`.

### Security scan automation

The Maven `security-scan` profile (OWASP Dependency-Check +
CycloneDX) and `gitleaks` are configured but not wired into CI.
Adding a scheduled GitHub Actions workflow would catch CVEs and
secrets early.

## Technical debt

### Wide entity graphs

`IndividualRepository.findDetailById` and similar use broad
`@EntityGraph` definitions. They are correct and fast for current
data volume but should be reviewed when individuals routinely carry
>50 bones with dated samples.

### Manual mapping

Mapping is done manually in static `Mapper` classes. This avoids
MapStruct's annotation processor but means new fields require
coordinated edits in the entity, the create DTO, the update DTO,
the response DTO, and the mapper. A future move to records-only
DTOs plus generated mappers (e.g. MapStruct or a small in-house
helper) would reduce boilerplate; today's pattern is intentional and
deliberate.

### `bone_catalog_component` view bound by id

The view uses fixed `bone_catalog_id` values for grouped→component
mappings. Seed data must use stable explicit IDs. A future
refactor could expose this composition through a real table or a
versioned data file; today the view is the contract.

### Missing add-one / remove-one endpoints for many-to-many sets

PATCH on `Site.references`, `ArchaeologicalContext.references`, and
`FuneraryContext.individuals` **replaces** the full set when the
list is defined. If the backoffice needs incremental edits, add new
endpoints rather than redefining the existing PATCH behavior — the
replacement semantics are explicit and worth preserving.

## API changes requested by the frontend

The cross-cutting list of API changes the frontend wants from the backend —
with implementation status (`/api/individuals/list`, `/api/sites/countries`,
and `/api/sites?country=` are **implemented**; `/api/individuals/{id}/bundle`,
`/api/stats/analysis`, the bone-site-search summary/inventory split, multi-id
`/api/dated-samples`, and Brotli compression are **not**) — is consolidated in
the platform roadmap:
[../08-roadmap.md → Cross-cutting API contract wishlist](../08-roadmap.md#cross-cutting-api-contract-wishlist).
Verify against the live backend before scheduling work.

## Speculative ideas

These are ideas; verify need before implementing.

- Cache `/api/stats/map-timeline` and `/api/stats/home` results in
  memory with a short TTL (the cached ZIP for `/api/dataset/download`
  is the pattern). Today the response is recomputed per request and
  relies on HTTP `Cache-Control` to absorb load.
- `ETag` support on detail endpoints (`/api/individuals/{id}`,
  `/api/archaeological-contexts/{id}`) for revalidation against
  `If-None-Match`.
- Audit-log table or event sink for admin writes beyond what
  `SecurityAuditService` logs to the application logger.
- Package-level Javadoc/`package-info.java` for each domain package
  to document the boundary in the source tree.
- A small smoke endpoint at `/api/healthz` or actuator health for
  load-balancer probes. *Requires code verification — Spring
  Actuator is not currently included in `pom.xml`.*

## Blockers and dependencies

- Testcontainers tests depend on Docker being available in the CI
  environment.
- A distributed rate limiter depends on a shared store
  (Redis/Memcached) provisioned alongside the backend.
- Adding Flyway/Liquibase requires backfilling a baseline migration
  for the current schema and updating the development workflow in
  [06-backend-workflow.md](./06-backend-workflow.md).

## Definitely-not-doing list

These are explicitly out of scope for the backend repository:

- Frontend caching strategies, bundle splitting, and HTTP transfer
  cache. Those live in the Angular workspace (`frontend/`).
- Reverse-proxy / CDN configuration. The backend exposes the right
  headers; the edge layer must enforce its own policy.
- Direct client-to-database access. The schema assumes the backend
  is the only DB client. Introducing direct access would require
  Row Level Security design first.

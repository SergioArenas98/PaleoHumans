# Backend Documentation

This folder is the knowledge base for the **PaleoHumans backend** (`backend/`) —
the Spring Boot REST API. It is written for AI coding agents and for any engineer
joining the project. Treat it as reference, not as a changelog.

> **Scope.** This folder describes **only** the backend service. Platform-wide
> topics — the project overview, system architecture, the scientific domain
> model, the database schema, and the REST API contract — are shared with the
> frontend and live at the `docs/` root. See [../README.md](../README.md) for the
> full map.

## Shared (platform-wide) references

| Topic | Document |
|---|---|
| What PaleoHumans is | [../01-project-overview.md](../01-project-overview.md) |
| How the components fit together | [../02-system-architecture.md](../02-system-architecture.md) |
| Scientific/domain model (v2 entities) | [../03-domain-model.md](../03-domain-model.md) |
| Database schema | [../04-database.md](../04-database.md) · raw DDL: [../database/schema.sql](../database/schema.sql) |
| REST API contract | [../05-api-contract.md](../05-api-contract.md) |
| Scientific source | [../references/Article.pdf](../references/Article.pdf) |

## Reading order (backend service)

| # | Document | Purpose |
|---|---|---|
| 0 | [README.md](./README.md) | This file. Index and scope. |
| 1 | [01-backend-architecture.md](./01-backend-architecture.md) | Spring Boot architecture, package layout, request lifecycle, configuration. |
| 2 | [02-security-and-auth.md](./02-security-and-auth.md) | Authentication, JWT, refresh tokens, CORS, rate limiting, security headers. |
| 3 | [03-validation-and-errors.md](./03-validation-and-errors.md) | Bean Validation, service-layer rules, RFC 7807 error responses. |
| 4 | [04-services-and-business-rules.md](./04-services-and-business-rules.md) | Service responsibilities and non-obvious invariants. |
| 5 | [05-performance-and-pagination.md](./05-performance-and-pagination.md) | Pagination contract, caching, entity graphs, response budgets. |
| 6 | [06-backend-workflow.md](./06-backend-workflow.md) | How to run, build, test; environment variables; conventions. |
| 7 | [07-roadmap.md](./07-roadmap.md) | Backend-specific gaps, technical debt, and speculative ideas. |

## Source-of-truth rules

1. The Java source under `src/main/java/...` is the authoritative description of
   API behaviour, validation, and persistence mappings.
2. The PostgreSQL schema at `src/main/resources/db/database.sql` is the
   authoritative database definition. A reference copy is kept at
   `docs/database/schema.sql` (see [../04-database.md](../04-database.md) for the
   relationship between the two — they should be kept in sync).
3. These Markdown documents describe the system at a stable, conceptual level.
   When code and a Markdown claim disagree, the code is right and the document
   must be updated.
4. Anything that cannot be verified from this repository is marked `Unknown`,
   `Unverified`, or `Requires code verification`.

## v2 terminology

Use the canonical v2 entity and table names; the full model (active names,
relationships, invariants, and the v1→v2 replacement map) is in
[../03-domain-model.md](../03-domain-model.md). The active entities are `Site`,
`Culture`/`CultureFeature`, `ArchaeologicalContext`, `Individual`, `Bone`,
`BoneCatalog`, `Skeleton`, `DatedSample`, `DatingResult`, `DatingTechnique`,
`Reference` (table `bibliographic_reference`), `FuneraryContext`, plus the
server-only `AppUser` and `RefreshToken`. The removed v1 names
(`OsteologicalUnit`, `Specimen`, top-level `Date`, `BurialGroup`,
`ArchaeologicalUnit`) must **not** reappear as active model entities or route
names.

## How to update this documentation

When you modify backend code, update the relevant numbered file. Do not add new
history logs or audit reports. If you remove a feature, remove its documentation;
do not leave behind "previously this did X" sections.

For raw artifacts (SQL schema, fixtures), keep them under `docs/database/` and
link to them from the relevant Markdown file. Never inline the full SQL schema
into a Markdown document.

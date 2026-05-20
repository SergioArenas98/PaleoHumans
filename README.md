# PaleoHumans

Scientific web platform for exploring the European Upper Palaeolithic *Homo sapiens* skeletal record.

PaleoHumans turns a peer-reviewed palaeoanthropological dataset into a public, searchable, map-based and chronologically structured web application. It combines a public research interface, a protected data-curation backoffice, a Spring Boot REST API and a PostgreSQL database designed around the scientific structure of the evidence.

The project is based on:

Arenas del Amo, S., Armentano Oller, N., Daura, J., & Sanz, M. (2024). *Overview of the European Upper Palaeolithic: The Homo sapiens bone record*. *Journal of Archaeological Science: Reports*, 53, 104391. https://doi.org/10.1016/j.jasrep.2024.104391

## Contents

- [What PaleoHumans is](#what-paleohumans-is)
- [Dataset and scientific scope](#dataset-and-scientific-scope)
- [Main features](#main-features)
- [System architecture](#system-architecture)
- [Technology stack](#technology-stack)
- [Domain model](#domain-model)
- [API overview](#api-overview)
- [Security model](#security-model)
- [Performance and UX strategy](#performance-and-ux-strategy)
- [Getting started](#getting-started)
- [Testing and quality checks](#testing-and-quality-checks)
- [Roadmap](#roadmap)
- [License and citation](#license-and-citation)

## What PaleoHumans is

PaleoHumans is a full-stack scientific data platform for European Upper Palaeolithic sites that have yielded human remains. Its purpose is to make a complex palaeoanthropological record easier to explore without flattening the scientific detail behind it.

The platform has three main components:

```text
Angular public web      Angular backoffice
        \                 /
         \               /
          Spring Boot REST API
                  |
              PostgreSQL
```

The public web is read-only and designed for researchers, students, teachers and informed public users. The backoffice is an internal administration application for curating records, relationships and controlled vocabularies. The backend exposes public GET endpoints, admin-protected write operations, aggregate read models and dataset export.

This is not a generic catalogue app. The data model reflects palaeoanthropological curation decisions: minimum number of individuals, anatomical representation, cultural attribution, stratigraphic context, dating events, bibliographic references and funerary information all have explicit places in the system.

## Dataset and scientific scope

The current dataset derives from the published European Upper Palaeolithic *Homo sapiens* bone record.

| Dimension | Published dataset snapshot |
|---|---:|
| Archaeological sites | 248 |
| Countries | 20 |
| Minimum number of individuals | 804 |
| Human bone remains | 6,604 |
| Skeleton records | 105 |
| Chronological scope | c. 55,000-11,700 cal BP |

The scientific record covers the arrival, expansion and consolidation of anatomically modern humans in Europe during the Late Pleistocene. The original article highlights that the Early Upper Palaeolithic record is dominated by isolated or disarticulated remains, while the Full and Final phases show larger skeletal samples and broader evidence of funerary practices.

The platform preserves specialist terminology such as MNI, MNBR, cal BP, AMS, archaeological context, Aurignacian, Gravettian, Magdalenian and funerary context because these terms are part of the scientific model, not just display labels.

## Main features

### Public research web

- Home page with live dataset statistics.
- Site catalogue with search, country filtering, pagination and site-level detail pages.
- Individual catalogue with biological profile, age, sex, cultural and geographic context.
- Individual detail pages with bones, skeleton preservation, dated samples, archaeological context and funerary information.
- Anatomical bone search by bone catalogue, skeletal region, laterality, dental fields, vertebrae, ribs, phalanges, culture, country and dating availability.
- Interactive map driven by aggregated site and culture data.
- Timeline view for chronological exploration of Upper Palaeolithic cultures.
- Bibliography page with references used by the dataset.
- Dataset download as a ZIP archive of UTF-8 CSV files.
- Methodology and About pages explaining the scientific basis and data structure.
- Analysis route for internal/statistical exploration.

### Backoffice

- JWT-protected admin login.
- Dashboard with scalar counts across the domain.
- CRUD interfaces for sites, cultures, archaeological contexts, individuals, bones, bone catalogue entries, skeletons, funerary contexts, dated samples, dating results, dating techniques, bibliographic references and users.
- Relation editing for supported many-to-many associations, such as site references, archaeological-context references and funerary-context individuals.
- Admin user management and self-service password change.
- Dedicated lightweight admin list projections for high-traffic entities where implemented.

### Backend and data services

- Public read-only REST endpoints for the research interface.
- Admin-protected write operations.
- Aggregated stats endpoints for Home, Map and Timeline.
- Bone-site search aggregation endpoint.
- Full public dataset export endpoint.
- Stateless JWT authentication with refresh-token rotation.
- Validation of scientific invariants before writes.

## System architecture

### Frontend workspace

The frontend is an Angular CLI multi-project workspace with two independent applications:

```text
projects/
  web/          Public-facing research site
  backoffice/   Internal admin application
```

Both apps use Angular standalone components. They share dependencies and build tooling, but their domain models, services, routes and design systems remain intentionally separate.

Key frontend architecture decisions:

- Angular 21.
- Standalone components only.
- No NgModules.
- Server-side rendering through `@angular/build:application`.
- Public routes are prerendered where possible.
- Parametric and data-heavy routes use server rendering.
- Browser-only APIs such as `window`, `document`, `localStorage`, Leaflet and `IntersectionObserver` are guarded for SSR safety.
- Public web uses no authentication layer.
- Backoffice stores JWT state in a signal-based auth store and protects all `/admin/*` routes.

### Backend

The backend is a separate Spring Boot API:

```text
src/main/java/com/sergio/paleohumans/paleohumans_api/
  archaeological_context/
  auth/
  bone/
  bone_catalog/
  bone_site_search/
  common/
  config/
  culture/
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
```

Each domain follows a conventional layered structure:

```text
<domain>/
  <Domain>.java
  <Domain>Repository.java
  <Domain>Service.java
  <Domain>Controller.java
  <Domain>Mapper.java
  dto/
```

Controllers are thin HTTP adapters. Services own transaction boundaries and business rules. Repositories use Spring Data JPA, JPQL projections and entity graphs where appropriate. Mappers convert entities into flat DTOs and apply PATCH semantics.

### Database

PostgreSQL is the source of truth for the scientific data. The schema is manually managed and validated by Hibernate on startup in dev and production. It uses:

- named PostgreSQL enum types;
- generated columns, including statistical MNI;
- check constraints for core scientific invariants;
- foreign keys and many-to-many bridge tables;
- a read-only `bone_catalog_component` view for anatomical decomposition.

## Technology stack

### Public web and backoffice

- Angular 21
- Angular SSR / SSG
- TypeScript
- RxJS
- Vitest
- Leaflet for maps
- MapLibre GL Leaflet for vector basemap support
- Per-app CSS design tokens
- Browser hydration with event replay

### Backend

- Java 17
- Spring Boot 3.5.6
- Spring Web
- Spring Data JPA
- Hibernate 6
- Spring Security
- OAuth2 Resource Server / JOSE support
- Bean Validation
- PostgreSQL JDBC driver
- H2 for test profile
- Maven wrapper
- `jackson-databind-nullable` for JSON PATCH-style semantics

### Database

- PostgreSQL 13+
- H2 in PostgreSQL compatibility mode for standard tests
- Manual schema application
- Hibernate `ddl-auto=validate` in dev and production

## Domain model

PaleoHumans uses the v2 scientific model. Legacy v1 entities such as `OsteologicalUnit`, `Specimen`, `BurialGroup` and top-level `Date` have been removed and must not be reintroduced.

High-level hierarchy:

```text
Site
└── ArchaeologicalContext
    ├── Individual
    │   ├── Bone
    │   └── Skeleton
    ├── FuneraryContext
    └── BibliographicReference

DatedSample
└── DatingResult
    └── DatingTechnique

Culture
└── CultureFeature

BoneCatalog
└── BoneCatalogComponent view
```

### Core entities

- `Site` - geographic locality with country, region, municipality and WGS84 coordinates.
- `ArchaeologicalContext` - stratigraphic and cultural slice within a site.
- `Individual` - central anthropological record. It may represent a single individual, mixed individuals or unassigned remains.
- `Bone` - observed anatomical record attached to an individual, with quantity and curatorial metadata.
- `Skeleton` - whole-skeleton preservation assessment, separate from individual bone records.
- `FuneraryContext` - burial or funerary grouping within an archaeological context, linked to one or more individuals.
- `DatedSample` - the material or origin of a dating event. It points to exactly one archaeological context, bone or skeleton.
- `DatingResult` - the numerical result of a dating event.
- `Culture` - archaeological technocomplex or cultural attribution, including phase, BP range, colours and descriptive features.
- `BoneCatalog` - controlled anatomical vocabulary.
- `BibliographicReference` - literature linked to sites and archaeological contexts.

### Modelling decisions worth noticing

- MNI is stored as text to preserve curated wording such as `"?"`, while a generated `mni_statistical` integer supports statistics.
- `Individual` does not always mean one physically separated biological person; `individualType` makes that explicit.
- `Bone` and `Skeleton` are separate records, not interchangeable representations of the same thing.
- `DatedSample` separates what was dated from the dating result.
- Funerary information belongs to `FuneraryContext`, not directly to `Individual`.
- Bibliographic references are shared and linked rather than duplicated.

## API overview

The API is served under `/api`.

### Public endpoints

Public endpoints are anonymous GET endpoints used by the research web:

| Area | Endpoint examples |
|---|---|
| Stats | `GET /api/stats/home`, `GET /api/stats/map-timeline` |
| Sites | `GET /api/sites`, `GET /api/sites/:id`, `GET /api/sites/countries` |
| Archaeological contexts | `GET /api/archaeological-contexts`, `GET /api/archaeological-contexts/:id` |
| Individuals | `GET /api/individuals`, `GET /api/individuals/list`, `GET /api/individuals/:id` |
| Bones | `GET /api/bones`, `GET /api/bone-site-search` |
| Skeletons | `GET /api/skeletons` |
| Funerary contexts | `GET /api/funerary-contexts` |
| Dating | `GET /api/dated-samples`, `GET /api/dating-results`, `GET /api/dating-techniques` |
| Cultures | `GET /api/cultures`, `GET /api/cultures/:id` |
| References | `GET /api/references` |
| Dataset | `GET /api/dataset/download` |

### Admin endpoints

Write operations require an authenticated admin token:

```text
POST /api/auth/login
POST /api/auth/refresh
POST /api/auth/logout

POST   /api/<resource>
PATCH  /api/<resource>/:id
DELETE /api/<resource>/:id

GET /api/admin/stats/dashboard
GET /api/admin/sites
GET /api/admin/individuals
GET /api/admin/bones
```

### Pagination and PATCH semantics

Paginated endpoints return a standard envelope:

```ts
interface PaginatedResponse<T> {
  content: T[];
  page: number;
  size: number;
  totalElements: number;
  totalPages: number;
  first: boolean;
  last: boolean;
  hasNext: boolean;
  hasPrevious: boolean;
  sort: string[];
}
```

PATCH semantics distinguish between omitted fields, explicit `null` and actual values:

- absent key - leave unchanged;
- key with `null` - clear the field when allowed;
- key with value - update the field.

Frontend services strip `undefined` keys before sending PATCH payloads.

## Security model

The public web is read-only and anonymous. It never sends an `Authorization` header and does not store session state.

The backoffice uses JWT-based authentication:

- login returns an access token and refresh token;
- access tokens carry a `role` claim such as `ROLE_ADMIN`;
- refresh tokens rotate on use;
- logout revokes the refresh-token family;
- passwords are hashed with BCrypt;
- one role is currently modelled: `ROLE_ADMIN`.

Additional backend security features include:

- CORS allowlists for local and production origins.
- Sliding-window rate limiting for login and refresh.
- Public endpoint rate limiting for anonymous dataset download and bone-site search.
- Security headers such as CSP, Referrer-Policy, X-Frame-Options, X-Content-Type-Options, COOP and CORP.
- Structured audit events for login, password change, role change, user CRUD and dataset downloads.
- No HTTP sessions and no CSRF requirement for stateless bearer-token usage.

## Performance and UX strategy

PaleoHumans is optimized around a simple constraint: scientific data pages must be precise, but they must not feel heavy.

Implemented strategies include:

- SSR/SSG for SEO and first render quality.
- Hydration transfer cache for server-fetched data.
- Dedicated aggregate endpoints instead of reconstructing statistics client-side.
- Lightweight `/api/individuals/list` projection for public list pages.
- Map and Timeline powered by `/api/stats/map-timeline` instead of unfiltered collection sweeps.
- Service-level `shareReplay` caches for map/timeline stats, cultures and sites.
- Delayed loading indicators to avoid loader flicker.
- Dynamic browser-only imports for Leaflet.
- Responsive AVIF/WebP/JPG hero assets.
- Optimized logo variants.
- Subsetted Material Symbols loading.
- Map basemap fade-in and terrain veil to avoid white flashes during tile loading.

Known performance debt is documented rather than hidden. The main future wins are stronger cache headers, JSON compression at the edge/API layer, an individual detail bundle endpoint and a dedicated analysis stats endpoint.

## Roadmap

Confirmed or high-value future work includes:

- Add `/cultures/:id` on the public web.
- Replace hardcoded About and Methodology stats with live or aggregated values.
- Remove dead frontend references to legacy `/api/bone-names`.
- Add richer backend projections for expensive admin list pages.
- Add `country` and `cultureId` filters to `/api/individuals`.
- Add `GET /api/individuals/{id}/bundle` or an equivalent expansion endpoint.
- Add `GET /api/stats/analysis` to remove the heavy analysis sweep.
- Improve cache headers and response compression for large JSON endpoints.
- Add OpenAPI documentation for the backend.
- Add Flyway or Liquibase migrations.
- Add PostgreSQL Testcontainers coverage.
- Replace JVM-local rate limiting with shared infrastructure for multi-instance deployment.
- Consider finer-grained backoffice roles beyond `ROLE_ADMIN`.

## What this project demonstrates

For engineering reviewers, PaleoHumans demonstrates:

- domain modelling for non-trivial scientific data;
- full-stack architecture across Angular, Spring Boot and PostgreSQL;
- public read-only and private admin surfaces over the same domain;
- SSR-safe frontend engineering;
- authenticated CRUD with token rotation and audit logging;
- performance-aware API consumption;
- careful handling of many-to-many relations and PATCH semantics;
- transformation of a static academic dataset into an explorable research tool.

## License and citation

The article is published under CC BY 4.0. The dataset is documented as CC BY-SA 4.0.

Source-code licensing should be declared in the repository before public release if it differs from the dataset license.

Recommended citation:

```text
Arenas del Amo, S., Armentano Oller, N., Daura, J., & Sanz, M. (2024).
Overview of the European Upper Palaeolithic: The Homo sapiens bone record.
Journal of Archaeological Science: Reports, 53, 104391.
https://doi.org/10.1016/j.jasrep.2024.104391
```

## Acknowledgement

PaleoHumans exists to make a specialized palaeoanthropological record more accessible, reusable and inspectable while preserving the structure and uncertainty of the original scientific evidence.

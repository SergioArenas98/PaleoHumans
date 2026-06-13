# 01 — Project Overview

> **Canonical platform overview.** This is the single source of truth for *what
> PaleoHumans is*. Workspace-specific framing lives in
> [frontend/01-frontend-architecture.md](./frontend/01-frontend-architecture.md)
> and [backend/01-backend-architecture.md](./backend/01-backend-architecture.md).

## What PaleoHumans is

PaleoHumans is a public scientific web platform for the **European Upper
Palaeolithic _Homo sapiens_ skeletal record** — the documented human bone
remains from European sites dated to approximately **55,000–11,700 cal BP**. It
turns the static dataset behind a peer-reviewed study into a dynamic,
searchable, curatable relational application.

It is the digital companion to:

> Arenas del Amo, S., Armentano Oller, N., Daura, J., & Sanz, M. (2024).
> *Overview of the European Upper Palaeolithic: The Homo sapiens bone record.*
> *Journal of Archaeological Science: Reports*, 53, 104391.
> https://doi.org/10.1016/j.jasrep.2024.104391

The original spreadsheet behind the article was normalised into a relational
schema to ensure data integrity and enable advanced anatomical, geographic, and
chronological queries. The primary source PDF lives at
[references/Article.pdf](./references/Article.pdf).

## Scientific scope

| Item | Value (article snapshot) |
|---|---|
| Documented individuals | 804 |
| Bone remains | 6,604 |
| Archaeological sites | 248 |
| Countries | 20 |
| Time span | c. 55,000–11,700 cal BP |

> These are the **published article figures**. The public web still hardcodes
> them on the About and Methodology pages. Live counts come from
> `GET /api/stats/home` and may diverge from the snapshot. See
> [frontend/07-roadmap.md](./frontend/07-roadmap.md).

## Audience

The platform targets three readerships, in descending priority:

1. **Professional researchers** — archaeologists, osteologists,
   palaeoanthropologists.
2. **Students and teachers** — undergraduates, graduates, lecturers.
3. **Informed general public** — readers with sustained interest in prehistory.

The interface treats domain vocabulary (MNI, MNBR, cal BP, AMS, stratigraphic
context, Aurignacian, Gravettian, Magdalenian, etc.) as **non-negotiable**.
Plain-language rewording requires author review.

## Product goals

- Make the dataset **explorable** and **visually compelling**.
- Preserve **scientific rigour** — exact citations, exact dates, exact
  terminology.
- Provide multiple access paths: geographic (sites/map), biological
  (individuals), chronological (timeline, cultures), anatomical (bones).
- Make the dataset **downloadable** so it can be reused under its license.
- Provide a secure curation surface so authorised editors can maintain the
  dataset without touching code.

## Key features

- **Interactive database** — full-text search and advanced filtering on
  archaeological, chronological, and anatomical criteria.
- **Anatomical filtering** — query by specific bone types (e.g. Mandible,
  Femur, Teeth).
- **Geospatial visualization** — interactive maps (Leaflet) of archaeological
  sites.
- **Detailed records** — per-specimen pages with references, dating, and
  cultural/stratigraphic context.
- **API access** — documented REST API for external researchers and
  applications.
- **Data curation** — secure, JWT-protected backoffice for authorized editors.

## The platform at a glance

PaleoHumans is a **monorepo** with three deployable parts plus shared
documentation and database artifacts:

| Part | Stack | Role |
|---|---|---|
| Public web (`frontend/projects/web`) | Angular 21 (SSR) | Read-only research database UI; fully public |
| Backoffice (`frontend/projects/backoffice`) | Angular 21 (SSR) | JWT-protected admin/curator CRUD app |
| Backend (`backend/`) | Spring Boot 3.5.6 / Java 17 | REST API; owns all data access |
| Database | PostgreSQL | Single source of truth for the scientific data |

For how these connect, the deployment topology, and the monorepo layout, see
[02-system-architecture.md](./02-system-architecture.md).

## Licensing

- **Article:** CC BY 4.0.
- **Database:** CC BY-SA 4.0.

Both must remain clearly displayed on the public site (About, Dataset), along
with the article DOI and citation.

## Public dataset access

A dedicated public page at `/dataset` allows downloading the full structured
dataset as a ZIP of UTF-8 CSV files. The backend endpoint is
`GET /api/dataset/download` (no JWT). See
[frontend/02-public-web.md](./frontend/02-public-web.md) for the page and
[05-api-contract.md](./05-api-contract.md) for the endpoint contract.

## Current model version (v2)

The platform is on the **v2 scientific model**. The current entities are `Site`,
`ArchaeologicalContext`, `Individual`, `Bone`, `Skeleton`, `FuneraryContext`,
`DatedSample`, and `DatingResult`, plus supporting `Culture`, `BoneCatalog`,
`Reference`, and `DatingTechnique`. The deprecated v1 entities
`OsteologicalUnit`, `Specimen`, `BurialGroup`, and a top-level `Date` **must not
be reintroduced**. See [03-domain-model.md](./03-domain-model.md) for the full
model and the v1→v2 replacement map.

## Relationship to the underlying article

Every record in the database derives from the Arenas del Amo et al. (2024) paper
and the curated source bibliography behind it. The site treats the article as
the canonical scientific reference. Any change to scientific content (taxonomy,
dating, attribution) must round-trip through the authors, not through frontend
edits.

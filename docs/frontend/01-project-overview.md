# 01 — Project Overview

## What PaleoHumans is

PaleoHumans is a public scientific web platform for **European Upper Palaeolithic sites with human remains**. It is the digital companion to a peer-reviewed study:

> Arenas del Amo, S., Armentano Oller, N., Daura, J., & Sanz, M. (2024).
> *Overview of the European Upper Palaeolithic: The Homo sapiens bone record.*
> *Journal of Archaeological Science: Reports*, 53, 104391.
> https://doi.org/10.1016/j.jasrep.2024.104391

The database compiles and standardises osteological records from European Upper Palaeolithic sites (approximately **55,000–11,700 cal BP**), covering site localities, archaeological context, anthropological data, and mortuary behaviour for each documented individual. The web platform extends the article's static dataset into a live, queryable application.

## Scientific scope

| Item | Value (article snapshot) |
|---|---|
| Documented individuals | 804 |
| Bone remains | 6,604 |
| Archaeological sites | 248 |
| Countries | 20 |
| Time span | c. 55,000–11,700 cal BP |

> The figures above are the **published article figures**. The public web still hardcodes them on the About and Methodology pages. Live counts come from `GET /api/stats/home` and may diverge. See [10-roadmap.md](./10-roadmap.md).

## Audience

The platform targets three readerships, in descending priority:

1. **Professional researchers** — archaeologists, osteologists, palaeoanthropologists.
2. **Students and teachers** — undergraduates, graduates, lecturers.
3. **Informed general public** — readers with sustained interest in prehistory.

The interface treats domain vocabulary (MNI, MNBR, cal BP, AMS, stratigraphic context, Aurignacian, Gravettian, Magdalenian, etc.) as **non-negotiable**. Plain-language rewording requires author review.

## Main product goals

- Make the dataset **explorable** and **visually compelling**.
- Preserve **scientific rigour** — exact citations, exact dates, exact terminology.
- Provide multiple access paths: geographic (sites/map), biological (individuals), chronological (timeline, cultures), anatomical (bones).
- Make the dataset **downloadable** so it can be reused under its license.

## Two applications

The repository is an Angular CLI multi-project workspace with two distinct applications. They share `node_modules` and a build toolchain, but they are independent products.

| Concern | `projects/web` (public) | `projects/backoffice` (admin) |
|---|---|---|
| Purpose | Public research database — browse, explore, export | Data curation — full CRUD over every entity |
| Audience | Researchers, students, public | Project editors / curators |
| Auth | None — fully public | JWT (`/admin/login`) required |
| Routing | Eager + lazy under a public shell | Lazy-loaded, guarded under `/admin` |
| State | Component-local; signals where useful | `AdminAuthStore` (signal, `localStorage`) |
| API access | GET only (read) | Full GET/POST/PATCH/DELETE on `/api/*` and `/api/admin/*` |
| Visual system | DEEP TIME tokens (`--dt-*`) | TERRAIN tokens (light, ochre accent) |
| Redesign focus | Active | Stable — out of scope for the public redesign |

See [05-public-web.md](./05-public-web.md), [06-backoffice.md](./06-backoffice.md), and [07-design-system.md](./07-design-system.md).

## Licensing

- **Article:** CC BY 4.0.
- **Database:** CC BY-SA 4.0.

Both must remain clearly displayed on the public site (About, Dataset). The article DOI and citation must remain visible and correct on the relevant pages.

## Public dataset access

A dedicated public page at `/dataset` allows downloading the full structured dataset as a ZIP of UTF-8 CSV files. The backend endpoint is `GET /api/dataset/download` (no JWT). See [05-public-web.md](./05-public-web.md) for the page details and [04-backend-api-contract.md](./04-backend-api-contract.md) for the endpoint contract.

## Relationship to the underlying article

Every record in the database derives from the Arenas del Amo et al. (2024) paper and the curated source bibliography behind it. The site treats the article as the canonical scientific reference. Any change to scientific content (taxonomy, dating, attribution) needs to round-trip through the authors, not through frontend edits.

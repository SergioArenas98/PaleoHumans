# 01 — Project Overview

## What this backend is

`paleohumans-api` is a Spring Boot 3.5 / Java 17 REST API. It exposes
the curated European Upper Palaeolithic *Homo sapiens* skeletal record
as structured JSON, backed by a PostgreSQL database.

The data is the relational normalization of the dataset published in:

> Arenas del Amo S., Armentano Oller N., Daura J., Sanz M.
> *Overview of the European Upper Palaeolithic: The Homo sapiens bone record.*
> Journal of Archaeological Science: Reports, 53 (2024), 104391.

The original PDF lives at `docs/Article.pdf` as the scientific source.

## Role in the PaleoHumans platform

The platform has three components:

```text
[ Angular public web ]   [ Angular backoffice ]
            \                    /
             \                  /
              \                /
             [ paleohumans-api ]   <-- this repository
                      |
                      v
               [ PostgreSQL ]
```

- The **public web** consumes anonymous GET endpoints to render the
  site catalogue, individuals, bones, dated samples, the map/timeline,
  and bibliography. It does not authenticate.
- The **backoffice** authenticates as `ROLE_ADMIN` to write data
  (CRUD for sites, contexts, individuals, bones, skeletons, dated
  samples, dating results, funerary contexts, references, cultures,
  bone catalog entries, users).
- The **database** is owned by the backend. No frontend has direct
  database access.

This repository documents the backend from the backend perspective.
Frontend assumptions live in the frontend repositories.

## Main responsibilities

The backend is responsible for:

1. Serving anonymous, read-only data for the public web (no JWT
   required for `GET` endpoints on the public resources listed in
   [06-security-and-auth.md](./06-security-and-auth.md)).
2. Serving admin-authenticated write operations for the backoffice.
3. Enforcing the v2 scientific model invariants (MNI/individual-type
   consistency, dated-sample origin/dating-type consistency, funerary
   burial-context/type consistency, age-range coherence).
4. Providing aggregated read models for the public Home and
   Map/Timeline pages (`/api/stats/home`, `/api/stats/map-timeline`).
5. Providing a bone-site search aggregation
   (`/api/bone-site-search`) and an admin dashboard summary
   (`/api/admin/stats/dashboard`).
6. Exporting the full public dataset as a ZIP of CSV files
   (`/api/dataset/download`).
7. Authentication: stateless JWT access tokens plus persisted refresh
   tokens with family rotation, password hashing, and security audit
   events.

## Scientific purpose, in backend terms

Each row in the database represents a curated piece of palaeo-
anthropological evidence. The backend preserves the curation
decisions encoded in the v2 schema:

- A `Site` is a geographic locality.
- An `ArchaeologicalContext` is a stratigraphic/cultural slice within
  a site.
- An `Individual` is the central anthropological record (one
  individual, mixed individuals, or unassigned remains).
- A `Bone` is an observed anatomical record with quantity and
  curatorial fields.
- A `Skeleton` is a whole-skeleton preservation assessment (not a
  list of bones).
- A `DatedSample` describes the material/origin of a dating event;
  it points to exactly one of a context, a bone, or a skeleton.
- A `DatingResult` is the numeric output of a dating event.
- A `FuneraryContext` is a burial/funerary group within a context,
  with member individuals through `funerary_context_individual`.
- A `BibliographicReference` ties literature to sites and contexts.

See [03-domain-model.md](./03-domain-model.md) for the full model and
[04-database-model.md](./04-database-model.md) for the persistence
layer.

## Current model version

The current model is **v2**. The v1 model (centered on
`osteological_unit`, `specimen`, `date`, `burial_group`) is removed
from code and database. The canonical replacements are listed in
[docs/README.md](./README.md) and must be used in all new work.

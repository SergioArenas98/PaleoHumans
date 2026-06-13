# 03 — Domain Model (v2)

> **Canonical scientific model.** This is the single source of truth for the
> PaleoHumans **v2** entities and how they relate, shared by the backend
> (Java/JPA entities) and both frontends (TypeScript response models). The
> persistence-level detail (table names, columns, generated columns, triggers,
> indexes) lives in [04-database.md](./04-database.md); the exact JSON DTO
> shapes live in [05-api-contract.md](./05-api-contract.md).

The platform is on the **v2 model**. The v1 entities `OsteologicalUnit`,
`Specimen`, `BurialGroup`, and a top-level `Date` have been replaced by
`ArchaeologicalContext`, `Individual` (now owning MNI fields), `FuneraryContext`,
`DatedSample`, and `DatingResult`. Anything still referencing the v1 names in
old comments or redirects is deprecated — see [Deprecated v1 names](#deprecated-v1-names-do-not-reintroduce).

## High-level hierarchy

```text
Site                              ← primary geographic locality
└── ArchaeologicalContext         ← stratigraphic + cultural slice within a site
    ├── Individual                ← anthropological record (1, mixed, or unassigned remains)
    │     ├── Bone                ← per-element record; many-to-many via bone_individual
    │     └── Skeleton            ← whole-skeleton preservation record (1:1 with Individual)
    └── FuneraryContext           ← burial group; many-to-many with Individual

DatedSample  ──>  exactly one of: ArchaeologicalContext | Bone | Skeleton
   └── DatingResult ──> DatingTechnique

BibliographicReference (entity Reference)
   ├── many-to-many with Site
   └── many-to-many with ArchaeologicalContext

Culture
   ├── one-to-many CultureFeature
   └── nullable on ArchaeologicalContext
```

The hierarchy is intentionally narrow: each child has a single required parent,
with funerary membership and bibliography being the only many-to-many relations.

> **`Individual` ↔ `Bone` is many-to-many** (the v2 `bone_individual` bridge).
> The tree nests bones under individuals for readability, but a single `Bone`
> record may be associated with one *or several* `Individual` records and is not
> exclusively owned by any of them. `Skeleton` remains strictly 1:1 with
> `Individual`. Aggregate bone counts must deduplicate by `boneId`
> (`COUNT(DISTINCT bone.bone_id)`) — see [04-database.md](./04-database.md).

The four research access dimensions — geographic (sites/map), biological
(individuals), chronological (timeline/cultures), and anatomical (bones) — map
onto the public web routes documented in
[frontend/02-public-web.md](./frontend/02-public-web.md).

## Active v2 entities

### Site

A geographic locality with georeferenced coordinates. Required: `siteName`
(unique), `country`, `region`, `municipality`, `latitude`, `longitude`.
Optional `description`. The response exposes `siteId`, `siteName`, `country`,
`region`, `municipality`, `latitude`, `longitude`, `updatedAt`.

`Site` carries **no aggregate count fields**. Site-level counts (individuals,
contexts, dated samples) come from `/api/stats/map-timeline` or
`/api/bone-site-search`, not from the site response.

Relationships:

- one-to-many `ArchaeologicalContext` (cascade delete via DB).
- many-to-many `Reference` through `site_bibliographic_reference`.

### Culture and CultureFeature

`Culture` is an archaeological technocomplex (Aurignacian, Gravettian,
Magdalenian, …). It carries `cultureName` (unique), a `phase` enum, a BP range
(`startBp >= endBp`, enforced by DB and `CultureService`), display colours
(`color`, `colorRgb` — authoritative for timeline/map), optional `region`, and
optional `description`. `features` is a list of repeatable textual
characteristics (`CultureFeature`, one-to-many).

A `Culture` may classify many `ArchaeologicalContext` rows. The embedded form
(`CultureEmbedResponse`) omits `region`, `description`, and `updatedAt`.

### ArchaeologicalContext

The stratigraphic and cultural slice within a `Site`. Replaces the v1
`OsteologicalUnit`. Required: `site` (FK), `stratigraphicContext`. Optional
`culture` FK.

It does **not** carry MNI, burial info, year of discovery, specimen labels, or
bone-processing flags — those moved to `Individual`, `FuneraryContext`, `Bone`,
and `Skeleton`:

- `unitType`, `mni`, `mniStatistical` → now on `Individual`.
- `burialContext`, `burialType`, `graveGoods`, `tracesOfOchre`,
  `boneProcessing` → now on `FuneraryContext` (`ochrePresence` replaces
  `tracesOfOchre`).
- `yearDiscovery` → now on `Bone` / `Skeleton`.

Relationships:

- many belong to one `Site`; many may share one `Culture` (nullable, `ON DELETE
  SET NULL`).
- one has many `Individual`, many `FuneraryContext`, and (when
  `sampleOrigin = CONTEXT`) many `DatedSample`.
- many-to-many `Reference` through
  `archaeological_context_bibliographic_reference`.

There is **no standalone public route** for archaeological contexts; the
frontend renders them only inside parent views.

### Individual

The central anthropological record. Every `Individual` belongs to exactly one
`ArchaeologicalContext`. Fields:

- `individualName` (nullable label such as "Paviland 1");
- `individualType`: `INDIVIDUAL` | `MIXED_INDIVIDUALS` | `UNASSIGNED_REMAINS`;
- `mni` (text — `?` or a positive-integer string);
- `mniStatistical` (read-only, DB-generated integer);
- `ageAtDeathText`, `ageAtDeathMin`, `ageAtDeathMax`, `ageUnit`,
  `ageClassMain`, `ageClassSubcategory`;
- `sex`, `sexCertain` (when `false`, the UI appends "?", e.g. `Female?`).

`individualType` encodes what the row represents:

- `INDIVIDUAL` — one named or identifiable individual.
- `MIXED_INDIVIDUALS` — remains that cannot be separated cleanly.
- `UNASSIGNED_REMAINS` — remains not confidently assigned to a specific
  individual.

Service-level invariants (see
[backend/03-validation-and-errors.md](./backend/03-validation-and-errors.md)):

- `INDIVIDUAL` requires `mni == "1"`;
- `MIXED_INDIVIDUALS` requires `mni != "1"`;
- `UNASSIGNED_REMAINS` accepts any valid mni;
- `mni` must match `^\?$|^[1-9][0-9]*$`;
- if both ages are present, `ageAtDeathMin <= ageAtDeathMax`.

Relationships:

- many belong to one `ArchaeologicalContext`.
- many-to-many with `Bone` through `bone_individual`.
- one-to-many `Skeleton` (a skeleton is 1:1 with its individual).
- many-to-many with `FuneraryContext` through `funerary_context_individual`.

A lightweight projection, `IndividualSummary` (returned by
`GET /api/individuals/list`), carries only the columns the public list page
renders.

### BoneCatalog

Controlled anatomical vocabulary. Each row is one anatomical label with
`boneCatalogName` (unique), `skeletonMain` (`CRANIAL` | `POSTCRANIAL`),
`skeletonRegion`, `boneCategory` (`TOOTH` | `VERTEBRA` | `RIB` | `PHALANX` |
`OTHER`), `termType` (`ATOMIC` | `GROUPED` | `VAGUE_STRUCTURED` |
`VAGUE_GENERIC`), and a self-referential `parentBoneCatalogId`.

Compositional decomposition of grouped terms (e.g. `cranium → neurocranium +
splanchnocranium`) is exposed through the read-only view
`bone_catalog_component`. `BoneCatalog` (vocabulary) is intentionally distinct
from `Bone` (observed data); never collapse one into the other.

> A legacy frontend service `BoneNameService` still references the removed
> `/api/bone-names` path and is unused; the active service is `BoneCatalogService`
> at `/api/bone-catalog` (cleanup tracked in
> [frontend/07-roadmap.md](./frontend/07-roadmap.md)).

### Bone

An observed per-element anatomical record. Required: at least one associated
`Individual` (through `bone_individual`), `boneCatalog` FK, `boneSource` (raw
source wording, non-blank), `boneQuantityMin` (DB check `>= 1`). Other fields:

- curatorial: `specimenName`, `repository`, `yearDiscovery`, `boneProcessing`
  (moved from the v1 `Specimen`);
- source quantity wording: `boneQuantitySource`;
- anatomical qualifiers: `laterality`, tooth/vertebra/phalanx/rib fields and
  their `_confidence` booleans, `handFootBoneSegment`, `boneDetails`.

The response exposes `individualIds: number[]` — the IDs of **every** associated
individual — not a single `individual`/`individualId`. The array is always
non-empty (a bone is linked to at least one individual, and may be linked to
several). In mixed assemblages, a bone that cannot be confidently attributed to
one person is linked to every plausible individual **without duplicating the
bone record**, so aggregate totals must count `COUNT(DISTINCT bone.bone_id)`.

> The pre-v2 field names `boneName` and `boneQuantity` no longer exist. Use
> `boneSource` and `boneQuantitySource`.

Relationships:

- many-to-many with `Individual` through `bone_individual`.
- many reference one `BoneCatalog` term (`ON DELETE RESTRICT`).
- one may have many `DatedSample` (when `sampleOrigin = BONE`).

### Skeleton

A whole-skeleton preservation assessment for one `Individual`. Required:
`individual` FK, `skeletonCategory` (enum). Optional: `preservationIndex` (DB
check `0..100`), curatorial fields (`specimenName`, `repository`,
`yearDiscovery`, `boneProcessing`), `description`.

`Skeleton` is **not** a list of bones — it is a curatorial preservation record
for the skeleton as a whole. Do not merge it into `Bone`.

Relationships:

- many belong to one `Individual`.
- one may have many `DatedSample` (when `sampleOrigin = SKELETON`).

> **Remains exclusivity.** An individual may have **one or more bones** **or**
> **one skeleton**, but never both, and at most one skeleton. This is enforced
> server-side (`RemainsExclusivityGuard`) plus DB constraints — see
> [backend/04-services-and-business-rules.md](./backend/04-services-and-business-rules.md).

### FuneraryContext

A burial/funerary group within an `ArchaeologicalContext`. Replaces the v1
`BurialGroup` and absorbs former `OsteologicalUnit` burial fields. Required:
`archaeologicalContext` FK, `burialContext` (`YES` | `NO` | `POSSIBLE` |
`UNCERTAIN`). Optional: `burialType` (primary/secondary × single/double/triple/
multiple), `ochrePresence`, `graveGoods`, `notes`.

`graveGoods` and `ochrePresence` are **Boolean flags**, not descriptive text.

Schema and service invariants:

- `burialContext = NO` → `burialType` must be null.
- `burialContext = YES` → `burialType` must be non-null.
- `POSSIBLE` / `UNCERTAIN` allow either.

A double burial is one `FuneraryContext` with two member individuals; a triple
has three. The DB does not enforce member counts against the burial type — that
is left to the service or backoffice UX.

Relationships:

- many belong to one `ArchaeologicalContext`.
- many-to-many with `Individual` through `funerary_context_individual`.

### DatedSample and DatingResult

The unified dating model. A `DatedSample` links a physical sample to **exactly
one** of an `ArchaeologicalContext` (`sampleOrigin = CONTEXT`), a `Bone`
(`BONE`), or a `Skeleton` (`SKELETON`). It carries `material` (required text)
and `datingType` (`DIRECT` | `INDIRECT` | `UNCERTAIN`), and has one or more
`DatingResult`s.

A `DatingResult` is the numeric output: `datesBpUncal` (uncalibrated BP),
`datesRange` (±sigma), `notes`, and a required `datingTechnique` FK.

Schema and service invariants:

- exactly one origin FK is non-null, and it must match `sampleOrigin`;
- `CONTEXT` may only have `datingType` `INDIRECT` or `UNCERTAIN`;
- `BONE` and `SKELETON` may only have `datingType` `DIRECT` or `UNCERTAIN`.

> **There is no calibrated BP range field anywhere in the model.** Only the
> uncalibrated BP value and its sigma exist. Any UI design that requires
> `calBpMin` / `calBpMax` requires a schema change.

`DatingTechnique` is a lookup table (AMS ¹⁴C, OSL, TL, AAR, …) with a unique
`datingTechniqueName`. `dating_result.dating_technique_id` uses `ON DELETE
RESTRICT`, so techniques in use cannot be deleted. Dating techniques are managed
only via the backoffice (`/admin/dating-techniques`).

### Reference (BibliographicReference)

Bibliographic entries. Required: `authors`, `year`, `title`, `journal`.
Optional: `volume`, `issue`, `pages`, `doi`, `url`, `publisher`, `notes`.

> The Java class is `Reference` and the table is `bibliographic_reference`. The
> API resource path is `/api/references` (paginated, default sort `year,desc`
> then `bibliographicReferenceId,asc`).

Relationships:

- many-to-many with `Site` through `site_bibliographic_reference`.
- many-to-many with `ArchaeologicalContext` through
  `archaeological_context_bibliographic_reference`.

There is no direct M2M between `Individual` and `Reference`; cite via context.

### AppUser and RefreshToken

Backoffice authentication entities. Not part of the archaeological domain — see
[backend/02-security-and-auth.md](./backend/02-security-and-auth.md).

## API DTO model vs. JPA entity model

API consumers see a flattened projection of the entity graph:

- Response DTOs use one-level-deep `*SummaryResponse` records to represent
  related entities; they never serialize JPA inverse collections.
- PATCH update DTOs use `JsonNullable<T>` so callers can distinguish "omitted"
  from "explicit null". Update mappers apply only the defined fields.
- Many-to-many sets (`FuneraryContext.individuals`, `Site.references`,
  `ArchaeologicalContext.references`, `Bone.individualIds`) are **replaced in
  their entirety** when the corresponding id list is present in a PATCH. There
  are no add-one / remove-one endpoints.

See [05-api-contract.md](./05-api-contract.md) for the full DTO map.

## Deprecated v1 names (do not reintroduce)

These names are removed from code and from the v2 schema. They appear here only
so future contributors recognise them and avoid recreating them.

| Removed v1 name | v2 replacement |
| --- | --- |
| `OsteologicalUnit` (table `osteological_unit`) | `ArchaeologicalContext` (stratigraphic/cultural data) + fields on `Individual` (MNI, biological data) + `FuneraryContext` (burial info) |
| `Specimen` (table `specimen`) | curatorial fields directly on `Bone` and `Skeleton` (`specimenName`, `repository`, `yearDiscovery`, `boneProcessing`) |
| top-level `Date` / `dates` table | split into `DatedSample` (origin/material) + `DatingResult` (numeric output) |
| `BurialGroup` (table `burial_group`) | `FuneraryContext` (+ `funerary_context_individual`) |
| any name of the form `ArchaeologicalUnit` | the canonical name is `ArchaeologicalContext` — do not introduce `ArchaeologicalUnit` |

The associated v1 bridge tables (`individual_osteological_unit`,
`osteological_unit_specimen`, `specimen_bone`,
`osteological_unit_bibliographic_reference`, `burial_group_osteological_unit`,
`burial_group_individual`) are also gone; the v2 equivalents are listed under
each entity above.

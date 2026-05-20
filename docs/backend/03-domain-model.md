# 03 — Domain Model (v2)

This document describes the **scientific/domain model** at the level
of Java entities and how they relate. Persistence-level details
(table names, columns, generated columns, triggers, indexes) live in
[04-database-model.md](./04-database-model.md).

## High-level hierarchy

```text
Site
  └── ArchaeologicalContext
        ├── Individual
        │     ├── Bone
        │     └── Skeleton
        └── FuneraryContext  (many-to-many with Individual)

DatedSample  ──>  exactly one of: ArchaeologicalContext | Bone | Skeleton
   └── DatingResult ──> DatingTechnique

BibliographicReference
   ├── many-to-many with Site
   └── many-to-many with ArchaeologicalContext

Culture
   ├── one-to-many CultureFeature
   └── nullable on ArchaeologicalContext
```

The hierarchy is intentionally narrow: each child has a single
required parent, with funerary membership and bibliography being the
only many-to-many relations. Detail response DTOs use shallow
`*SummaryResponse` records to avoid recursive nesting; see
[02-architecture.md](./02-architecture.md).

## Active v2 entities

### Site

Package: `site` · Entity: `Site` · Table: `site`.

A geographic locality. Required fields: `siteName` (unique),
`country`, `region`, `municipality`, `latitude`, `longitude`.
Optional `description`.

Relationships:

- one-to-many `ArchaeologicalContext` (cascade delete via DB).
- many-to-many `BibliographicReference` through
  `site_bibliographic_reference`.

### Culture and CultureFeature

Packages: `culture`, `culture_feature` ·
Entities: `Culture`, `CultureFeature` ·
Tables: `culture`, `culture_feature`.

`Culture` carries `cultureName` (unique), a `phase` enum, a BP
range (`startBp >= endBp` enforced by DB and by `CultureService`),
display colors, optional `region`, and optional `description`.

`CultureFeature` is a repeatable textual feature for a culture
(`feature` is the text). One culture has many features.

A `Culture` may classify many `ArchaeologicalContext` rows.

### ArchaeologicalContext

Package: `archaeological_context` ·
Entity: `ArchaeologicalContext` ·
Table: `archaeological_context`.

The stratigraphic/cultural slice within a site. Required fields:
`site` (required FK), `stratigraphicContext`. Optional `culture` FK.

Relationships:

- many `ArchaeologicalContext` belong to one `Site`.
- many `ArchaeologicalContext` may share one `Culture`.
- one `ArchaeologicalContext` has many `Individual`.
- one `ArchaeologicalContext` has many `FuneraryContext`.
- one `ArchaeologicalContext` may have many `DatedSample` rows
  (when `sampleOrigin = CONTEXT`).
- many-to-many `BibliographicReference` through
  `archaeological_context_bibliographic_reference`.

`ArchaeologicalContext` does **not** carry MNI, burial info, year of
discovery, specimen labels, or bone-processing flags. Those live on
`Individual`, `FuneraryContext`, `Bone`, and `Skeleton`.

### Individual

Package: `individual` · Entity: `Individual` · Table: `individual`.

The central anthropological record. Every `Individual` belongs to
exactly one `ArchaeologicalContext`.

Fields (see DTOs for the API shape):

- `individualName` (nullable label such as "Paviland 1");
- `individualType`: `INDIVIDUAL` | `MIXED_INDIVIDUALS` |
  `UNASSIGNED_REMAINS`;
- `mni` (text — `?` or a positive integer string);
- `mniStatistical` (read-only, DB-generated integer);
- `ageAtDeathText`, `ageAtDeathMin`, `ageAtDeathMax`, `ageUnit`,
  `ageClassMain`, `ageClassSubcategory`;
- `sex`, `sexCertain`.

An `Individual` row does not always mean a fully separated biological
person. The `individualType` enum encodes this:

- `INDIVIDUAL` — one named or identifiable individual.
- `MIXED_INDIVIDUALS` — remains that cannot be separated cleanly.
- `UNASSIGNED_REMAINS` — remains not confidently assigned to a
  specific individual.

Service-level invariants (see [07-validation-and-errors.md](./07-validation-and-errors.md)):

- `INDIVIDUAL` requires `mni == "1"`;
- `MIXED_INDIVIDUALS` requires `mni != "1"`;
- `UNASSIGNED_REMAINS` accepts any valid mni;
- `mni` must match `^\?$|^[1-9][0-9]*$`;
- if both ages are present, `ageAtDeathMin <= ageAtDeathMax`.

Relationships:

- many `Individual` belong to one `ArchaeologicalContext`.
- one `Individual` has many `Bone`.
- one `Individual` has many `Skeleton`.
- many-to-many with `FuneraryContext` through
  `funerary_context_individual`.

### BoneCatalog

Package: `bone_catalog` · Entity: `BoneCatalog` · Table: `bone_catalog`.

Controlled anatomical vocabulary. Each row is one anatomical label
with:

- `boneCatalogName` (unique label);
- `skeletonMain` (`CRANIAL` | `POSTCRANIAL`);
- `skeletonRegion` (e.g. `NEUROCRANIUM`, `UPPER_LIMB`);
- `boneCategory` (`TOOTH` | `VERTEBRA` | `RIB` | `PHALANX` | `OTHER`);
- `termType` (`ATOMIC` | `GROUPED` | `VAGUE_STRUCTURED` |
  `VAGUE_GENERIC`);
- self-referential `parentBoneCatalogId` for the hierarchy.

`parent_bone_catalog_id` is hierarchical (term/parent term).
Compositional decomposition of grouped terms is exposed through the
read-only view `bone_catalog_component` (e.g. `cranium → neurocranium
+ splanchnocranium`). See [04-database-model.md](./04-database-model.md).

### Bone

Package: `bone` · Entity: `Bone` · Table: `bone`.

An observed anatomical record on one individual. Required:
`individual` FK, `boneCatalog` FK, `boneSource` (raw source wording),
`boneQuantityMin` (DB check `>= 1`).

Other fields:

- curatorial: `specimenName`, `repository`, `yearDiscovery`,
  `boneProcessing` (boolean);
- source quantity wording: `boneQuantitySource`;
- anatomical qualifiers: `laterality`, tooth fields, vertebra fields,
  phalanx fields, rib number, `handFootBoneSegment`, `boneDetails`.

Relationships:

- many `Bone` belong to one `Individual`.
- many `Bone` reference one `BoneCatalog` term.
- one `Bone` may have many `DatedSample` rows
  (when `sampleOrigin = BONE`).

`Bone` is observational data. It is intentionally distinct from
`BoneCatalog` (vocabulary). Observed quantity is
`bone.bone_quantity_min`; source wording is
`bone.bone_quantity_source`. Do **not** use `bone_catalog_component`
component quantities as observed quantities.

### Skeleton

Package: `skeleton` · Entity: `Skeleton` · Table: `skeleton`.

A whole-skeleton preservation assessment for one individual.
Required: `individual` FK, `skeletonCategory` (enum). Optional:
`preservationIndex` (DB check `0..100`), curatorial fields
(`specimenName`, `repository`, `yearDiscovery`, `boneProcessing`),
`description`.

`Skeleton` is **not** a list of bones. It is a curatorial preservation
record for the skeleton as a whole.

Relationships:

- many `Skeleton` belong to one `Individual`.
- one `Skeleton` may have many `DatedSample` rows
  (when `sampleOrigin = SKELETON`).

### DatingTechnique

Package: `dating_technique` · Entity: `DatingTechnique` ·
Table: `dating_technique`.

Lookup table. `datingTechniqueName` is required and unique.
`dating_result.dating_technique_id` uses `ON DELETE RESTRICT`, so
techniques that are in use cannot be deleted.

### DatedSample

Package: `dated_sample` · Entity: `DatedSample` ·
Table: `dated_sample`.

The material/origin of a dating event. Each row references **exactly
one** of:

- `archaeologicalContext` (`sampleOrigin = CONTEXT`)
- `bone` (`sampleOrigin = BONE`)
- `skeleton` (`sampleOrigin = SKELETON`)

Other fields: `material` (required text, e.g. "human bone",
"charcoal", "sediment"), `datingType`
(`DIRECT` | `INDIRECT` | `UNCERTAIN`).

Schema and service invariants:

- exactly one origin FK is non-null;
- `sampleOrigin` must match the non-null FK;
- `CONTEXT` may only have `datingType` `INDIRECT` or `UNCERTAIN`;
- `BONE` and `SKELETON` may only have `datingType` `DIRECT` or
  `UNCERTAIN`.

Relationships:

- one `DatedSample` has many `DatingResult`.

### DatingResult

Package: `dating_result` · Entity: `DatingResult` ·
Table: `dating_result`.

Numeric output of a dating event. Required: `datedSample` FK,
`datingTechnique` FK. Optional: `datesBpUncal`, `datesRange`, `notes`.

### BibliographicReference

Package: `reference` · Entity: `Reference` ·
Table: `bibliographic_reference`.

Required: `authors`, `year`, `title`, `journal`. Optional: `volume`,
`issue`, `pages`, `doi`.

Relationships:

- many-to-many with `Site` through `site_bibliographic_reference`.
- many-to-many with `ArchaeologicalContext` through
  `archaeological_context_bibliographic_reference`.

> The Java class is `Reference` and the table is
> `bibliographic_reference`. API resource path is `/api/references`.

### FuneraryContext

Package: `funerary_context` · Entity: `FuneraryContext` ·
Table: `funerary_context`.

A funerary/burial group within an archaeological context. Required:
`archaeologicalContext` FK, `burialContext`
(`YES` | `NO` | `POSSIBLE` | `UNCERTAIN`). Optional: `burialType`
(primary/secondary × single/double/triple/multiple), `ochrePresence`,
`graveGoods`, `notes`.

Schema and service invariants:

- `burialContext = NO` → `burialType` must be null.
- `burialContext = YES` → `burialType` must be non-null.
- `POSSIBLE` and `UNCERTAIN` allow either.

Relationships:

- many `FuneraryContext` belong to one `ArchaeologicalContext`.
- many-to-many with `Individual` through
  `funerary_context_individual`.

A double burial is one `FuneraryContext` with two member individuals;
a triple burial has three. The database does not enforce member
counts against the burial type — that is left to the service or to
backoffice UX.

### AppUser and RefreshToken

Packages: `user`, `refresh_token` · Entities: `AppUser`,
`RefreshToken`. Not part of the archaeological domain — see
[06-security-and-auth.md](./06-security-and-auth.md).

## Deprecated v1 names (do not reintroduce)

The following names are removed from code and from the v2 schema.
They appear here only so that future agents recognize them and avoid
recreating them.

| Removed v1 name | v2 replacement |
| --- | --- |
| `OsteologicalUnit` (table `osteological_unit`) | `ArchaeologicalContext` (stratigraphic/cultural data) + fields on `Individual` (MNI, biological data) + `FuneraryContext` (burial info) |
| `Specimen` (table `specimen`) | curatorial fields directly on `Bone` and `Skeleton` (`specimenName`, `repository`, `yearDiscovery`, `boneProcessing`) |
| top-level `Date` / `dates` table | split into `DatedSample` (origin/material) + `DatingResult` (numeric output) |
| `BurialGroup` (table `burial_group`) | `FuneraryContext` |
| any name with the form `ArchaeologicalUnit` | the canonical name is `ArchaeologicalContext` — do not introduce `ArchaeologicalUnit` |

The associated v1 bridge tables (`individual_osteological_unit`,
`osteological_unit_specimen`, `specimen_bone`,
`osteological_unit_bibliographic_reference`,
`burial_group_osteological_unit`, `burial_group_individual`) are also
gone. The v2 equivalents are listed under each section above.

## API DTO model vs. JPA entity model

API consumers see a flattened projection of the entity graph.
Specifically:

- Response DTOs use `common/summary/*` records (one level deep) to
  represent related entities. They never serialize JPA inverse
  collections such as `Site.archaeologicalContexts`.
- PATCH update DTOs use `JsonNullable<T>` so callers can distinguish
  "omitted" from "explicit null". Update mappers apply only the
  defined fields.
- Many-to-many sets (e.g. `FuneraryContext.individuals`,
  `Site.references`, `ArchaeologicalContext.references`) are replaced
  in their entirety when the corresponding id list is present in a
  PATCH request. There are no add-one / remove-one endpoints.

See [05-api-contract.md](./05-api-contract.md) for the full DTO map.

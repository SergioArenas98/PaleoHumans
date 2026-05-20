# 03 — Domain Model

This is the canonical description of PaleoHumans entities **as the frontend sees them**. For the underlying database schema, modelling principles, and the raw SQL DDL, see [11-database.md](./11-database.md).

> The platform is on the **v2 model**. The v1 entities `OsteologicalUnit`, `Specimen`, `BurialGroup`, and `Date` have been replaced by `ArchaeologicalContext`, `Individual` (now owning MNI fields), `FuneraryContext`, `DatedSample`, and `DatingResult`. Anything still referencing the v1 names in old comments or branch redirects is deprecated.

---

## Entity hierarchy

```text
Site                              ← primary geographic entity
└── ArchaeologicalContext         ← stratigraphic + cultural slice within a site
    ├── Individual(s)             ← anthropological record (1, mixed, or unassigned remains)
    │   ├── Bone(s)               ← per-element record with curatorial fields
    │   └── Skeleton(s)           ← whole-skeleton preservation record
    ├── FuneraryContext(s)        ← burial group, M2M with Individual
    └── Reference(s)              ← cited literature, M2M

DatedSample                       ← exactly one of: ArchaeologicalContext | Bone | Skeleton
└── DatingResult(s)               ← BP value, ±sigma, technique, notes
```

Supporting classification entities (not part of the hierarchy):

- `Culture` — archaeological technocomplex (e.g., Aurignacian, Gravettian, Magdalenian).
- `BoneCatalog` (`/api/bone-catalog`) — canonical anatomical reference data.
- `DatingTechnique` (`/api/dating-techniques`) — controlled list of dating methods.

---

## Site

A geographic locality with georeferenced coordinates.

| Field | Type | Display | Notes |
|---|---|---|---|
| `siteId` | Integer | Internal | Routing key. |
| `siteName` | String | Primary | Page title, map marker label. |
| `country` | String | Primary | Filter dimension. |
| `region` | String | Secondary | — |
| `municipality` | String | Secondary | — |
| `latitude` | Double | Primary | WGS84. |
| `longitude` | Double | Primary | WGS84. |
| `updatedAt` | Instant | Tertiary | Admin metadata. |

`SiteResponse` contains **no aggregate count fields**. Site-level counts (individuals, contexts, dated samples) come from `/api/stats/map-timeline` or `/api/bone-site-search`, not from `SiteResponse`.

The public site detail page renders archaeological contexts for the site via `GET /api/archaeological-contexts?siteId=`. The backoffice exposes a list (`/admin/sites`) and a per-site reference link/unlink UI.

---

## ArchaeologicalContext

The stratigraphic and cultural slice within a Site. Replaces the v1 `OsteologicalUnit`.

| Field | Type | Notes |
|---|---|---|
| `archaeologicalContextId` | Integer | Routing key. |
| `site` | embedded SiteSummary | Includes `siteId`, `siteName`, `country`, `region`, `municipality`, lat/lng. |
| `stratigraphicContext` | String \| null | Layer designation (e.g. "Layer C", "Couche IV"). |
| `culture` | embedded CultureSummary \| null | `cultureId`, `cultureName`, `phase`, `startBp`, `endBp`, `color`, `colorRgb`, `features[]`. |
| `references` | ReferenceSummary[] | Linked bibliography. |
| `individuals` | IndividualSummary[] | All individuals attributed to this context. |
| `funeraryContexts` | FuneraryContextSummary[] | Burial groupings inside this context. |
| `datedSamples` | DatedSampleSummary[] (optional) | Embedded if the backend response includes them. |
| `updatedAt` | Instant | — |

What ArchaeologicalContext **does not** contain (moved out from v1 OsteologicalUnit):

- `unitType`, `mni`, `mniStatistical` → now on `Individual`.
- `burialContext`, `burialType`, `graveGoods`, `tracesOfOchre`, `boneProcessing` → now on `FuneraryContext` (with `ochrePresence` replacing `tracesOfOchre`).
- `yearDiscovery` → now on `Bone` / `Skeleton` (curatorial field).

There is **no standalone public route** for archaeological contexts. The frontend renders them only inside parent views (`/sites/:id`, `/individuals/:id`, `/bones`).

---

## Individual

The anthropological record attached to one ArchaeologicalContext. Now the primary biological record.

| Field | Type | Notes |
|---|---|---|
| `individualId` | Integer | Routing key. |
| `individualName` | String \| null | Archaeological label (e.g., "Paviland 1", "Cro-Magnon 1"). |
| `individualType` | `INDIVIDUAL` \| `MIXED_INDIVIDUALS` \| `UNASSIGNED_REMAINS` | Enum. |
| `mni` | String \| null | Free text, may be "ca. 2" or numeric. |
| `mniStatistical` | Integer \| null | Statistical MNI estimate. |
| `ageAtDeathText` | String \| null | "25–35 years", etc. |
| `ageAtDeathMin`, `ageAtDeathMax` | Integer \| null | Numeric bounds. |
| `ageUnit` | `YEARS` \| `MONTHS` \| null | Enum. |
| `ageClassMain` | `SUBADULT` \| `ADULT` \| `INDETERMINATE` \| `SUBADULT_ADULT` \| null | Enum. |
| `ageClassSubcategory` | enum \| null | `FETAL`, `PERINATAL`, `INFANT_I`, `INFANT_II`, `JUVENILE`, `YOUNG_ADULT`, `MATURE_ADULT`, `SENILE_ADULT`. |
| `sex` | `MALE` \| `FEMALE` \| `INDETERMINATE` \| null | Enum. |
| `sexCertain` | Boolean \| null | When `false`, formatted UI appends "?" (e.g. `Female?`). |
| `archaeologicalContext` | embedded summary \| null | Includes site, culture, stratigraphy. |
| `bones`, `skeletons`, `funeraryContexts`, `datedSamples` | embedded summaries (optional) | Backend may include these on the detail endpoint; otherwise the frontend enriches via dedicated calls. |
| `updatedAt` | Instant | — |

`IndividualSummary` (returned by `GET /individuals/list`) is a lightweight projection used by the public individuals list page. It carries only the columns the list renders (`individualId`, `individualName`, `sex`, `sexCertain`, age fields, `siteId`, `siteName`, `country`, `cultureId`, `cultureName`, `cultureColor`, `cultureColorRgb`).

---

## Bone

A per-element bone record attached to a single Individual. The v1 `Specimen → Bone` link is gone; bones now reference `individualId` directly, and curatorial fields previously stored on `Specimen` live here.

| Field | Type | Notes |
|---|---|---|
| `boneId` | Integer | — |
| `boneCatalog` | BoneCatalogSummary | `boneCatalogId`, `boneCatalogName`, `skeletonMain`, `skeletonRegion`, `boneCategory`. |
| `individual` | IndividualSummary \| null | Slim embed with `archaeologicalContextId`, `siteId`, `siteName`. |
| `boneSource` | String | **Required**, `@NotBlank`. Free-text source descriptor. |
| `boneQuantitySource` | String \| null | Free-text quantity descriptor. |
| `boneQuantityMin` | Integer | `@Positive`, required. Normalized minimum count. |
| `specimenName`, `repository`, `yearDiscovery` | String \| null | Curatorial fields (moved from v1 Specimen). |
| `boneProcessing` | Boolean \| null | Taphonomic flag. |
| `laterality` | enum \| null | `LEFT`, `RIGHT`, `BILATERAL`, `INDETERMINATE`. |
| `toothType` / `toothVerticalPosition` / `toothNumber` / `toothNumberConfidence` | enum / Integer / Boolean | Dental fields. |
| `vertebraType` / `vertebraNumber` / `vertebraNumberConfidence` | enum / Integer / Boolean | Vertebral fields. |
| `phalanxType` / `phalanxNumber` / `phalanxNumberConfidence` / `handFootBoneSegment` | enum / Integer / Boolean / enum | Hand/foot fields. |
| `ribNumber` / `ribNumberConfidence` | Integer / Boolean | Costal fields. |
| `boneDetails` | String \| null | Notes from literature. |
| `datedSamples` | BoneDatedSampleSummary[] (optional) | Embedded when present. |
| `updatedAt` | Instant | — |

> The pre-v2 field names `boneName` and `boneQuantity` no longer exist. Always use `boneSource` and `boneQuantitySource`.

---

## Skeleton

A whole-skeleton preservation record attached to one Individual.

| Field | Type | Notes |
|---|---|---|
| `skeletonId` | Integer | — |
| `individual` | IndividualSummary \| null | Slim embed. |
| `skeletonCategory` | SkeletonCategory enum | — |
| `preservationIndex` | Number \| null | API (preservation) value. |
| `specimenName`, `repository`, `yearDiscovery` | String \| null | Curatorial fields (moved from v1 Specimen). |
| `boneProcessing` | Boolean \| null | Same taphonomic flag as `Bone.boneProcessing`. |
| `description` | String \| null | Free text. |
| `datedSamples` | SkeletonDatedSampleSummary[] (optional) | Embedded when present. |
| `updatedAt` | Instant | — |

---

## FuneraryContext

Burial group within an ArchaeologicalContext. Replaces the v1 `BurialGroup` and absorbs former `OsteologicalUnit` burial fields. Many-to-many with `Individual` through `funerary_context_individual`.

| Field | Type | Notes |
|---|---|---|
| `funeraryContextId` | Integer | — |
| `archaeologicalContext` | embedded summary \| null | Includes `siteId`, `siteName`, `cultureId`, `cultureName`. |
| `burialContext` | BurialContext enum \| null | e.g. `YES`, `NO`, `POSSIBLE`, `UNCERTAIN`. |
| `burialType` | BurialType enum \| null | — |
| `ochrePresence` | Boolean \| null | Renames the v1 `tracesOfOchre` flag. |
| `graveGoods` | Boolean \| null | Presence/absence flag. |
| `notes` | String \| null | Free text. |
| `individuals` | FuneraryContextIndividualMember[] | M2M members; each carries `individualId`, name, type, sex, age fields. |
| `updatedAt` | Instant | — |

> `graveGoods` and `ochrePresence` are **Boolean flags**, not descriptive text. The UI can only display presence/absence.

---

## DatedSample and DatingResult

The unified dating model. A `DatedSample` links a physical sample to exactly one of: an `ArchaeologicalContext`, a `Bone`, or a `Skeleton`. A `DatedSample` carries one or more `DatingResult`s.

### DatedSample

| Field | Type | Notes |
|---|---|---|
| `datedSampleId` | Integer | — |
| `sampleOrigin` | SampleOrigin enum | Indicates which entity link is used. |
| `archaeologicalContext` \| `bone` \| `skeleton` | embedded summary | Exactly one populated. |
| `material` | String \| null | "Bone collagen", "Charcoal", etc. |
| `datingType` | DatingType enum | — |
| `datingResults` | DatingResult[] | Embedded; mirrors `DatingResultResponse` exactly. |
| `updatedAt` | Instant | — |

### DatingResult

| Field | Type | Notes |
|---|---|---|
| `datingResultId` | Integer | — |
| `datedSample` | summary | Reverse link. |
| `datingTechnique` | embedded `{ datingTechniqueId, datingTechniqueName }` | — |
| `datesBpUncal` | Number \| null | **Uncalibrated BP** date. |
| `datesRange` | Number \| null | ±sigma. |
| `notes` | String \| null | — |
| `updatedAt` | Instant | — |

> **There is no calibrated BP range field anywhere in the API.** Only the uncalibrated BP value and its sigma exist. Any UI design that requires `calBpMin` / `calBpMax` requires a backend schema change.

`GET /api/dated-samples` accepts the filter `archaeologicalContextId`, `boneId`, `skeletonId`, plus `sampleOrigin` and `datingType`. The endpoint is **public**. `GET /api/dating-results` is also public.

> **2026-05 note.** The v1 endpoints `/api/dates` and `/api/dating-techniques` were ADMIN-only and dating data was exposed publicly only as `Specimen.dates[]`. In v2 the public surface uses `/api/dated-samples` and `/api/dating-results`; verify the current public/admin classification against the live backend before assuming. `Requires code verification.`

---

## Culture

Reference data for archaeological technocomplexes.

| Field | Type | Notes |
|---|---|---|
| `cultureId` | Integer | — |
| `cultureName` | String | "Aurignacian", "Gravettian", "Magdalenian", etc. |
| `phase` | CulturePhase enum | `EARLY_UP`, `FULL_UP`, `FINAL_UP`. |
| `startBp`, `endBp` | Integer | Chronological bounds (BP). |
| `color`, `colorRgb` | String | Authoritative visual identifiers for timeline/map. |
| `region` | String \| null | Geographic range. |
| `description` | String \| null | Editorial narrative (timeline cards). |
| `features` | String[] | Key characteristics. |
| `updatedAt` | Instant | — |

The compact form (`CultureEmbedResponse`) embedded in other entities omits `region`, `description`, and `updatedAt`.

---

## Reference

Bibliographic entries.

| Field | Type | Notes |
|---|---|---|
| `referenceId` | Integer | — |
| `authors` | String | — |
| `year` | Integer \| null | — |
| `title` | String \| null | — |
| `journal` | String \| null | — |
| `volume`, `issue`, `pages` | String \| null | — |
| `doi` | String \| null | External link. |
| `url` | String \| null | — |
| `publisher` | String \| null | — |
| `notes` | String \| null | — |
| `updatedAt` | Instant | — |

`GET /api/references` is paginated (default sort `year,desc` then `bibliographicReferenceId,asc`). The public `/bibliography` page consumes the paginated envelope; do not expect a bare array.

---

## BoneCatalog

Anatomical reference data: canonical bone names, classifications, hierarchical parent/component relations.

`GET /api/bone-catalog` accepts `q`, `skeletonMain`, `skeletonRegion`, `boneCategory`, `termType`, `parentBoneCatalogId`.

> A legacy frontend service `BoneNameService` (`projects/web/src/app/features/bone-names/services/bone-name-service.ts`) still references the old `/api/bone-names` path and is unused. The active backoffice service is `BoneCatalogService` at `/api/bone-catalog`. The unused public-web service is candidate for cleanup (see [10-roadmap.md](./10-roadmap.md)).

---

## DatingTechnique

Controlled list of dating methods (e.g. AMS ¹⁴C, OSL, TL, AAR).

| Field | Type | Notes |
|---|---|---|
| `datingTechniqueId` | Integer | — |
| `datingTechniqueName` | String | — |
| `description` | String \| null | — |

Managed only via the backoffice (`/admin/dating-techniques`).

---

## Information architecture (research access paths)

| Audience | Entry path |
|---|---|
| Researcher (site-first) | Sites → Site detail (contexts + individuals + references) |
| Researcher (individual-first) | Individuals → Individual detail (`/individuals/:id`) |
| Researcher (chronological) | Timeline → click site dot → Site detail |
| Researcher (bone-first) | `/bones` (site-first aggregated bone search) → Site detail |
| Teacher / Student | Home → Map → Sites → Site detail |
| General public | Home → Map → Timeline (culture cards) → About |

There is no standalone public route for `ArchaeologicalContext`, `Bone` individual records, `Skeleton` individual records, `FuneraryContext`, `DatedSample`, or `Culture` (`/cultures/:id` does not exist on the public web — open gap; see [10-roadmap.md](./10-roadmap.md)).

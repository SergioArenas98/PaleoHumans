# 04 — Database

> **Canonical database reference.** This is the single source of truth for the
> PostgreSQL structure that powers PaleoHumans: tables, columns, enums,
> constraints, triggers, indexes, JPA mappings, and modelling principles. The
> conceptual entity model lives in [03-domain-model.md](./03-domain-model.md);
> the REST DTO shapes live in [05-api-contract.md](./05-api-contract.md). When
> the three disagree, the running backend response wins, then the database,
> then the narrative docs.

## 1. Purpose

`paleohumans` is a PostgreSQL database for storing, normalising, and serving
structured information about **European Upper Palaeolithic human remains**. It
is the upstream source of truth for the Spring Boot REST API, the public Angular
`web` app, and the Angular `backoffice` admin app.

The schema encodes more than storage — it encodes **curation decisions** about
how palaeoanthropological evidence is represented: where MNI lives, what
"individual" can mean, how dating attaches to a sample, and how burials group
individuals together.

## 2. Authoritative artifact and schema management

- **Authoritative schema:** `backend/src/main/resources/db/database.sql`. This
  is what is applied to a real database and what Hibernate validates against.
- **Reference copy:** [`database/schema.sql`](./database/schema.sql) in this
  docs tree, kept for convenient reference next to the documentation.

> **Requires verification.** These two files are intended to be kept in sync but
> have **drifted**: the docs reference copy currently carries an extra
> `chk_individual_type_mni_consistency` CHECK on `individual` that the backend
> copy does not. Reconcile them so the reference copy matches the authoritative
> backend schema (and confirm which constraints the live database actually
> enforces). Do not treat the two files as byte-identical until this is
> resolved.

**Schema management is manual.** `dev` and `production` use
`spring.jpa.hibernate.ddl-auto=validate`: Hibernate validates the schema on
startup and refuses to mutate it. There is no Flyway or Liquibase configuration;
ad-hoc migration scripts live in `backend/src/main/resources/db/migrations/`.
The `test` profile uses `ddl-auto=none` with H2 in PostgreSQL compatibility
mode.

Stack:

- PostgreSQL 13+ in `dev` and `production`.
- Hibernate 6 via Spring Data JPA.
- PostgreSQL named enum types bound through `@JdbcTypeCode(SqlTypes.NAMED_ENUM)`.
- H2 in PostgreSQL compatibility mode in `test`. H2 cannot fully validate
  PostgreSQL named enums, generated columns, and the `bone_catalog_component`
  view, so persistence tests for those features should run against real
  PostgreSQL (Testcontainers) — see
  [backend/07-roadmap.md](./backend/07-roadmap.md).

## 3. v2 model

The schema is on **v2**. The active hierarchy:

```text
site
  → archaeological_context
      → individual
          → bone        (many-to-many via bone_individual)
          → skeleton
      → funerary_context
          ↔ individual              (via funerary_context_individual)
      ↔ bibliographic_reference     (via archaeological_context_bibliographic_reference)

dated_sample (exactly one of: archaeological_context | bone | skeleton)
  → dating_result → dating_technique

site ↔ bibliographic_reference      (via site_bibliographic_reference)
culture → culture_feature
```

Removed v1 tables — **must not be recreated** in backend or frontend models:

```text
osteological_unit        burial_group
specimen                 burial_group_osteological_unit
date                     burial_group_individual
individual_osteological_unit
osteological_unit_specimen
specimen_bone
osteological_unit_bibliographic_reference
```

Replacement map:

```text
osteological_unit → archaeological_context + individual + funerary_context
specimen          → curatorial fields on bone/skeleton + dated_sample
old date          → dated_sample + dating_result
burial_group      → funerary_context + funerary_context_individual
```

See [03-domain-model.md](./03-domain-model.md) for the conceptual replacement
detail.

## 4. Subsystems

The database divides into small, independently reasoned subsystems joined
through a few well-defined FKs.

### 4.1 Sites and archaeological contexts

`site` (geographic locality, georeferenced, unique `site_name`) and
`archaeological_context` (stratigraphic/cultural slice inside a site; FKs
`site_id` required, `culture_id` nullable). **No aggregate counts** are stored
here; per-site counts come from joins or the dedicated stats endpoints.

### 4.2 Individuals

`individual` — central anthropological record (FK `archaeological_context_id`
required). Owns MNI, age-at-death, sex, age class, and `individual_type`.

`individual_type_enum`:

- `INDIVIDUAL` — one identifiable person; requires `mni = '1'`.
- `MIXED_INDIVIDUALS` — multiple individuals that cannot be cleanly separated;
  requires `mni <> '1'`.
- `UNASSIGNED_REMAINS` — remains not confidently assignable; any valid MNI.

MNI is stored as **text** to preserve raw curation wording (`'?'`, `'2'`, …) and
exposed as a generated integer `mni_statistical` for stats and filtering.

### 4.3 Bones and skeletons

- `bone_catalog` — controlled anatomical vocabulary, self-referential
  (`parent_bone_catalog_id`). Seed rows use **stable explicit IDs**.
- `bone_catalog_component` — a **view** (not a table) listing fixed anatomical
  decompositions for grouped terms (`cranium → neurocranium + splanchnocranium`,
  `coxal → ilium + ischium + pubis`, …). Composition is a fixed fact, not
  observational data, which is why it lives in view form.
- `bone` — observed per-element record (FK `bone_catalog_id`, `ON DELETE
  RESTRICT`). Has **no** `individual_id` column; its link to individuals is the
  M2M `bone_individual` bridge. Stores anatomical detail, curatorial fields, and
  quantity (`bone_quantity_source` raw wording, `bone_quantity_min` reliable
  count).
- `bone_individual` — M2M bridge between `bone` and `individual` (PK `(bone_id,
  individual_id)`, both FKs `ON DELETE CASCADE`). A bone may be associated with
  several individuals without being duplicated.
- `skeleton` — whole-skeleton preservation assessment (FK `individual_id`, a
  single required FK — skeleton is 1:1 with individual). Stores
  `skeleton_category`, `preservation_index` (0–100), and the same curatorial
  fields as `bone`.

`bone` and `skeleton` are intentionally **separate**; the controlled
duplication of curatorial fields is preferred over a polymorphic
`osteological_record` table.

### 4.4 Funerary contexts

`funerary_context` (burial-level grouping inside an `archaeological_context`;
carries `burial_context`, `burial_type`, `ochre_presence`, `grave_goods`,
`notes`) and `funerary_context_individual` (M2M bridge modelling
double/triple/multiple burials). Constraint: `burial_context = 'NO'` ⇒
`burial_type` null; `'YES'` ⇒ `burial_type` non-null.

### 4.5 Dating

- `dating_technique` — lookup of methods (unique by name; `ON DELETE RESTRICT`
  from `dating_result`).
- `dated_sample` — *what was dated*: `sample_origin` (`CONTEXT` | `BONE` |
  `SKELETON`), one of three nullable origin FKs, `material`, `dating_type`
  (`DIRECT` | `INDIRECT` | `UNCERTAIN`).
- `dating_result` — *the numeric outcome*: `dates_bp_uncal`, `dates_range`
  (sigma), `notes`, FK to `dating_technique`.

Two constraint pairs guard the model: exactly one origin FK non-null and
matching `sample_origin`; and `CONTEXT` ⇒ `INDIRECT`/`UNCERTAIN`,
`BONE`/`SKELETON` ⇒ `DIRECT`/`UNCERTAIN`. **There is no calibrated BP column** —
only `dates_bp_uncal` ± `dates_range`.

### 4.6 Cultures

`culture` (technocomplex; `phase`, `start_bp ≥ end_bp`, visualisation `color` /
`color_rgb`) and `culture_feature` (one-to-many textual features).
`archaeological_context.culture_id` is `ON DELETE SET NULL`.

### 4.7 Bibliography

`bibliographic_reference` plus two M2M bridges: `site_bibliographic_reference`
and `archaeological_context_bibliographic_reference`. A reference can attach at
both site and context level; there is no direct `individual`↔reference link.

### 4.8 Authentication

`app_user` (backoffice account) and `refresh_token` (refresh-token metadata for
JWT sessions). These are **not part of the archaeological domain**, are never
exported by `/api/dataset/download`, and connect to no archaeological table.

## 5. Tables and JPA mappings

### `site` (`site/Site.java`)

| Column | Type | Notes |
| --- | --- | --- |
| `site_id` | INTEGER PK GENERATED BY DEFAULT AS IDENTITY | |
| `site_name` | TEXT NOT NULL UNIQUE | |
| `country`, `region`, `municipality` | TEXT NOT NULL | |
| `latitude`, `longitude` | DOUBLE PRECISION NOT NULL | |
| `description` | TEXT | |
| `updated_at` | TIMESTAMPTZ NOT NULL DEFAULT now() | trigger-maintained |

### `culture` / `culture_feature`

`culture` (`culture/Culture.java`): `culture_id` PK; `culture_name` TEXT NOT NULL
UNIQUE; `phase` `culture_phase_enum` NOT NULL; `start_bp`, `end_bp` INTEGER NOT
NULL (CHECK `start_bp >= end_bp`); `color`, `color_rgb` TEXT NOT NULL; `region`,
`description` TEXT; `updated_at` (trigger).

`culture_feature` (`culture_feature/CultureFeature.java`): `culture_feature_id`
PK; `culture_id` FK → `culture` ON DELETE CASCADE; `feature` TEXT NOT NULL;
`updated_at` (trigger).

### `archaeological_context` (`archaeological_context/ArchaeologicalContext.java`)

`archaeological_context_id` PK; `site_id` FK → `site` ON DELETE CASCADE
(required); `stratigraphic_context` TEXT NOT NULL; `culture_id` FK → `culture`
ON DELETE SET NULL (nullable); `updated_at` (trigger).

### `individual` (`individual/Individual.java`)

`individual_id` PK; `archaeological_context_id` FK ON DELETE CASCADE (required);
`individual_name` TEXT; `individual_type` `individual_type_enum` NOT NULL; `mni`
TEXT NOT NULL DEFAULT `'1'`; `mni_statistical` INTEGER **GENERATED ALWAYS AS …
STORED**; `age_at_death_text` TEXT; `age_at_death_min`, `age_at_death_max`
INTEGER; `age_unit` `age_unit_enum`; `age_class_main` `age_class_main_enum` NOT
NULL; `age_class_subcategory` `age_class_subcategory_enum`; `sex` `sex_enum` NOT
NULL; `sex_certain` BOOLEAN NOT NULL; `updated_at` (trigger).

Constraints: `chk_mni_format` (`mni = '?' OR mni ~ '^[1-9][0-9]*$'`);
`chk_age_range` (if both bounds present, `min <= max`);
`chk_individual_type_mni_consistency` (`INDIVIDUAL` ⇒ `mni = '1'`;
`MIXED_INDIVIDUALS` ⇒ `mni <> '1'`; `UNASSIGNED_REMAINS` ⇒ no extra constraint).
*(See the verification note in §2 — confirm this CHECK is present in the
authoritative schema.)*

Generated column:

```sql
mni_statistical INTEGER GENERATED ALWAYS AS (
  CASE
    WHEN mni ~ '^[1-9][0-9]*$' THEN mni::INTEGER
    WHEN mni = '?' THEN 1
    ELSE 1
  END
) STORED
```

JPA maps it `insertable = false, updatable = false`. Never write to it.

### `bone_catalog` (`bone_catalog/BoneCatalog.java`)

`bone_catalog_id` PK — **seed data must use stable explicit IDs** because the
`bone_catalog_component` view references fixed ids; `bone_catalog_name` TEXT NOT
NULL UNIQUE; `skeleton_main` `skeleton_main_enum`; `skeleton_region`
`skeleton_region_enum`; `bone_category` `bone_category_enum` NOT NULL;
`term_type` `bone_term_type_enum` NOT NULL DEFAULT `'ATOMIC'`;
`parent_bone_catalog_id` FK → `bone_catalog` ON DELETE SET NULL.

### `bone_catalog_component` (view)

A `CREATE OR REPLACE VIEW` of hard-coded `(bone_catalog_id,
component_bone_catalog_id, component_quantity)` tuples expressing anatomical
composition of grouped terms. Treat it as read-only: do not add table
constraints, do not let Hibernate manage DDL for it, and do not regenerate
catalog ids that would invalidate it.

### `bone` (`bone/Bone.java`)

`bone_id` PK; associated `individual` rows via the `bone_individual` bridge (no
direct `individual_id` column); `bone_catalog_id` FK → `bone_catalog` **ON
DELETE RESTRICT** (required); `bone_source` TEXT NOT NULL; `specimen_name`,
`repository`, `year_discovery` TEXT; `bone_processing` BOOLEAN;
`bone_quantity_source` TEXT; `bone_quantity_min` INTEGER NOT NULL CHECK `>= 1`;
`laterality` `laterality_enum`; tooth/vertebra/phalanx/rib qualifier fields and
their `_confidence` booleans; `hand_foot_bone_segment`
`hand_foot_bone_segment_enum`; `bone_details` TEXT; `updated_at` (trigger).

### `bone_individual` (bridge)

`bone_id` FK → `bone(bone_id)` ON DELETE CASCADE; `individual_id` FK →
`individual(individual_id)` ON DELETE CASCADE; PRIMARY KEY `(bone_id,
individual_id)`. `Bone` is the owning JPA side (`@ManyToMany` with
`@JoinTable(name = "bone_individual")`); `Individual.bones` is the inverse side
with no cascade, so deleting an individual removes only its bridge rows and never
the shared bone. Aggregate bone counts joining through this bridge must use
`COUNT(DISTINCT bone.bone_id)`.

### `skeleton` (`skeleton/Skeleton.java`)

`skeleton_id` PK; `individual_id` FK → `individual` ON DELETE CASCADE (required);
`skeleton_category` `skeleton_category_enum` NOT NULL; `preservation_index`
NUMERIC CHECK `0..100`; `specimen_name`, `repository`, `year_discovery` TEXT;
`bone_processing` BOOLEAN; `description` TEXT; `updated_at` (trigger).

### `dating_technique` (`dating_technique/DatingTechnique.java`)

`dating_technique_id` PK; `dating_technique_name` TEXT NOT NULL UNIQUE;
`description` TEXT; `updated_at` (trigger).

### `dated_sample` (`dated_sample/DatedSample.java`)

`dated_sample_id` PK; `sample_origin` `sample_origin_enum` NOT NULL;
`archaeological_context_id`, `bone_id`, `skeleton_id` INTEGER NULL FKs;
`material` TEXT NOT NULL; `dating_type` `dating_type_enum` NOT NULL; `updated_at`
(trigger).

Constraints (also enforced in `DatedSampleService.validate`): exactly one origin
FK non-null; the non-null FK matches `sample_origin`;
`chk_dating_type_origin_consistency` (`CONTEXT` → `INDIRECT`/`UNCERTAIN`;
`BONE`/`SKELETON` → `DIRECT`/`UNCERTAIN`).

> The `dated_sample` origin FK columns are declared without explicit `ON DELETE`
> modifiers (PostgreSQL default `NO ACTION`), so they do **not** cascade from
> their parents. `Requires code verification` if exact behaviour matters for a
> migration.

### `dating_result` (`dating_result/DatingResult.java`)

`dating_result_id` PK; `dated_sample_id` FK → `dated_sample` ON DELETE CASCADE;
`dating_technique_id` FK → `dating_technique` **ON DELETE RESTRICT**;
`dates_bp_uncal` NUMERIC; `dates_range` NUMERIC; `notes` TEXT; `updated_at`
(trigger).

### `bibliographic_reference` and bridges (`reference/Reference.java`)

`bibliographic_reference_id` PK; `authors`, `year`, `title`, `journal` NOT NULL;
`volume`, `issue`, `pages`, `doi` optional; `updated_at` (trigger).
`site_bibliographic_reference` and
`archaeological_context_bibliographic_reference` are composite-PK bridges with
both FKs ON DELETE CASCADE.

### `funerary_context` and bridge (`funerary_context/FuneraryContext.java`)

`funerary_context_id` PK; `archaeological_context_id` FK ON DELETE CASCADE
(required); `burial_context` `burial_context_enum` NOT NULL DEFAULT
`'UNCERTAIN'`; `burial_type` `burial_type_enum` (nullable, constrained against
`burial_context` by `chk_funerary_context_type`); `ochre_presence`,
`grave_goods` BOOLEAN; `notes` TEXT; `updated_at` (trigger).
`funerary_context_individual` is a composite-PK bridge, both FKs ON DELETE
CASCADE.

### `app_user` and `refresh_token`

Server-only authentication tables. Not exported by the dataset endpoint, not
referenced by the archaeological domain. See
[backend/02-security-and-auth.md](./backend/02-security-and-auth.md).

## 6. PostgreSQL enum types

| Enum type | Values |
| --- | --- |
| `culture_phase_enum` | `EARLY_UPPER_PALAEOLITHIC`, `FULL_UPPER_PALAEOLITHIC`, `FINAL_UPPER_PALAEOLITHIC`, `UNCERTAIN` |
| `sex_enum` | `MALE`, `FEMALE`, `INDETERMINATE` |
| `age_class_main_enum` | `SUBADULT`, `ADULT`, `INDETERMINATE`, `SUBADULT_ADULT` |
| `age_class_subcategory_enum` | `PERINATAL`, `FETAL`, `INFANT_I`, `INFANT_II`, `JUVENILE`, `YOUNG_ADULT`, `SENILE_ADULT`, `MATURE_ADULT` |
| `age_unit_enum` | `MONTHS`, `YEARS` |
| `individual_type_enum` | `INDIVIDUAL`, `MIXED_INDIVIDUALS`, `UNASSIGNED_REMAINS` |
| `bone_category_enum` | `TOOTH`, `VERTEBRA`, `RIB`, `PHALANX`, `OTHER` |
| `skeleton_main_enum` | `CRANIAL`, `POSTCRANIAL` |
| `skeleton_region_enum` | `NEUROCRANIUM`, `SPLANCHNOCRANIUM`, `UPPER_LIMB`, `LOWER_LIMB`, `HANDS`, `FEET`, `PELVIS`, `TORSO` |
| `bone_term_type_enum` | `ATOMIC`, `GROUPED`, `VAGUE_STRUCTURED`, `VAGUE_GENERIC` |
| `tooth_type_enum` | `DECIDUOUS`, `PERMANENT`, `SUPERNUMERARY`, `UNKNOWN` |
| `laterality_enum` | `LEFT`, `RIGHT`, `UNKNOWN` |
| `phalanx_type_enum` | `HAND`, `FOOT` |
| `vertebra_type_enum` | `CERVICAL`, `THORACIC`, `LUMBAR`, `UNKNOWN` |
| `tooth_vertical_position_enum` | `UPPER`, `LOWER`, `UNKNOWN` |
| `hand_foot_bone_segment_enum` | `PROXIMAL`, `MIDDLE`, `DISTAL`, `INTERMEDIATE`, `LATERAL`, `UNKNOWN` |
| `skeleton_category_enum` | `ALMOST_COMPLETE_SKELETON`, `COMPLETE_SKELETON`, `PARTIAL_SKELETON`, `INCOMPLETE_SKELETON`, `POSTCRANIAL_SKELETON`, `UNDEFINED_SKELETON` |
| `dating_type_enum` | `DIRECT`, `INDIRECT`, `UNCERTAIN` |
| `sample_origin_enum` | `CONTEXT`, `BONE`, `SKELETON` |
| `burial_context_enum` | `YES`, `NO`, `POSSIBLE`, `UNCERTAIN` |
| `burial_type_enum` | `PRIMARY_SINGLE_BURIAL`, `PRIMARY_DOUBLE_BURIAL`, `PRIMARY_TRIPLE_BURIAL`, `PRIMARY_MULTIPLE_BURIAL`, `SECONDARY_SINGLE_BURIAL`, `SECONDARY_DOUBLE_BURIAL`, `SECONDARY_TRIPLE_BURIAL`, `SECONDARY_MULTIPLE_BURIAL` |

JPA mapping pattern:

```java
@Enumerated(EnumType.STRING)
@JdbcTypeCode(SqlTypes.NAMED_ENUM)
@Column(name = "<column>", columnDefinition = "<enum_name>_enum")
private <JavaEnum> value;
```

## 7. Triggers and timestamps

A shared trigger function maintains `updated_at`:

```sql
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS trigger AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

Tables with this trigger: `site`, `archaeological_context`, `individual`,
`bone`, `skeleton`, `culture`, `culture_feature`, `dating_technique`,
`dated_sample`, `dating_result`, `funerary_context`, `bibliographic_reference`.
JPA maps `updated_at` as `insertable = false, updatable = false`. Bridge tables
and `bone_catalog` carry no `updated_at` and no trigger. `app_user.created_at`
and `refresh_token.created_at` use `@CreationTimestamp` / SQL `DEFAULT
CURRENT_TIMESTAMP`.

Treat `updated_at` as advisory metadata only — do not derive cache keys or
visibility rules from it.

## 8. Cascade behaviour

Most archaeological-domain FKs use `ON DELETE CASCADE`
(`site → archaeological_context → individual → skeleton`;
`archaeological_context → funerary_context`; `dated_sample → dating_result`;
both bibliography bridges; `funerary_context_individual`; and `bone_individual`
on both `bone_id` and `individual_id`).

Two FKs use `ON DELETE RESTRICT`: `bone.bone_catalog_id` and
`dating_result.dating_technique_id` — controlled vocabulary cannot be removed
while in use. `archaeological_context.culture_id` is `ON DELETE SET NULL`.

Because `bone` is no longer a child of `individual`, deleting an `individual`
removes only its `bone_individual` associations — a bone shared with other
individuals is **not** deleted, and a bone with no remaining associations is not
auto-deleted (whether the service prunes orphaned bones is application policy,
`Requires code verification`). The backoffice should display strong delete
confirmations.

## 9. Indexes

| Index | Target |
| --- | --- |
| `idx_archaeological_context_site_id` | `archaeological_context(site_id)` |
| `idx_archaeological_context_culture_id` | `archaeological_context(culture_id)` |
| `idx_individual_archaeological_context_id` | `individual(archaeological_context_id)` |
| `idx_individual_sex` | `individual(sex)` |
| `idx_individual_age_class_main` | `individual(age_class_main)` |
| `idx_bone_individual_individual_id` | `bone_individual(individual_id)` |
| `idx_bone_bone_catalog_id` | `bone(bone_catalog_id)` |
| `idx_skeleton_individual_id` | `skeleton(individual_id)` |
| `idx_dated_sample_context_id` | `dated_sample(archaeological_context_id)` |
| `idx_dated_sample_bone_id` | `dated_sample(bone_id)` |
| `idx_dated_sample_skeleton_id` | `dated_sample(skeleton_id)` |
| `idx_dating_result_sample_id` | `dating_result(dated_sample_id)` |
| `idx_dating_result_technique_id` | `dating_result(dating_technique_id)` |
| `idx_funerary_context_archaeological_context_id` | `funerary_context(archaeological_context_id)` |
| `idx_funerary_context_individual_individual_id` | `funerary_context_individual(individual_id)` |
| `idx_refresh_token_user_id` | `refresh_token(user_id)` |
| `idx_refresh_token_expires_at` | `refresh_token(expires_at)` |
| `idx_refresh_token_family_id` | `refresh_token(family_id)` |

Preserve these unless the corresponding query pattern goes away.

## 10. Modelling principles

These explain the *why* behind the schema:

1. **Source wording and normalised values are stored separately.**
   `bone_catalog.bone_catalog_name` (normalised vocabulary), `bone.bone_source`
   (observed wording), `bone.bone_quantity_source` (quantity wording), and
   `bone.bone_quantity_min` (minimum reliable count) are distinct. This is
   required for MNBR computation and to preserve source-level uncertainty.
2. **Composition is not observation.** `bone_catalog_component` is a view with
   fixed ids declaring anatomical composition. Never use it as observed data.
3. **Curatorial duplication on `bone` and `skeleton` is intentional.** A
   polymorphic `osteological_record` table was considered and rejected.
4. **Dating attaches to a sample, not directly to a thing.** `dated_sample`
   carries origin/material/type; `dating_result` carries the numbers. One sample
   can produce several results.
5. **Funerary groupings are first-class.** Member-count vs burial-type
   consistency is a service-level concern, not a DB constraint.
6. **Authentication is isolated from the research domain.**
7. **Most editable tables maintain `updated_at`** via the shared trigger.
8. **Most relations cascade; controlled vocabulary is guarded** (`ON DELETE
   RESTRICT`).

## 11. Entity model vs. API DTO model

API consumers see a narrower, flat projection of the JPA entity graph:

- Response DTOs use `common/summary/*` records (one level deep) to avoid
  recursive serialization.
- API DTOs use camelCase; columns use snake_case.
- The API resource path `/api/references` exposes the `bibliographic_reference`
  table.
- `mni_statistical` and `updated_at` are read-only in the API.

See [05-api-contract.md](./05-api-contract.md) for the full DTO map and
[backend/01-backend-architecture.md](./backend/01-backend-architecture.md) for
the mapper layer.

## 12. Raw schema script

The full PostgreSQL DDL — `CREATE DATABASE`, all tables, enums, views, triggers,
and indexes — is available as a reference copy:

**Reference schema script:** [`database/schema.sql`](./database/schema.sql)

Rules of engagement:

- **Reference material, not the editable source.** The authoritative schema is
  `backend/src/main/resources/db/database.sql`; schema evolution happens there.
- **Do not run it blindly against a live database** — it contains `CREATE
  DATABASE paleohumans` and is intended for fresh-environment bootstrapping.
- **Do not inline its contents** into Markdown documents.
- **Keep it in sync with the authoritative backend schema** (see the drift note
  in §2).

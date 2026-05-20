# 11 — Database

This document explains the **PostgreSQL database** that powers the PaleoHumans platform. It is the canonical reference for database structure, table responsibilities, enum vocabulary, constraints, and modelling principles.

> **Scope.** This document describes the database. It does **not** describe the frontend-facing entity contract (see [03-domain-model.md](./03-domain-model.md)) or the REST endpoints exposed on top of it (see [04-backend-api-contract.md](./04-backend-api-contract.md)). When the three documents disagree, the running backend response wins, then the database, then the frontend doc.

> **The raw SQL schema is not inlined here.** The authoritative DDL is in [`./database/paleohumans-schema.sql`](./database/paleohumans-schema.sql). Treat that file as reference material — do not edit it casually; the schema is owned by the backend repository and any change must originate there as a migration, not as a documentation tweak.

---

## 1. Purpose

`paleohumans` is a PostgreSQL database for storing, normalising, and serving structured information about **European Upper Palaeolithic human remains**. It is the upstream source of truth for:

- the Spring Boot REST API,
- the public Angular `web` app,
- the Angular `backoffice` admin app,
- AI coding agents working on either side.

The schema encodes more than storage. It encodes **curation decisions** about how palaeoanthropological evidence should be represented: where MNI lives, what "individual" can mean, how dating attaches to a sample, and how burials group individuals together.

---

## 2. Database and the platform

```text
PostgreSQL (paleohumans)
        │
        ▼
Spring Boot backend (separate repo)
        │     REST: http://127.0.0.1:8080/api
        ▼
Angular workspace (this repo)
   ├─ projects/web        (public site)
   └─ projects/backoffice (admin CRUD)
```

Implications for frontend agents:

- Table names, column names, enum values, and FK directions documented here are **the** vocabulary. The backend exposes camelCase variants in DTOs; the underlying truth is here.
- The frontend never speaks SQL. Anything that looks like ad-hoc SQL on the frontend is a bug.
- A field on the frontend that has no counterpart in this document is either a derived/computed value or stale — investigate before assuming it is real.

---

## 3. v2 model

The schema is on **v2**. The previous v1 entities `osteological_unit`, `specimen`, `burial_group`, and the top-level `date` table have been replaced. The active hierarchy:

```text
site
  → archaeological_context
      → individual
          → bone
          → skeleton
      → funerary_context
          ↔ individual         (via funerary_context_individual)
      ↔ bibliographic_reference (via archaeological_context_bibliographic_reference)

dated_sample (exactly one of: archaeological_context | bone | skeleton)
  → dating_result

site ↔ bibliographic_reference (via site_bibliographic_reference)
culture → culture_feature
```

Vocabulary reminders:

- a **site** is a geographic locality,
- an **archaeological_context** is a stratigraphic + cultural slice within a site (replaces most of v1 `osteological_unit`),
- an **individual** is the anthropological record for one person, mixed individuals, or unassigned remains (now owns MNI),
- **bone** and **skeleton** are separate categories of osteological record, never merged,
- **dated_sample + dating_result** replaces the old `date` table,
- **funerary_context** is a group-level burial entity (replaces v1 `burial_group`),
- **bibliographic_reference** is shared across sites and archaeological contexts.

Removed v1 entities — these **must not be recreated** in backend or frontend models:

```text
osteological_unit
specimen
date
individual_osteological_unit
osteological_unit_specimen
specimen_bone
osteological_unit_bibliographic_reference
burial_group
burial_group_osteological_unit
burial_group_individual
```

Replacement map:

```text
osteological_unit → archaeological_context + individual + funerary_context
specimen          → curatorial fields on bone/skeleton + dated_sample
old date          → dated_sample + dating_result
burial_group      → funerary_context + funerary_context_individual
```

---

## 4. Subsystems

The database is divided into six subsystems. Each subsystem is small enough to reason about in isolation; cross-subsystem joins happen through a few well-defined FKs.

### 4.1 Sites and archaeological contexts

Tables:

- `site` — geographic locality, georeferenced (`latitude`, `longitude`), unique `site_name`.
- `archaeological_context` — stratigraphic/cultural slice inside a `site`. FKs: `site_id` (required), `culture_id` (nullable). Carries `stratigraphic_context` text.

This subsystem has **no aggregate counts**. Counts of individuals, bones, skeletons, or dated samples per site are computed via joins or dedicated stats endpoints — they are not denormalised onto `site` or `archaeological_context`.

### 4.2 Individuals

Tables:

- `individual` — central anthropological record. FK: `archaeological_context_id` (required). Owns MNI, age-at-death, sex, age class, and `individual_type`.

`individual_type_enum` semantics:

- `INDIVIDUAL` — one identifiable person; requires `mni = '1'`.
- `MIXED_INDIVIDUALS` — multiple individuals that cannot be cleanly separated; requires `mni <> '1'`.
- `UNASSIGNED_REMAINS` — remains not confidently assignable to a specific individual; any valid MNI.

MNI is stored as **text** to preserve raw curation wording (`'?'`, `'2'`, etc.) and exposed as a generated integer `mni_statistical` for stats and filtering.

### 4.3 Bones and skeletons

Tables:

- `bone_catalog` — controlled anatomical vocabulary. Self-referential (`parent_bone_catalog_id`). Seed rows use **stable explicit IDs**.
- `bone_catalog_component` — a **view** (not a table) listing fixed anatomical decompositions for grouped terms such as `cranium` → `neurocranium` + `splanchnocranium`, or `coxal` → `ilium` + `ischium` + `pubis`. Anatomical composition is a fixed fact, not observational data, which is why it lives in code/view form.
- `bone` — observed per-element record. FKs: `individual_id`, `bone_catalog_id`. Stores anatomical detail (laterality, tooth/vertebra/phalanx/rib subfields), curatorial fields (`specimen_name`, `repository`, `year_discovery`, `bone_processing`), and quantity data (`bone_quantity_source` raw wording, `bone_quantity_min` reliable count).
- `skeleton` — whole-skeleton preservation assessment. FK: `individual_id`. Stores `skeleton_category`, `preservation_index` (0–100), and the same curatorial fields as `bone`.

`bone` and `skeleton` are intentionally **separate**. They share a few curatorial fields because the same individual may have bones discovered in different campaigns or stored in different repositories. The controlled duplication is preferred over a polymorphic `osteological_record` table.

### 4.4 Funerary contexts

Tables:

- `funerary_context` — burial-level grouping inside an `archaeological_context`. Carries `burial_context` (`YES`/`NO`/`POSSIBLE`/`UNCERTAIN`), `burial_type`, `ochre_presence`, `grave_goods`, `notes`.
- `funerary_context_individual` — many-to-many bridge between `funerary_context` and `individual`. Models double/triple/multiple burials by attaching multiple individuals to one funerary context.

Constraint: if `burial_context = 'NO'`, `burial_type` must be null; if `'YES'`, `burial_type` must be non-null; `POSSIBLE`/`UNCERTAIN` are flexible.

### 4.5 Dating: dated samples and dating results

Tables:

- `dating_technique` — lookup table of methods (AMS ¹⁴C, OSL, TL, AAR, etc.). Unique by name. `dating_result.dating_technique_id` is `ON DELETE RESTRICT`: techniques in use cannot be deleted.
- `dated_sample` — the **what was dated**. Carries `sample_origin` (`CONTEXT` | `BONE` | `SKELETON`), one of three nullable origin FKs (`archaeological_context_id`, `bone_id`, `skeleton_id`), `material` text, and `dating_type` (`DIRECT` | `INDIRECT` | `UNCERTAIN`).
- `dating_result` — the **numerical outcome**. Carries `dates_bp_uncal` (uncalibrated BP), `dates_range` (sigma), `notes`, and an FK to `dating_technique`.

Two constraint pairs guard the model:

1. Exactly **one** of `archaeological_context_id`, `bone_id`, `skeleton_id` must be non-null, and it must match `sample_origin`.
2. `sample_origin = 'CONTEXT'` allows only `INDIRECT` or `UNCERTAIN`; `BONE`/`SKELETON` allow only `DIRECT` or `UNCERTAIN`. Direct dating of contextual material and indirect dating of human bone are both rejected.

There is **no calibrated BP range column** anywhere in the schema. Only `dates_bp_uncal` ± `dates_range`. Any frontend feature that requires calibrated dates needs a backend schema change.

### 4.6 Cultures

Tables:

- `culture` — archaeological technocomplex/culture (Aurignacian, Gravettian, Magdalenian, …). Carries `phase` (`culture_phase_enum`), chronological bounds `start_bp ≥ end_bp`, and visualisation metadata (`color`, `color_rgb`).
- `culture_feature` — repeated textual features per culture (one-to-many).

`archaeological_context.culture_id` is nullable and `ON DELETE SET NULL`: deleting a culture does not delete its contexts.

### 4.7 Bibliography / references

Tables:

- `bibliographic_reference` — normalised bibliography entry (authors, year, title, journal, volume, issue, pages, doi).
- `site_bibliographic_reference` — M2M bridge: site ↔ reference.
- `archaeological_context_bibliographic_reference` — M2M bridge: archaeological_context ↔ reference.

A reference can be attached at both the site and context level. There is no direct M2M between `individual` and `bibliographic_reference`; cite via context.

### 4.8 Authentication / application tables

Tables:

- `app_user` — backoffice user account (`username`, `password_hash`, `role`, `enabled`, `created_at`).
- `refresh_token` — refresh-token metadata for JWT sessions (`token_hash`, `family_id`, `issued_at`, `expires_at`, `family_expires_at`, `revoked`, `revoked_at`, `replaced_by_token_id` self-FK).

These are **not part of the archaeological domain**. Frontend domain agents can ignore them. They exist so the backoffice JWT auth flow has a place to persist users and refresh tokens; see [02-architecture.md](./02-architecture.md) and [06-backoffice.md](./06-backoffice.md) for the consumption side.

---

## 5. Entity relationship overview

The cardinalities below summarise the structural shape of the schema. They are normative for backend/DTO design; the FK columns and constraints live in [`./database/paleohumans-schema.sql`](./database/paleohumans-schema.sql).

```text
site (1) ──── (N) archaeological_context
site (M) ──── (N) bibliographic_reference         [via site_bibliographic_reference]

archaeological_context (1) ──── (N) individual
archaeological_context (1) ──── (N) funerary_context
archaeological_context (1) ──── (N) dated_sample  [when sample_origin = 'CONTEXT']
archaeological_context (M) ──── (N) bibliographic_reference
                                                   [via archaeological_context_bibliographic_reference]
archaeological_context (N) ──── (1) culture       [nullable; ON DELETE SET NULL]

individual (1) ──── (N) bone
individual (1) ──── (N) skeleton
individual (M) ──── (N) funerary_context          [via funerary_context_individual]

bone     (1) ──── (N) dated_sample                [when sample_origin = 'BONE']
skeleton (1) ──── (N) dated_sample                [when sample_origin = 'SKELETON']

dated_sample (1) ──── (N) dating_result
dating_result (N) ──── (1) dating_technique       [ON DELETE RESTRICT]

bone (N) ──── (1) bone_catalog                    [ON DELETE RESTRICT]
bone_catalog (1) ──── (N) bone_catalog            [self, parent_bone_catalog_id]
bone_catalog → bone_catalog_component (view)      [grouped term → atomic components]

culture (1) ──── (N) culture_feature

app_user (1) ──── (N) refresh_token
refresh_token (N) ──── (0..1) refresh_token       [self, replaced_by_token_id]
```

Indexes follow the common query dimensions: contexts by `site_id` and `culture_id`; individuals by `archaeological_context_id`, `sex`, and `age_class_main`; bones by `individual_id` and `bone_catalog_id`; skeletons by `individual_id`; dated samples by each of the three origin FKs; dating results by `dated_sample_id` and `dating_technique_id`; funerary contexts by `archaeological_context_id`; funerary-context membership by `individual_id`; refresh tokens by `user_id`, `expires_at`, and `family_id`.

---

## 6. Modelling principles

These principles explain the *why* behind the schema. Agents should re-derive design decisions from these rather than from defaults.

### 6.1 Source wording and normalised values are stored separately

The bone subsystem keeps:

- `bone_catalog.bone_catalog_name` — normalised anatomical vocabulary,
- `bone.bone_source` — observed source wording for the bone,
- `bone.bone_quantity_source` — source wording for the quantity,
- `bone.bone_quantity_min` — minimum reliable count.

This is required for MNBR computation and to preserve source-level uncertainty. Never collapse source wording into the normalised vocabulary.

### 6.2 Composition is not observation

`bone_catalog_component` is a **view** with fixed IDs. It declares anatomical composition (e.g. `cranium = neurocranium + splanchnocranium`). It must not replace `bone.bone_quantity_min` and must not be used as observed data. Because the view uses fixed `bone_catalog_id` values, seed data must insert `bone_catalog` rows with stable explicit IDs.

### 6.3 Curatorial duplication on `bone` and `skeleton` is intentional

`specimen_name`, `repository`, `year_discovery`, and `bone_processing` are present on both `bone` and `skeleton`. A single individual may have bones discovered in different campaigns and stored in different repositories. A polymorphic `osteological_record` table was considered and rejected.

### 6.4 Dating attaches to a sample, not directly to a thing

The `dated_sample` row carries the origin (context/bone/skeleton), the material, and the dating type. The `dating_result` row carries the numbers and technique. One sample can produce several results (e.g. duplicate runs, different techniques). The schema enforces that origin + dating type combinations are coherent.

### 6.5 Funerary groupings are first-class

Double and triple burials are represented as one `funerary_context` linked to multiple individuals through `funerary_context_individual`. The schema does **not** enforce that `burial_type = PRIMARY_DOUBLE_BURIAL` implies exactly two members — that consistency must be enforced in service-level validation (backend or backoffice forms), not in the DB.

### 6.6 Authentication is isolated from the research domain

`app_user` and `refresh_token` do not connect to any archaeological table. Authentication exists to gate backoffice writes; it does not own or annotate research content.

### 6.7 Most editable tables maintain `updated_at`

A shared trigger function `set_updated_at()` runs `BEFORE INSERT OR UPDATE` on the editable domain tables (`site`, `archaeological_context`, `individual`, `bone`, `skeleton`, `culture`, `culture_feature`, `dating_technique`, `dated_sample`, `dating_result`, `funerary_context`, `bibliographic_reference`). Bridge tables do not carry `updated_at`. Treat `updated_at` as advisory metadata only — do not derive cache keys or visibility rules from it.

### 6.8 Most relations cascade; references are guarded

- Domain trees cascade: deleting a `site` removes its contexts, individuals, bones, skeletons, and funerary contexts.
- `archaeological_context.culture_id` is `ON DELETE SET NULL`.
- `bone.bone_catalog_id` and `dating_result.dating_technique_id` are `ON DELETE RESTRICT` — controlled vocabulary cannot be removed while in use.
- Bridge tables use composite primary keys; deleting either side cascades the bridge row.

---

## 7. Notes for AI agents

### 7.1 Source of truth

- For **table/column/enum names, FK directions, and constraint logic**: this document and [`./database/paleohumans-schema.sql`](./database/paleohumans-schema.sql) are authoritative.
- For **JSON DTO shapes and casing** consumed by the frontend: [04-backend-api-contract.md](./04-backend-api-contract.md) is authoritative; the live backend response is the final arbiter.
- For **the frontend's view of entities**: [03-domain-model.md](./03-domain-model.md) is authoritative.

When the docs disagree, code wins (live backend response → SQL schema → frontend models → narrative docs).

### 7.2 What frontend agents *may* infer from this document

- Field existence, nullability, and enum value sets.
- Relationship cardinalities (1-to-N, M-to-N).
- That a field is generated (`mni_statistical`) or trigger-maintained (`updated_at`).
- That the dating model has no calibrated-BP column.
- That `funerary_context_individual` is the only way an `individual` joins a burial group.
- That bone catalog IDs are stable and used by a view; reseeding requires care.

### 7.3 What frontend agents *may not* infer from this document

- The **JSON shape** of any API response (field casing, embedded summaries, pagination envelopes). Use [04-backend-api-contract.md](./04-backend-api-contract.md) and the live response.
- The set of **public vs admin** endpoints. The database does not know.
- The **filter and sort parameters** accepted by REST endpoints.
- Which entities are exposed publicly — for example, `dating_technique` is a normal table here but is only managed via the backoffice on the frontend.
- The behaviour of **PATCH** semantics: absent key vs `null` vs value. Those rules live in the backend/frontend layer (see `cleanPatch()` in the backoffice).
- SSR strategy, route maps, components, services, or styling — those belong to the frontend docs (01–10).

### 7.4 Maintenance rule

When the backend repository changes the schema:

1. The backend ships a migration.
2. The new DDL is mirrored into [`./database/paleohumans-schema.sql`](./database/paleohumans-schema.sql).
3. This document is updated where it describes the changed shape.
4. If the frontend DTO shape changes, [03-domain-model.md](./03-domain-model.md) and [04-backend-api-contract.md](./04-backend-api-contract.md) are updated.

Do not append migration logs or changelog entries here. The git history of the backend repository is the change log.

---

## 8. Raw schema script

The full PostgreSQL DDL — `CREATE DATABASE`, all tables, enums, views, triggers, and indexes — lives next to this document.

**Raw schema script:** [`./database/paleohumans-schema.sql`](./database/paleohumans-schema.sql)

Rules of engagement:

- **Reference material, not an editable artifact.** The frontend repository is not where schema evolution happens; the backend repository owns migrations. Treat this file as a verified snapshot.
- **Do not run it against a live database from this repo.** It contains `CREATE DATABASE paleohumans` and is intended for fresh-environment bootstrapping by backend infrastructure.
- **Do not inline its contents** into Markdown documents. The file is the canonical form; duplicating it invites drift.
- **Rename or replace only as a coordinated change** with the backend repository. Renaming the file requires updating the link in this document and in any agent prompt that mentions it.

If a future dump ships notes or comments that are not pure DDL, keep the `.txt` extension and document the exception here.

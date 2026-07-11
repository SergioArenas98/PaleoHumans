# 07 — Validation and Errors

This document describes how the backend rejects bad input and how it
shapes error responses.

## Two-layer validation

1. **Bean Validation** on `*CreateRequest` and `*UpdateRequest` DTOs
   runs at the controller boundary via `@Valid`. Failures produce
   HTTP `400` `validation` errors with field-level details.
2. **Service-layer validation** enforces business invariants that
   Bean Validation cannot express (cross-field rules, set membership,
   enum/string interplay, lookup-by-id). Failures throw
   `IllegalArgumentException` (mapped to HTTP `400`) or
   `NotFoundException` (HTTP `404`).

The PostgreSQL schema also enforces invariants. A DB violation at
runtime produces a `DataIntegrityViolationException` which is
mapped to HTTP `409 Conflict` with a refined message when the cause
matches a known unique constraint.

## Bean Validation patterns

DTOs are Java records with `jakarta.validation.constraints.*`
annotations. Common patterns:

- `@NotNull` on required reference ids and required enums.
- `@NotBlank` on required textual fields (where used).
- `@Min(0)` / `@Min(1)` on numeric bounds.
- `@AssertTrue` on derived record methods for cross-field checks
  (e.g. `IndividualCreateRequest.isAgeRangeValid()`).

Example: `IndividualCreateRequest`:

```java
@NotNull Integer archaeologicalContextId,
String individualName,
@NotNull IndividualType individualType,
String mni,
String ageAtDeathText,
@Min(0) Integer ageAtDeathMin,
@Min(0) Integer ageAtDeathMax,
AgeUnit ageUnit,
@NotNull AgeClassMain ageClassMain,
AgeClassSubcategory ageClassSubcategory,
@NotNull Sex sex,
@NotNull Boolean sexCertain
```

with:

```java
@AssertTrue(message = "Minimum age-at-death cannot be greater than maximum age-at-death")
public boolean isAgeRangeValid() {
    return ageAtDeathMin == null
        || ageAtDeathMax == null
        || ageAtDeathMin <= ageAtDeathMax;
}
```

## PATCH semantics (`*UpdateRequest`)

Update requests are mutable classes whose fields are
`JsonNullable<T>`:

- `JsonNullable.undefined()` — field omitted in the JSON; do not
  change.
- `JsonNullable.of(value)` — field present with a value; set.
- `JsonNullable.of(null)` — field present with explicit `null`;
  clear (only valid when the column allows null).

`common/util/MapperUtils.isDefined(JsonNullable<?>)` returns `true`
when the field was sent. Service code branches on that to decide
whether to apply the change.

`MapperUtils.requireNonBlank(value, fieldName)` rejects blank strings
where the schema requires non-null text and trims leading/trailing
whitespace.

`MapperUtils.normalize(value)` trims strings and converts blanks to
`null`.

`JsonNullableModule` is registered through `JacksonConfig` so
round-trip serialization is consistent.

## Service-level invariants by domain

These rules complement the DB CHECK constraints. The service throws
`IllegalArgumentException` (mapped to `400`) when violated.

### Individual (`IndividualService.validateIndividual`)

- `individualType`, `ageClassMain`, `sex`, `sexCertain` must be
  non-null.
- `ageAtDeathMin <= ageAtDeathMax` when both are present.
- `mni` must match `^\?$|^[1-9][0-9]*$`.
- `INDIVIDUAL` requires `mni == "1"`.
- `MIXED_INDIVIDUALS` requires `mni != "1"`.
- `UNASSIGNED_REMAINS` accepts any valid mni.

### Dated sample (`DatedSampleService.validate`)

- exactly one of `archaeologicalContextId`, `boneId`, `skeletonId`
  is non-null.
- `sampleOrigin` must match the populated reference:
  - `CONTEXT` ↔ `archaeologicalContextId`
  - `BONE` ↔ `boneId`
  - `SKELETON` ↔ `skeletonId`
- `CONTEXT` cannot have `datingType = DIRECT`.
- `BONE` and `SKELETON` cannot have `datingType = INDIRECT`.

### Funerary context (`FuneraryContextService`)

- `burialContext = NO` ⇒ `burialType` must be null.
- `burialContext = YES` ⇒ `burialType` must be non-null.
- `POSSIBLE` and `UNCERTAIN` allow either.
- The `individualIds` payload on PATCH **replaces** the full member
  set.

### Skeleton (`SkeletonService`)

- `skeletonCategory` required (enum).
- `preservationIndex ∈ [0, 100]` when present.

### Bone (`BoneService`, `BoneMapper`)

- `boneSource` required and non-blank.
- `boneQuantityMin >= 1`.
- `individualIds` must be non-empty on create (at least one associated
  individual — no orphan bones); every referenced individual and the
  `boneCatalogId` must reference existing rows. Duplicate ids are ignored.
- On PATCH, a defined `individualIds` array **replaces** the full
  association set; `null` or an empty array is rejected (a bone must keep
  at least one individual). An absent `individualIds` leaves it unchanged.

### Individual remains exclusivity (`RemainsExclusivityGuard`)

A cross-table invariant enforced on every write that attaches remains to
an individual:

- an individual may have **one or more bones** **or** **one general
  skeleton**, but **never both**;
- an individual may have **at most one skeleton**.

Enforced server-side on `POST/PATCH /api/bones` and
`POST/PATCH /api/skeletons`. The guard takes a `PESSIMISTIC_WRITE` lock on
the parent individual (`SELECT … FOR UPDATE`) **inside the write
transaction**, then checks the opposite table, so concurrent requests
cannot create a bone and a skeleton for the same individual. Violations
throw `RemainsExclusivityException` → HTTP `409` with the stable code
`INDIVIDUAL_REMAINS_EXCLUSIVITY` (see *Remains-exclusivity conflict*
below). A DB unique index + cross-table triggers act as a race-proof
backstop; their SQLSTATE `23505`/`23514` is mapped to the same body.

### Culture (`CultureService`)

- `startBp >= endBp`.

### Reference id lookups (all services)

When a request supplies a foreign-key id, the service loads the
target through the corresponding repository and throws
`NotFoundException("<Entity> not found: <id>")` if missing. This
produces HTTP `404`.

### Many-to-many replacement semantics

When a PATCH request includes a defined `*Ids` list, the
corresponding many-to-many set is **replaced** with that list. There
are no add-one / remove-one endpoints. Examples:

- `FuneraryContextUpdateRequest.individualIds` replaces members.
- `SiteUpdateRequest.referenceIds` replaces site bibliography.
- `ArchaeologicalContextUpdateRequest.referenceIds` replaces context
  bibliography.

## Error response shape

`GlobalExceptionHandler` (in `exception/`) returns RFC 7807
`ProblemDetail` bodies. The base shape is:

```json
{
  "type": "about:blank",
  "title": "<short title>",
  "status": <int>,
  "detail": "<message>"
}
```

Validation errors add a `fieldErrors` property:

```json
{
  "type": "about:blank",
  "title": "Validation error",
  "status": 400,
  "detail": null,
  "fieldErrors": {
    "ageAtDeathMin": "must be greater than or equal to 0",
    "isAgeRangeValid": "Minimum age-at-death cannot be greater than maximum age-at-death"
  }
}
```

## HTTP status map

| Exception | HTTP | Title |
| --- | --- | --- |
| `NotFoundException` | 404 | "Not found" |
| `org.springframework.web.servlet.resource.NoResourceFoundException` | 404 | "Not found" |
| `UnauthorizedException` | 401 | "Unauthorized" |
| `org.springframework.security.authentication.BadCredentialsException` | 401 | "Unauthorized" (detail: "Invalid username or password") |
| `org.springframework.security.core.AuthenticationException` | 401 | "Unauthorized" (detail: "Authentication failed") |
| `org.springframework.security.oauth2.jwt.JwtException` | 401 | "Unauthorized" (detail: "Invalid or expired token") |
| `org.springframework.security.access.AccessDeniedException` | 403 | "Forbidden" |
| `MethodArgumentNotValidException` | 400 | "Validation error" (+ `fieldErrors`) |
| `IllegalArgumentException` | 400 | "Bad request" |
| `RemainsExclusivityException` | 409 | "Conflicting remains for individual" (+ `code`, `fieldErrors.individualId`, `individualId`) |
| `IndividualContextReassignmentBlockedException` | 409 | "Move blocked by related records" (+ `code` `INDIVIDUAL_CONTEXT_REASSIGNMENT_BLOCKED`, `blockingFuneraryContexts[]`, `fieldErrors.archaeologicalContextId`, `individualId`) |
| `DatingRecordValidationException` | 422 | "Invalid dating record" (+ stable `code`, `fieldErrors`) |
| `SelfActionForbiddenException` | 409 | "Self-action not allowed" (+ stable `code`, optional `fieldErrors`) |
| `PasswordChangeException` | 422 | "Invalid password change" (+ stable `code`, `fieldErrors`) |
| `DataIntegrityViolationException` | 409 | "Conflict" |
| `TooManyRequestsException` | 429 | "Too Many Requests" |
| anything else | 500 | "Internal Server Error" (detail: "An unexpected error occurred") |

## Conflict messages

`GlobalExceptionHandler.conflict` inspects the most specific cause
and rewrites the detail for known unique constraints:

- `bone_catalog.bone_catalog_name` collision → `boneCatalogName already exists`.
- `site.site_name` (or `uq_site_name_lower`) collision → `A site with this name already exists`.
- `app_user.username` collision → `A user with this username already exists`.

The same handler also recognises the remains-exclusivity backstop: a
unique-index (`uq_skeleton_one_per_individual`) or trigger violation
(message mentioning a skeleton/bones conflict) is re-mapped to the
remains-exclusivity Problem Detail below instead of the generic message.

Unknown DB integrity violations return the generic `Data integrity
violation` message. Stack traces and SQL details are never leaked to
clients.

### Remains-exclusivity conflict

`RemainsExclusivityException` (and the DB backstop) produce a stable,
frontend-readable RFC 7807 body. The `code` is constant across the four
operations; `detail`/`fieldErrors.individualId` vary per case:

```json
{
  "type": "https://paleohumans.org/problems/individual-remains-exclusivity",
  "title": "Conflicting remains for individual",
  "status": 409,
  "detail": "Individual 42 already has a skeleton record. An individual cannot have both bone records and a skeleton record.",
  "code": "INDIVIDUAL_REMAINS_EXCLUSIVITY",
  "individualId": 42,
  "fieldErrors": {
    "individualId": "This individual already has a skeleton; bones cannot be added."
  }
}
```

Triggered by: `POST /api/bones` (or `PATCH` reassign) for an individual
with a skeleton; `POST /api/skeletons` (or `PATCH` reassign) for an
individual with bones or an existing skeleton.

### Dating-record validation

The transactional dating aggregate (`/api/admin/dating-records`,
`DatingRecordController`) enforces the dated-sample domain rules server-side
*before* anything is written, and raises `DatingRecordValidationException` →
HTTP `422` with a stable `code` and `fieldErrors`:

```json
{
  "type": "https://paleohumans.org/problems/dating-record-invalid",
  "title": "Invalid dating record",
  "status": 422,
  "detail": "sampleOrigin=CONTEXT only allows datingType INDIRECT or UNCERTAIN.",
  "code": "DATING_RECORD_INVALID_TYPE_FOR_ORIGIN",
  "fieldErrors": {
    "datingType": "sampleOrigin=CONTEXT only allows datingType INDIRECT or UNCERTAIN."
  }
}
```

| `code` | Meaning | `fieldErrors` key |
| --- | --- | --- |
| `DATING_RECORD_INVALID_ORIGIN` | Not exactly one origin id, or the origin id does not match `sampleOrigin`. | `sampleOrigin` |
| `DATING_RECORD_INVALID_TYPE_FOR_ORIGIN` | `datingType` not allowed for the origin (`CONTEXT`→`INDIRECT`/`UNCERTAIN`; `BONE`/`SKELETON`→`DIRECT`/`UNCERTAIN`). | `datingType` |
| `DATING_RECORD_REQUIRES_RESULTS` | The aggregate has no dating results (missing/empty `datingResults`). | `datingResults` |
| `DATING_RECORD_TECHNIQUE_NOT_FOUND` | A result references an unknown `datingTechniqueId`. | `datingResults[i].datingTechniqueId` |

These mirror the DB constraints (`dated_sample` XOR-origin /
`chk_dating_type_origin_consistency`), surfacing them as readable field errors
instead of raw `409` integrity violations. Plain missing-field errors (e.g. a
blank `material` or `@NotNull datingTechniqueId`) stay as the standard Bean
Validation `400`. An unknown origin id (`archaeologicalContextId`/`boneId`/
`skeletonId`) is a `404` `NotFoundException`, consistent with the other
dated-sample write paths.

### User / account self-action conflicts

`SelfActionForbiddenException` guards the backoffice account rules (see
[02-security-and-auth.md](./02-security-and-auth.md#account-management-rules-backoffice)).
It is mapped to HTTP `409` with a stable `code`:

```json
{
  "type": "https://paleohumans.org/problems/user-self-action-forbidden",
  "title": "Self-action not allowed",
  "status": 409,
  "detail": "You cannot disable your own account.",
  "code": "USER_SELF_DISABLE_FORBIDDEN",
  "fieldErrors": {
    "enabled": "You cannot disable your own account."
  }
}
```

| `code` | Triggered by | `fieldErrors` key |
| --- | --- | --- |
| `USER_SELF_DELETE_FORBIDDEN` | `DELETE /api/users/{id}` where `{id}` is the caller. | — |
| `USER_SELF_DISABLE_FORBIDDEN` | `PATCH /api/users/{id}` with `enabled=false` where `{id}` is the caller. | `enabled` |
| `USER_SELF_ROLE_CHANGE_FORBIDDEN` | `PATCH /api/users/{id}` with a `role` different from the caller's current role, where `{id}` is the caller. | `role` |

### Self password change errors

`PasswordChangeException` (raised by `POST /api/users/me/password`) is mapped to
HTTP `422` with a stable `code` and `fieldErrors`. The caller is authenticated —
the submitted value is what is wrong — so this is an unprocessable-entity error
rather than a `401`, which avoids colliding with the client's access-token-expiry
handling:

```json
{
  "type": "https://paleohumans.org/problems/user-password-change-invalid",
  "title": "Invalid password change",
  "status": 422,
  "detail": "Current password is incorrect.",
  "code": "USER_CURRENT_PASSWORD_INVALID",
  "fieldErrors": {
    "currentPassword": "Current password is incorrect."
  }
}
```

| `code` | Meaning | `fieldErrors` key |
| --- | --- | --- |
| `USER_CURRENT_PASSWORD_INVALID` | The supplied `currentPassword` does not match. | `currentPassword` |
| `USER_PASSWORD_POLICY_VIOLATION` | `newPassword` violates a policy (too short, or identical to the current one). | `newPassword` |

Plain missing/format errors (`@NotBlank currentPassword`, `@Size(min=8) newPassword`)
remain standard Bean Validation `400`s with `fieldErrors`.

## Common failure examples

### Missing required body field

```http
POST /api/individuals HTTP/1.1
Content-Type: application/json

{ "individualName": "x" }
```

→ `400` with `fieldErrors` listing the missing `@NotNull` fields
(`archaeologicalContextId`, `individualType`, `ageClassMain`, `sex`,
`sexCertain`).

### MNI/individual-type mismatch

```http
POST /api/individuals  { "individualType": "INDIVIDUAL", "mni": "2", ... }
```

→ `400 Bad request` `INDIVIDUAL requires mni='1'`.

### Wrong dated-sample origin

```http
POST /api/dated-samples
{
  "sampleOrigin": "CONTEXT",
  "boneId": 1,
  "material": "...",
  "datingType": "UNCERTAIN"
}
```

→ `400 Bad request` `sampleOrigin=CONTEXT requires archaeologicalContextId`.

### Duplicate site

```http
POST /api/admin/sites   (with admin token)
{ "siteName": "Arene Candide", ... }
```

If a site with that name exists:

→ `409 Conflict` `A site with this name already exists`.

### Unauthorized write

```http
POST /api/sites          (no JWT)
```

→ `401 Unauthorized` from the resource server / `403 Forbidden` once
authenticated as a non-admin. `GlobalExceptionHandler` maps each
appropriately.

### Rate-limit hit

→ `429 Too Many Requests` `Too many requests. Please try again later.`
(auth) or `Too many anonymous requests. Please try again later.`
(public).

## Internal logging

The catch-all `genericError` handler logs the exception with
`log.error("Unhandled exception", ex)` so that unexpected failures
appear in server logs without leaking detail to the client.

`SecurityAuditService` writes structured audit events for security-
relevant actions; see [02-security-and-auth.md](./02-security-and-auth.md).

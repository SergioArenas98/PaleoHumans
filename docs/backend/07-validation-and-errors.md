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
- `individualId` and `boneCatalogId` must reference existing rows.

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
| `DataIntegrityViolationException` | 409 | "Conflict" |
| `TooManyRequestsException` | 429 | "Too Many Requests" |
| anything else | 500 | "Internal Server Error" (detail: "An unexpected error occurred") |

## Conflict messages

`GlobalExceptionHandler.conflict` inspects the most specific cause
and rewrites the detail for known unique constraints:

- `bone_catalog.bone_catalog_name` collision → `boneCatalogName already exists`.
- `site.site_name` (or `uq_site_name_lower`) collision → `A site with this name already exists`.
- `app_user.username` collision → `A user with this username already exists`.

Unknown DB integrity violations return the generic `Data integrity
violation` message. Stack traces and SQL details are never leaked to
clients.

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
relevant actions; see [06-security-and-auth.md](./06-security-and-auth.md).

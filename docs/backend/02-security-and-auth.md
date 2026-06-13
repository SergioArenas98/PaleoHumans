# 06 — Security and Authentication

## Components

Authentication code lives in:

- `auth/` — `AuthController`, `AuthRateLimiter`, request records.
- `security/` — `SecurityConfig`, `JwtConfig`, `JwtService`,
  `AuthManagerConfig`, `PasswordConfig`, `AppUserDetailsService`,
  `PublicEndpointRateLimiter`, `SecurityAuditService`.
- `refresh_token/` — `RefreshToken`, `RefreshTokenService`,
  `RefreshTokenRepository`, `RefreshTokenCleanupJob`,
  `TokenHashService`.
- `user/` — `AppUser`, `AppUserRepository`, `UserController`,
  `UserService`, user DTOs.

Spring Security is configured stateless (no HTTP session). Access is
controlled by JWT bearer tokens at the resource-server layer plus
in-controller `@PreAuthorize` checks on `/api/users/**`.

## Public vs admin routes

### Public — anonymous, no JWT required

From `SecurityConfig.filterChain`:

- `POST /api/auth/**` (login, refresh, logout — all public)
- `GET /api/sites/**`
- `GET /api/individuals/**`
- `GET /api/bones/**`
- `GET /api/bone-catalog/**`
- `GET /api/skeletons/**`
- `GET /api/dating-techniques/**`
- `GET /api/dated-samples/**`
- `GET /api/dating-results/**`
- `GET /api/references/**`
- `GET /api/funerary-contexts/**`
- `GET /api/cultures/**`
- `GET /api/stats/**`
- `GET /api/bone-site-search/**`
- `GET /api/archaeological-contexts/**`
- `GET /api/dataset/download`

> Open question (Unverified): `SecurityConfig` still declares
> `permitAll` for `GET /api/osteological-units/**`,
> `GET /api/specimens/**`, `GET /api/dates/**`, and
> `GET /api/burial-groups/**`. No controller responds to those paths
> in v2, so the entries are dead config rather than a real surface,
> but they should be removed. Tracked in
> [07-roadmap.md](./07-roadmap.md).

### Authenticated (any role)

- `GET /api/users/me` and `POST /api/users/me/password` — requires
  authentication (the in-controller `@PreAuthorize("isAuthenticated()")`
  is defense in depth on top of the chain rule).

### Admin (`ROLE_ADMIN`)

- All other `/api/**` paths — this includes every `POST`/`PATCH`/`DELETE`
  on the domain resources, the entire `/api/admin/**` namespace, and
  user-management endpoints other than the `/me` variants.

### Default deny

Any request that does not match the rules above is denied
(`anyRequest().denyAll()`).

## JWT contract

Tokens are HS256 JWTs signed with the symmetric secret
`app.security.jwt.secret`. The issuer claim is
`app.security.jwt.issuer`.

### Access token

Issued by `JwtService.createAccessToken`.

Claims:

- `iss` — issuer
- `sub` — username
- `iat`, `exp` — `exp = iat + app.security.jwt.access-token-minutes`
  (defaults: dev `5`, production from yml `15`).
- `jti` — random UUID
- `typ` — `"access"`
- `role` — Spring authority string (e.g. `"ROLE_ADMIN"`)

Validation in `SecurityConfig.jwtAuthenticationConverter`:

```java
String role = jwt.getClaimAsString("role");
return role == null || role.isBlank()
    ? Collections.emptyList()
    : Collections.singletonList(new SimpleGrantedAuthority(role));
```

The backend reads the `role` claim directly into a single granted
authority. It does **not** use Spring's default `scope`/`scp` mapping
or `authorities` array.

### Refresh token

Issued by `JwtService.createRefreshSession` /
`rotateRefreshSession`. Claims include `typ = "refresh"` and
`familyId` (UUID).

Absolute family lifetime: `app.security.jwt.refresh-token-days` days
from issue. Default `14`.

Per-token idle lifetime: `app.security.jwt.refresh-token-idle-days`
days. Default `7`. The effective per-token `exp` is
`min(now + idle, familyExpiresAt)`.

Refresh tokens are validated through a separate decoder bean
(`refreshTokenDecoder`).

## Login flow (`POST /api/auth/login`)

1. `AuthRateLimiter.checkLogin(remoteIp, username)` enforces sliding
   windows per IP and per account.
2. `AuthenticationManager.authenticate(new UsernamePasswordAuthenticationToken(...))`
   — `AppUserDetailsService` resolves the user and BCrypt validates
   the password (`PasswordConfig`).
3. On failure, `SecurityAuditService` logs `login_failure` (reason
   `bad_credentials` or `user_disabled`) and the controller throws
   `UnauthorizedException`.
4. On success, `JwtService.createAccessToken` issues the access JWT
   and `JwtService.createRefreshSession` issues the first refresh
   token of a new family.
5. `RefreshTokenService.storeNew` persists the refresh token hash
   (via `TokenHashService`), family id, and lifetimes.
6. `SecurityAuditService` logs `login_success`.
7. Response: `{ accessToken, refreshToken }`.

## Refresh-token rotation (`POST /api/auth/refresh`)

1. `AuthRateLimiter.checkRefresh(remoteIp)` enforces a sliding
   window.
2. `jwtDecoder.decode(req.refreshToken())` verifies the signature
   and expiry; the controller asserts `typ == "refresh"`.
3. `RefreshTokenService.consumeForRotationOrThrow` atomically
   consumes the current refresh row (conditional update). If the
   update count is zero the token is treated as replay or invalid;
   the family is revoked.
4. The `familyId` claim must equal the persisted `family_id`; if it
   doesn't, the family is revoked.
5. The user must still exist and be enabled.
6. A new access token is issued.
7. A new refresh token is issued in the same family
   (`rotateRefreshSession`).
8. `RefreshTokenService.linkReplacement` writes
   `replaced_by_token_id` on the old row, linking the chain.
9. Response: `{ accessToken, refreshToken }`.

If a revoked or expired token in a family is presented, the
remaining active tokens in that family are revoked.

## Logout (`POST /api/auth/logout`)

`RefreshTokenService.revokeFamily(refreshToken, now)` marks the
family revoked. Access tokens are stateless and remain valid until
their normal expiry; the frontend is expected to discard them.

## Refresh-token cleanup

`RefreshTokenCleanupJob` is a scheduled task that deletes refresh-
token rows after their `expires_at` has passed. It does not
prematurely delete active rows.

## Production startup safety

`ProductionStartupValidator` runs only under the `production` profile
and fails fast if any of these are missing:

- `spring.datasource.url`
- `spring.datasource.username`
- `spring.datasource.password`
- `app.security.jwt.secret`

There is no committed fallback JWT secret in `application.yml`. The
`dev` profile reads the same `${JWT_SECRET}` env var; the `test`
profile uses a fixed value in `application-test.properties`.

## Bootstrap admin (opt-in)

`BootstrapAdminInitializer` (`config/`) seeds an initial admin only
if **all three** of these are set:

- `app.bootstrap-admin.enabled=true` (env `BOOTSTRAP_ADMIN_ENABLED`)
- `app.bootstrap-admin.username` (env `BOOTSTRAP_ADMIN_USERNAME`)
- `app.bootstrap-admin.password` (env `BOOTSTRAP_ADMIN_PASSWORD`)

If the flag is enabled but the credentials are missing, startup
throws `IllegalStateException`. If the username already exists,
seeding logs a warning and skips. After first use, disable the flag.

## Password hashing

BCrypt via `PasswordConfig.passwordEncoder()`.
`UserService.changePassword` and `UserService.create` route through
the encoder; raw passwords never persist.

## Account management rules (backoffice)

These rules are enforced **server-side** in `UserService` so the backend is the
final authority; the backoffice mirrors them in the UI for convenience only.

### Self-action safety

- **Self-delete is forbidden.** `DELETE /api/users/{id}` targeting the
  authenticated caller is rejected before anything is deleted →
  `SelfActionForbiddenException` → HTTP `409` with stable code
  `USER_SELF_DELETE_FORBIDDEN` and detail "You cannot delete your own account."
- **Self-disable is forbidden.** `PATCH /api/users/{id}` with `enabled=false`
  targeting the caller is rejected before any mutation →
  `SelfActionForbiddenException` → HTTP `409` with stable code
  `USER_SELF_DISABLE_FORBIDDEN` and `fieldErrors.enabled`.
- **Self-role-change is forbidden.** `PATCH /api/users/{id}` with a `role`
  different from the caller's current role is rejected before any mutation →
  `SelfActionForbiddenException` → HTTP `409` with stable code
  `USER_SELF_ROLE_CHANGE_FORBIDDEN` and `fieldErrors.role`. A self-PATCH that
  repeats the same role is a no-op and allowed; self password change (here or via
  `POST /api/users/me/password`) remains allowed.

These guards stop an admin from locking themselves — or, for the last admin,
everyone — out of the backoffice (by deletion, disabling, or accidental demotion).

### Refresh-token invalidation on credential change

`UserService` revokes refresh tokens whenever a credential or enabled-state
change should end existing sessions (`RefreshTokenService.revokeAllActiveForUser`
marks the rows `revoked=true`; a subsequent refresh fails with "Refresh token
revoked"). The access token is stateless and remains valid until its short
expiry — there is no access-token blacklist — so the frontend signs out
proactively after a self password change.

| Trigger | Effect |
| --- | --- |
| `POST /api/users/me/password` (self change) | Revoke all of the caller's refresh tokens. |
| `PATCH /api/users/{id}` with `password` (admin reset) | Revoke the **target** user's refresh tokens (also applies when an admin resets their own password via this endpoint). |
| `PATCH /api/users/{id}` with `enabled=false` (disable another user) | Revoke that user's refresh tokens (they also can no longer refresh/log in). |

### Deleting a user that owns refresh tokens

`refresh_token.user_id` references `app_user(user_id)` with no `ON DELETE`
action, so a naive delete would hit a FK violation. **Chosen behavior (the
"preferred option"): revoke/delete the user's tokens in the same transaction,
then delete the account.** `UserService.delete` calls
`RefreshTokenService.deleteAllForUser(user)` first, which clears the
self-referencing `replaced_by_token_id` links among the user's tokens
(`clearReplacementLinksByUser`) and then bulk-deletes them
(`deleteAllByUser`) before `repository.delete(user)`. Deleting a user with
existing tokens therefore returns `204` and never a raw `500`.

> History: the previous `RefreshTokenRepository.deleteByUser` derived query
> threw `ClassCastException` when invoked, so any delete of a token-owning user
> would have produced a `500`. It is replaced by the explicit `@Modifying`
> bulk queries above.

## Role model

A single role is used: `ROLE_ADMIN`. `AppUser.role` is the role name
without the `ROLE_` prefix (e.g. `"ADMIN"`). `AuthController.login`
prefixes it when building the granted authority. The JWT `role`
claim carries the full Spring authority (e.g. `"ROLE_ADMIN"`).

## CORS

Configured in `SecurityConfig.corsConfigurationSource`.

- `dev` profile: `allowedOriginPatterns = ["http://localhost:*", "http://127.0.0.1:*"]`.
- Non-dev profiles: `allowedOrigins = app.security.cors.production.allowed-origins`
  (comma-separated) and `allowedOriginPatterns = app.security.cors.production.allowed-origin-patterns`
  (comma-separated; supports `https://*.vercel.app`).
- Allowed methods: `GET`, `POST`, `PATCH`, `DELETE`, `OPTIONS`.
- Allowed headers: `Authorization`, `Content-Type`.
- `allowCredentials = true`. Wildcard origins are never combined with
  credentials.
- `maxAge = 3600` seconds.

Defaults defined in `application.yml`:

```yaml
app:
  security:
    cors:
      production:
        allowed-origins: "https://paleohumans.org,https://www.paleohumans.org,https://admin.paleohumans.com"
        allowed-origin-patterns: "https://*.vercel.app"
```

The `SecurityConfig` constructor `@Value` defaults are slightly
broader (also include `https://admin.paleohumans.org` and a small
set of explicit Vercel preview hostnames). The YAML values win at
runtime.

## Rate limiting

Two in-memory sliding-window limiters live in this repo. They are
local to the JVM and reset on restart. They are a fallback — a
multi-instance production deployment should also enforce limits at
an edge layer.

### `AuthRateLimiter` (`auth/`)

Keys:

- `login-ip:<remote-ip>` — `app.security.rate-limit.login-ip-max-requests`
- `login-account:<username>` — `app.security.rate-limit.login-account-max-requests`
- `refresh-ip:<remote-ip>` — `app.security.rate-limit.refresh-max-requests`

Window: `app.security.rate-limit.window-seconds` (default `60`).
Cleanup: `app.security.rate-limit.cleanup-interval-seconds` (default
`300`).

Defaults from `application.yml`:

```yaml
app.security.rate-limit:
  max-requests: 10                  # fallback for the three below
  login-ip-max-requests: 10
  login-account-max-requests: 5
  refresh-max-requests: 20
  window-seconds: 60
```

Triggered through `AuthController` only.

### `PublicEndpointRateLimiter` (`security/`)

Used by anonymous public endpoints:

- `bone-site-search:<remote-ip>` — applied only when the request has
  no `Authorization` header (`BoneSiteSearchController`).
- `dataset:<remote-ip>` — applied only when the request has no
  `Authorization` header (`DatasetController`).

Configured by `app.security.public-rate-limit` (default
`30 requests / 60 seconds`, cleanup every `300` seconds).

A limit hit raises `TooManyRequestsException`, which
`GlobalExceptionHandler` maps to HTTP `429`.

## Security headers

`SecurityConfig.filterChain` sets the following headers on every
response:

- `Content-Security-Policy: default-src 'self'; object-src 'none'; frame-ancestors 'none'; base-uri 'self'`
- `Referrer-Policy: no-referrer`
- `Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=(), usb=()`
- `X-Frame-Options: DENY`
- `X-Content-Type-Options: nosniff`
- `Cross-Origin-Opener-Policy: same-origin`
- `Cross-Origin-Resource-Policy: same-origin`
- HSTS on HTTPS responses: `max-age=31536000; includeSubDomains; preload`

A reverse proxy or CDN in front of the API should also apply the
same header policy to non-application responses.

## Audit logging

`SecurityAuditService` writes structured audit events (no passwords,
tokens, or `Authorization` headers are logged). Current audited
events:

- `login_success`, `login_failure`
- `password_change`
- `role_change`
- `user_create`, `user_update`, `user_delete`
- `dataset_download`

Password reset is not implemented; there is no event for it.

## CSRF and sessions

CSRF is disabled at the filter chain (`csrf.disable()`) — the API is
stateless and uses bearer tokens. Session creation policy is
`STATELESS`.

## Database isolation

`app_user` and `refresh_token` are server-only tables. They are not
exposed via the API (except through `/api/users/**`) and never
exported. PostgreSQL Row Level Security is not configured — the
threat model assumes only the backend has DB access. If direct
client DB access is ever introduced, RLS must be designed first.

## Caveats and open questions

- Stale legacy paths in `SecurityConfig` (`/api/osteological-units`,
  `/api/specimens`, `/api/dates`, `/api/burial-groups`) and in
  `CacheControlInterceptor`. See [07-roadmap.md](./07-roadmap.md).
- Rate limiters are JVM-local. For multi-instance deployments,
  enforce limits at an edge layer (Redis, gateway, or reverse
  proxy).
- The JWT secret is symmetric (HS256). Rotation is manual.
- No password reset flow.
- Only one role (`ADMIN`). Finer-grained backoffice permissions
  (vocabulary editor, user manager, etc.) are not yet modelled.
- `gitleaks` and OWASP Dependency-Check are configured but not run
  in CI; see [06-backend-workflow.md](./06-backend-workflow.md).

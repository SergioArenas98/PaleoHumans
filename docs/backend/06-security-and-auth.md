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
> [11-roadmap.md](./11-roadmap.md).

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
  `CacheControlInterceptor`. See [11-roadmap.md](./11-roadmap.md).
- Rate limiters are JVM-local. For multi-instance deployments,
  enforce limits at an edge layer (Redis, gateway, or reverse
  proxy).
- The JWT secret is symmetric (HS256). Rotation is manual.
- No password reset flow.
- Only one role (`ADMIN`). Finer-grained backoffice permissions
  (vocabulary editor, user manager, etc.) are not yet modelled.
- `gitleaks` and OWASP Dependency-Check are configured but not run
  in CI; see [10-development-workflow.md](./10-development-workflow.md).

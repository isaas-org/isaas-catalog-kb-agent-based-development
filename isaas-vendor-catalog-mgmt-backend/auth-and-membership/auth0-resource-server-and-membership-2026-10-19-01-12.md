# Auth0 resource server, vendor membership, staff review and access tokens

Repo: `isaas-org/isaas-vendor-catalog-mgmt-backend`
Branch: `feature/FD-7_Auth_resource_server`, PR #2 (merged 2026-10-31)
Design: `isaas-platform/architecture/authn-authz-design-2026-10-18-20-20.md`, `agent-auth-and-test-users-2026-10-18-20-20.md`, `auth-decisions-after-fd8-2026-10-19-00-45.md`. Schema: `isaas-database` FD-8.

## 1. What was done

Every endpoint except `/actuator/health` and `/actuator/info` now needs a bearer token. Packages under `org.isaas.vendorcatalog`:

| Package | Contents |
|---------|----------|
| `security` | `SecurityConfig`, `BearerTokenRouter`, personal access token provider and format, `Auth0Claims` |
| `identity` | `CurrentUser`, `CurrentUserService` (lookup and first-login binding), argument resolver, `GET /me` |
| `vendor` | Entities with `@JdbcTypeCode(SqlTypes.NAMED_ENUM)`, onboarding and membership endpoints |
| `staff` | Staff review endpoints |
| `accesstoken` | Create, list, revoke personal access tokens |
| `api` | `ApiException` and problem-detail error mapping |

Dependency: `spring-boot-starter-security-oauth2-resource-server` (the Boot 4 name). `postgresql` moved from runtime to compile scope to read `PSQLException.getServerErrorMessage()`.

## 2. Authentication

One `Authorization: Bearer` header carries either token kind. A value starting with `isaas_` goes to the access token check, anything else to Auth0 JWT validation (issuer `https://isaas-dev.us.auth0.com/`, audience `https://api.isaas/vendor`, from `AUTH0_ISSUER_URI` / `AUTH0_AUDIENCE` with those defaults):

```java
boolean accessToken = header != null && header.regionMatches(true, 0, BEARER, 0, BEARER.length())
		&& AccessTokens.looksLikeAccessToken(header.substring(BEARER.length()).trim());
return accessToken ? accessTokenManager : jwtManager;
```

Both end at the same `CurrentUser`. The backend needs no Auth0 client ID or secret: it only verifies signatures with the tenant's public keys. Boot resolves the issuer lazily, so the app and its tests start without reaching Auth0.

## 3. Authorization

- Membership and roles come only from `vendors.vendor_users` (and `platform.staff_users`), looked up per request. Token claims never decide company or role.
- First-login binding: when no row has the token's `sub`, the backend binds it to a row with the same email, case-insensitive, only if `https://isaas/email_verified` is true and the row is unbound:

```sql
UPDATE vendors.vendor_users
SET auth_provider_id = :authProviderId
WHERE lower(email) = lower(:email) AND auth_provider_id IS NULL
```

- The vendor a request acts on is always the caller's own: `/vendor/...` never takes an ID. Another vendor's member ID returns 404.
- `CurrentUser.requireMember()`, `requireAdmin()`, `requireLogin()` (Auth0, not a token), `requireVerifiedEmail()`, `requireStaff()`.
- Database rules are not repeated in Java: activation (`vendors_active_requires_verified`), the join request gate, unique emails, slug format. Their errors become 409 problem details with the database message and `constraint`. Java-side rules use a `code`: `NOT_A_MEMBER`, `ADMIN_REQUIRED`, `LAST_ADMIN`, `EMAIL_NOT_VERIFIED`, `LOGIN_REQUIRED`, `STAFF_REQUIRED`, `ALREADY_A_MEMBER`, `NOT_PENDING`.

## 4. Endpoints

| Endpoint | Who |
|----------|-----|
| `GET /me` | Anyone authenticated; also the agent apps' `whoami` |
| `POST /vendors` (new pending vendor, caller becomes admin), `GET /vendors?q=` (active vendors only), `POST /vendors/{id}/join-requests`, `GET /me/join-requests` | Signed in, verified email, not yet a member |
| `GET /vendor`, `/vendor/members`, `/vendor/verification-documents` | Active members |
| `POST /vendor/invitations`, `PATCH /vendor/members/{id}`, `GET /vendor/join-requests`, approve and reject, `POST /vendor/verification-documents` | Vendor admins; the last active admin can't be demoted or disabled |
| `POST/GET /me/access-tokens`, `DELETE /me/access-tokens/{id}` | Signed-in members, not tokens |
| `GET /staff/vendors`, `GET /staff/vendors/{id}`, `POST /staff/verification-documents/{id}/review`, `POST /staff/vendors/{id}/verification`, `POST /staff/vendors/{id}/status` | Active staff, signed in; verify, then activate |

## 5. Personal access tokens

`isaas_vpat_` plus 43 base62 characters (32 random bytes). Only the SHA-256 is stored; the value is returned once. A token acts as its owner with the owner's current role, stops working when revoked, expired or the owner is disabled, can't manage tokens, and never carries staff rights. `last_used_at` is written at most once a minute. The lifetime cap (1 to 90 days, default 30) is checked in the request and by the database.

## 6. Testing and CI

25 integration tests against Postgres 18 with `isaas-database` migrations applied. Logins use Spring Security's `jwt()`, so tests never call Auth0; access tokens go through the real filter chain. Each test writes uniquely named rows, so no cleanup or transactions are needed (a failed statement would otherwise abort a shared transaction). CI now checks out `isaas-org/isaas-database` `main` and runs `dbmate up` against the service container before `./mvnw verify`.

## 7. Next steps

- SSO auto-provisioning (default role `support`) once an enterprise connection exists; the access token would need the connection as a claim.
- Staff MFA before a prod tenant.
- Catalog listing endpoints (drafts, plans, publish) on top of `CurrentUser`.

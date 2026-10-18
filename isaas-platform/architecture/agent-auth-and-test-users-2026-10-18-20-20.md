# Agent app authentication and durable test users

Scope: both MCP agent apps, and how test logins are set up for a project that won't deploy for a long time
Status: decided in a Claude Code session on 2026-10-18. Companion to `authn-authz-design-2026-10-18-20-20.md`.

## 1. Agent apps until now

Both agent apps send a shared `X-API-Key` that neither backend checks. The key identifies the agent *app*, not a user. The rule that holds regardless of mechanism: a backend must never accept a `vendorId` or company ID from a request authenticated only by a shared key, or anyone holding it could act as any company. Company data is reached only through a real user identity resolved to a membership row.

Both MCP servers have no authentication of their own, so they must stay bound to `127.0.0.1` until per-user auth exists.

## 2. Options for acting as a user

| | A. Personal access tokens | B. OAuth through Auth0 (MCP authorization spec) |
|---|---|---|
| How | User creates a token in the web app; backend stores its hash; agent sends it; backend resolves it to the user row | Agent's client sends the user to Auth0; MCP server checks the user token and passes it on |
| Backend | Second way in, same current-user lookup | Nothing new |
| MFA, revocation | Built by us (expiry, revoke) | Auth0 |
| Client support | Any client that can send a header | Clients implementing MCP OAuth; needs Auth0 dynamic client registration |

**Decision:** A in this round. B is implemented the next time the agent apps get real tools, resources or context. A stays as the fallback for clients that can't do OAuth.

## 3. Personal access tokens (option A)

| Aspect | Decision |
|--------|----------|
| Format | `isaas_vpat_<43 base62 chars>` (vendors), `isaas_bpat_...` (buyers): 32 random bytes. The prefix lets the backend tell it from a JWT and makes leaks easy to spot |
| Storage | SHA-256 hash only, value shown once. A slow hash isn't needed for 32 random bytes, and a plain hash allows direct lookup |
| Permissions | Acts as its owner with the owner's current role; no extra scopes this round. Disabling the user kills the token on the next request |
| Expiry | Required, at most 90 days (enforced by the database) |
| Limits | A token can't create tokens; staff endpoints never accept tokens |
| Both sides | Buyers get tokens too, so `whoami` can be tested on both sides |

Changes per repo:

- **Database (FD-8, built):** `vendors.vendor_access_tokens`, `buyer_companies.buyer_access_tokens`. See `isaas-database/schema-and-seed-data/auth-membership-schema-2026-10-18-20-34.md`.
- **Backends (FD-7):** `Authorization: Bearer` values starting with `isaas_` are checked against the token table, anything else as an Auth0 JWT; both end at the same current-user lookup. `POST/GET /me/access-tokens`, `DELETE /me/access-tokens/{id}`, all requiring an Auth0 login. `last_used_at` updated at most once a minute. Logs record which way a request came in.
- **Frontends (FD-6):** an "Agent access tokens" settings page: create (shown once), list, revoke.
- **Agent apps (FD-11):** read `BACKEND_ACCESS_TOKEN` and send it as `Authorization: Bearer`; remove `BACKEND_API_KEY` and `X-API-Key`; run in `stdio` mode so one process serves one user; add a `whoami` tool (`tools/mcp_identity.py`) calling `GET /me`; document HTTP mode as single-user and localhost-only.

## 4. Durable test users

The risk is test logins that live only in a hand-clicked dashboard, or tests that need Auth0 reachable on every run. Testing is split into three tiers:

1. **Backend tests never call Auth0.** MockMvc with `spring-security-test`'s `jwt()` post-processor, or tokens signed by a key generated in the test. They run offline indefinitely.
2. **Test data comes from code.**
   - `isaas-auth0-config` holds the tenant config (applied with the Auth0 Deploy CLI, chosen over Terraform because it needs no state store) and a re-runnable script that creates or resets every test user with `email_verified: true` and a known password. Secrets come from environment variables or a password manager, never the repo.
   - `isaas-database` seeds the matching user rows with emails only. `auth_provider_id` is bound on first login, so seeds never hold a tenant-specific ID and survive a rebuilt tenant.
3. **Frontend tests against real Auth0 are a separate, guarded tier.** A Playwright global setup logs each user in once and saves the browser state, behind a `check-config` style guard. Brute-force and bot detection must be relaxed for test users.

A separate repo was chosen over setting things up by hand: manual setup works once, but the Action code and callback URLs then exist only in the dashboard, and a lost tenant or the prod tenant means redoing everything from memory.

## 5. Test user matrix

Seeded in `isaas-database` (`02_vendors.sql`, `05_buyer_companies.sql`, `06_test_users.sql`); each needs an Auth0 user with the same email.

| Company | State | Users |
|---------|-------|-------|
| Northstar CRM (vendor) | active | `admin@`, `sales@`, `support@northstar-crm.test` |
| Pulseboard (vendor) | active | `admin@pulseboard.test`, `dana@pulseboard.test` (sales); cross-vendor isolation |
| Cascade Labs (vendor) | pending | `admin@cascade-labs.test`; publish gate and staff review |
| Harbor Health Group (buyer) | active | `admin@`, `purchaser@`, `approver@`, `viewer@harbor-health.test` |
| Brightline Retail Co. (buyer) | active | `admin@brightline-retail.test`, `dana@pulseboard.test` (viewer); cross-company isolation |
| Fernway Logistics (buyer) | pending | `admin@fernway-logistics.test`; buyer gates and staff review |
| None | unaffiliated | One Auth0-only user for the create-company and join flows |
| Staff | admin | The repo owner, seeded by `08_staff.sql` |

Dana is one login with two memberships, one per side.

## 6. Next steps

- Create `isaas-auth0-config` and assign it an FD number.
- Implement option B when the agent apps next get tools; then decide whether A is kept as a fallback.
- Record every agent action as agent-originated once an audit table exists.

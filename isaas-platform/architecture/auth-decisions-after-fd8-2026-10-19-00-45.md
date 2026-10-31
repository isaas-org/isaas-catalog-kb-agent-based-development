# Auth decisions made after FD-8

Scope: Auth0 tenant setup, staff MFA, SSO defaults, service secrets
Status: decided in a Claude Code session on 2026-10-18/19 while building FD-7. Updates `authn-authz-design-2026-10-18-20-20.md`, which still says otherwise on the points below.

## 1. What changed from the FD-8 design

| Topic | FD-8 design said | Decided now | Why |
|-------|------------------|-------------|-----|
| Staff MFA | Staff endpoints require MFA (`amr` contains `mfa`) | No MFA in the dev tenant. Staff endpoints check only an active `platform.staff_users` row | Auth0's Free plan has no MFA ("Pro MFA" starts at Essentials) |
| Vendor SSO role | Lowest role, open question (vendor roles have no `viewer`) | Vendor SSO members default to `support`; buyer SSO members to `viewer` | Owner's choice. Not enforced by the database |
| API identifiers | `https://api.isaas/buyer` and `/vendor` as examples | Kept. The owner does not own `isaas.com`; the prod tenant gets real-domain identifiers | Identifiers can't be changed later; the dev tenant is throwaway |
| Claim namespace | `https://isaas/` | Kept, same reasoning | |
| Agent auth | Personal access tokens now, MCP OAuth later | Unchanged | |

## 2. Staff MFA options for the prod tenant

Must be settled before a prod tenant exists:

| Option | Cost | Protection |
|--------|------|------------|
| A. No MFA (dev tenant only) | Free | Password plus the staff table check |
| B. Staff sign in only with Google | Free | Google 2-step verification; can't be checked from our side |
| C. Auth0 Essentials | Paid (third-party listings say from about $35/month B2C) | Real Auth0 MFA, as first designed |

A was chosen for the dev tenant. The post-login Action does not call `api.multifactor.enable`.

## 3. How the apps and APIs know each other

Nothing in the Auth0 dashboard links a web app to an API by default. Three things do:

1. The frontend asks for a token with `audience` set to its API's identifier (nextjs-auth0 `authorizationParameters`).
2. Each API's user access policy is "Allow via client-grant" (`require_client_grant`), and only the matching web app has a user grant. The vendor web app can't get a buyer API token.
3. Each backend validates `iss` (the tenant) and `aud` (its own identifier).

## 4. Secrets and where they live

| Secret | Where | Level |
|--------|-------|-------|
| `isaas-auth0-config` M2M client ID and secret, `TEST_USER_PASSWORD` | `.env` on the owner's machine | Local only. If a deploy workflow is ever added: repo-level secret in a reviewed environment on `isaas-auth0-config`, never org-level, because it can change the whole tenant |
| Frontend client secrets, `AUTH0_SECRET` | Each frontend's `.env.local` (FD-6) | Local only |
| Backend issuer and audience | `application.yaml` defaults, overridable by env | Not secret: issuer and JWKS are public |
| CI database credentials | Existing org secrets `CI_DATABASE_*` | Org-level, unchanged |

The backends need no client ID or secret: they only verify tokens with the tenant's public keys.

## 5. Next steps

- Choose B or C for staff MFA before the prod tenant.
- Enforce the SSO default roles when SSO auto-provisioning is built, and decide whether the database should check them.
- FD-6: frontends pass `audience` and keep their client secrets server-side.

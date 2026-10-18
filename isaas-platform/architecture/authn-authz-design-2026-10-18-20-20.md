# Authentication and authorization design

Scope: buyer users, vendor users and marketplace staff across all repos
Status: decided in a Claude Code session on 2026-10-18. Schema built in FD-8 (`isaas-database`); backends (FD-7), frontends (FD-6) and agent apps (FD-11) to follow.

This doc settles the open questions in `auth-provider-2026-09-27-20-49.md` and `auth-and-membership-2026-09-27-20-55.md`, and records where the design changed. Agent auth and test users are in `agent-auth-and-test-users-2026-10-18-20-20.md`.

## 1. Provider and tenant setup

Auth0 stays, for identity only. Spring Boot validates a token against an issuer and an audience, so switching providers later is mostly configuration.

| Item | Decision |
|------|----------|
| Tenants | One dev tenant now; a separate prod tenant at deployment. Test users never go in prod. |
| Applications | Two Regular Web Applications: buyer frontend (localhost:3000) and vendor frontend (localhost:3001) |
| APIs | Two, one per backend, with separate audiences, so a token issued for one backend is rejected by the other |
| Connections | Username-Password-Authentication now; SSO enterprise connections per company later |
| Post-login Action | Adds namespaced `email` and `email_verified` claims to the access token, which the backend needs to bind a login to an invited row (section 3) |
| Tenant config | Kept as code in a new repo, `isaas-auth0-config` (Auth0 Deploy CLI), not hand-clicked in the dashboard |

## 2. Membership comes only from our database

The token carries who the person is (`sub`, verified email), never a company or role. On every request the backend looks up the user row by `sub` and takes the company and role from it. A user can't claim another company by editing a token, cookie or request body, and a removal or role change applies on the next request.

A membership row is created only by someone else vouching:

| Path | Who vouches |
|------|-------------|
| Bootstrap (first person) | Staff, through company verification (section 4) |
| Admin invite | An admin of the company; the invitee proves ownership of the email by logging in with it verified |
| Join request (peer approval) | An admin of the company |
| SSO auto-provisioning | The company's own IdP; the role is the lowest one |

**Binding on first login.** `auth_provider_id` is nullable and unique. An invited or seeded row has only an email. On first login, the backend fills in `sub` on the row with the same email, only if the token says the email is verified. The same mechanism serves invites, seeded test users and a rebuilt Auth0 tenant. Signing up with an email does not grant membership unless a row already exists for it.

## 3. Verification proves the company and the person

The first admin is the weak point: anyone could create "Acme Corp". So verification checks both that the company exists and that this person may act for it:

| Check | Evidence |
|-------|----------|
| The company exists | Registration document, tax ID |
| The person may act for it | Named as officer or owner in the registry, or a signed authorisation letter |
| The person is who they say | Photo ID matching the Auth0 account |

`artifact_type` gained `proof_of_authority` and `identity_document` on both sides. For this round, documents are checked outside the system and only the outcome is recorded; `file_url` points at wherever the document lives. File storage, or an identity-verification provider, comes later.

## 4. Buyer verification becomes a hard gate (Option A)

The original design made buyer verification optional and triggered. Three options were compared:

| Option | Gate | Verdict |
|--------|------|---------|
| A. Verify before activation | Company starts `pending`; nothing commercial until verified | **Chosen** by the repo owner |
| B. Verify at first commitment | Paid subscriptions and payment methods only | Recommended by Claude, not chosen |
| C. Verify only for credit terms | Purchase orders and ACH only | Rejected: too little protection |

Buyer companies now mirror vendors: `pending`, `active` only while `verified`. What a company can do in each state applies to both sides:

| Action | Pending | Active |
|--------|---------|--------|
| Browse, watchlist, edit profile, upload verification documents | yes | yes |
| Admin invites members | yes | yes |
| Listed in company search; receives join requests; SSO provisioning | no | yes |
| Buyer: trials, paid subscriptions, payment methods, reviews | no | yes |
| Vendor: draft listings | yes | yes |
| Vendor: publish | no | yes |

Invites are allowed while pending because the person holding the verification documents is often not the one who created the company. Join requests are not, so an impostor's unverified "Acme" can't collect real Acme employees.

## 5. Staff

Staff review verification and recover companies with no usable admin. Decisions:

- New schema `platform`, table `platform.staff_users` (roles admin/reviewer). It fills the dangling `reviewed_by_staff_id` from `v1.0.0`.
- The first staff admin is the repo owner, added by `scripts/bootstrap_staff.sh` with the email passed in, not committed. Signing up on Auth0 never makes anyone staff.
- Staff endpoints require MFA (`amr` contains `mfa`).
- Each backend gets `/staff/...` endpoints for its own side. No staff UI this round; staff use an API client. A staff console, if needed, gets its own repo so staff pages never sit in a customer app.

## 6. Where each part is built

| Repo | FD | Branch |
|------|----|--------|
| `isaas-database` | FD-8 | `feature/FD-8_Auth_membership_schema` |
| Both backends | FD-7 | `feature/FD-7_Auth_resource_server` |
| Both frontends | FD-6 | `feature/FD-6_Auth0_login` |
| Both agent apps | FD-11 | `feature/FD-11_Agent_access_tokens` |
| `isaas-auth0-config` (new) | not assigned | not assigned |

Build order follows dependencies, not FD numbers: database, Auth0 config, backends, then agent apps and frontends.

Backend plan (FD-7): OAuth2 resource server; a request-scoped current user resolved as in section 2; company ID always from the current user, never from the request; 401 for no or bad token, 403 for wrong role, 403 `NOT_A_MEMBER` so the frontend can route to onboarding; endpoints `GET /me`, company creation, join requests, members, staff verification, access tokens. Public catalog GETs on the buyer backend stay anonymous.

Frontend plan (FD-6): `@auth0/nextjs-auth0` v4 wired through `proxy.ts` (Next.js 16), tokens obtained server-side and sent as Bearer from `src/lib/backend.ts`, per-user data never inside `"use cache"`, onboarding and member admin pages, access-token settings page.

## 7. Next steps

- Create the `isaas-auth0-config` repo and give it an FD number.
- Decide the lowest role for vendor SSO members: vendor roles are admin/sales/support, with no `viewer`.
- Decide on file storage for verification documents before real users submit IDs.
- Revisit the one-email-one-company rule (`lower(email)` is unique per users table) if consultants buying for several companies become a use case.

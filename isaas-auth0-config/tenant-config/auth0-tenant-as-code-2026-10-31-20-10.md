# Auth0 dev tenant as code, and test users

Repo: `isaas-org/isaas-auth0-config` (new repo, created by the owner)
Branch: `feature/FD-7_Auth0_tenant_config`, PR #1 (merged 2026-10-31)
Design: `isaas-platform/architecture/agent-auth-and-test-users-2026-10-18-20-20.md`, `auth-decisions-after-fd8-2026-10-19-00-45.md`

## 1. What was done

The dev tenant `isaas-dev.us.auth0.com` was first set up by hand in the dashboard, then described in `tenant.yaml` and applied with the Auth0 Deploy CLI (`auth0-deploy-cli` 9.0.0). A script creates the test users.

```
tenant.yaml                       APIs, apps, grants, database connection, Action, trigger
actions/add-isaas-claims/code.js  post-login Action
config/dev.json                   tenant domain, managed resource types, no deletes
scripts/sync-test-users.js        creates or resets test-users.json through the Management API
test/*.test.js                    offline tests (node --test)
```

## 2. What the tenant contains

| Resource | Contents |
|----------|----------|
| APIs | `iSaaS Buyer API` `https://api.isaas/buyer`, `iSaaS Vendor API` `https://api.isaas/vendor`, RS256 |
| Applications | `iSaaS Buyer Web` (localhost:3000), `iSaaS Vendor Web` (localhost:3001), Regular Web, callback `/auth/callback` (nextjs-auth0 v4) |
| Client grants | One user grant per web app, to its own side's API only |
| Database connection | `Username-Password-Authentication`, enabled for both web apps |
| Action | `Add iSaaS claims`, post-login (v3, node22) |

Each API only lets client-granted apps get user tokens, and denies machine-to-machine access:

```yaml
subject_type_authorization:
  user:
    policy: require_client_grant
  client:
    policy: deny_all
```

The Action adds what the backends need to bind a login to an invited row; access tokens for a custom API only accept namespaced custom claims:

```js
api.accessToken.setCustomClaim(NAMESPACE + 'email', event.user.email);
api.accessToken.setCustomClaim(NAMESPACE + 'email_verified', event.user.email_verified === true);
```

## 3. Decisions

| Decision | Picked | Rejected and why |
|----------|--------|------------------|
| Tool | Deploy CLI | Terraform: needs a state store, too much for a solo owner. Hand-clicking only: the Action and callbacks would live only in the dashboard |
| Starting point | Hand-written YAML matching the dashboard setup | Exporting the tenant: there is no dashboard export; a CLI export includes unmanaged defaults and can include secrets |
| Scope | `AUTH0_INCLUDED_ONLY` = resourceServers, clients, clientGrants, databases, actions, triggers; `AUTH0_ALLOW_DELETE: false` | Managing attack protection and MFA here: plan-dependent, kept in the dashboard |
| Database `options` | Left out, only `enabled_clients` managed | The Management API replaces the whole options object on update and would reset the dashboard's password policy |
| Values | Written directly in `tenant.yaml` | Keyword replacement (`##BUYER_API_AUDIENCE##` from `config/dev.json`) was used first; the owner asked in review to write the known values in. A prod tenant will need its own values |
| Test passwords | One shared `TEST_USER_PASSWORD` from the environment | One random password per user: harder to test by hand |

## 4. Test users

`npm run test-users` creates or resets the 13 users in `test-users.json`: verified email (`verify_email: false`, so no mail is sent to `.test` addresses), the shared password, and brute-force blocks cleared. They match the rows seeded by `isaas-database` (`02`, `05`, `06`), plus `newcomer@isaas.test`, who has no row, for the create-company and join flows. A social login with the same email is a different Auth0 user and is left alone.

## 5. Testing

`npm test` runs 12 offline tests: the Action's claims, the user script against a fake Management API, and checks on `tenant.yaml` (each app granted only its own API, references resolve, callbacks match `/auth/callback`, no keyword markers left, no secrets in config). CI runs it on every PR. `tenant.yaml` was also validated against the Deploy CLI's own JSON schemas. The import itself was not run in the session: it needs the M2M credentials.

## 6. Next steps

- The owner creates the `isaas-auth0-config` M2M app (Management API: read/create/update for resource servers, clients, client grants, connections, actions, triggers, users), fills `.env`, and runs `npm run deploy:dry-run`, `npm run deploy`, `npm run test-users`.
- A prod tenant: its own values, and the staff MFA decision first.

# Auth0 resource server, buyer company membership, staff review and access tokens

Repo: `isaas-org/isaas-buyer-catalog-browsing-backend`
Branch: `feature/FD-7_Auth_resource_server`, PR #2 (merged 2026-10-31)
Same design as `isaas-vendor-catalog-mgmt-backend/auth-and-membership/auth0-resource-server-and-membership-2026-10-19-01-12.md`; this doc covers what differs.

## 1. What was done

The buyer backend got the same structure as the vendor backend, derived from it with renames and then adjusted to the `buyer_companies` schema: packages `security`, `identity`, `company`, `staff`, `accesstoken`, `api` under `org.isaas.buyercatalog`.

## 2. Differences from the vendor backend

| Topic | Buyer backend |
|-------|---------------|
| Audience | `https://api.isaas/buyer` |
| Access token prefix | `isaas_bpat_` |
| Routes | `/companies` (create, search, join requests), `/company/...` (caller's own company), `/staff/companies/...` |
| Company fields | `name`, `websiteUrl`; no slug, no support email |
| New company | `pending` and `unverified` (vendors start `pending`/`pending`) |
| Roles | `admin`, `purchaser`, `approver`, `viewer`; SSO default will be `viewer` |
| Review queue | The first verification document moves an `unverified` company to `pending` |
| Gates | Only an active company can subscribe or add payment methods (database triggers from FD-8). No subscription or payment endpoints exist yet |
| Public catalog | Meant to stay anonymous; opened up when catalog endpoints are added |

The review-queue rule lives in the entity:

```java
public void markSubmittedForReview() {
	if (verificationStatus == VerificationStatus.unverified) {
		verificationStatus = VerificationStatus.pending;
	}
}
```

## 3. Review notes

The owner asked whether the backend should require the Auth0 client ID and secret. It doesn't: it only verifies tokens with the issuer's public keys and the audience. The client secret belongs to the buyer frontend (FD-6). Issuer and audience are public, so the dev-tenant defaults in `application.yaml` are safe to commit.

## 4. Testing

24 integration tests against Postgres 18 with `isaas-database` migrations, `jwt()` logins, no Auth0 calls. CI applies the migrations before `./mvnw verify`, as in the vendor backend.

## 5. Next steps

- Catalog browse, search and item detail on `items.published_catalog`, anonymous.
- Watchlist, subscriptions and payment methods on top of `CurrentUser`; gate errors surface as `409 RULE_VIOLATION`.
- SSO auto-provisioning (default `viewer`) and staff MFA, as on the vendor side.

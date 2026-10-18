# Spring Boot boilerplate and CI

Repo: `isaas-org/isaas-vendor-catalog-mgmt-backend`
Branch: `feature/FD-5_Initial_boilerplate`
PR: https://github.com/isaas-org/isaas-vendor-catalog-mgmt-backend/pull/1 (merged 2026-10-18 02:10 UTC)

## 1. What was done

The vendor catalog management API was scaffolded the same way as the buyer backend: Spring Initializr, Java 21, Spring Boot 4.1.1, Maven wrapper, and the Web MVC, Data JPA, Validation, Actuator and PostgreSQL driver dependencies. The package is `org.isaas.vendorcatalog` and the port is 8082. It doesn't serve endpoints beyond Actuator yet.

Configuration, CI and the decisions behind them are identical to the buyer backend. See `isaas-buyer-catalog-browsing-backend/boilerplate-and-ci/spring-boot-boilerplate-2026-10-18-02-11.md`. This doc covers what this PR's review changed and what differs for the vendor side.

## 2. Review: CI credentials moved to secrets

The review asked whether the CI database username and password could be stored as GitHub secrets. Claude recommended against it: the database is a throwaway container, a missing secret would break CI, and fork PRs don't get secrets. The owner still preferred secrets, named `CI_DATABASE_*`. Both backends were changed, with a guard job against the failure mode raised:

```yaml
check-config:
  steps:
    - name: Check CI database secrets
      run: |
        missing=""
        [ -n "$CI_DATABASE_URL" ] || missing="$missing CI_DATABASE_URL"
        ...
        case "$CI_DATABASE_URL" in
          */isaas_test|*/isaas_test\?*) ;;
          *) echo "::error::CI_DATABASE_URL must point at database isaas_test, ..."; exit 1 ;;
        esac
build:
  needs: check-config
```

The owner added the secrets at organization level. Claude can't read or set secrets from its environment, so the change was held locally until the secrets existed, then pushed. CI passed on both backends.

## 3. Review: local configuration made explicit

The README Configuration section now opens by saying the defaults are the local development configuration for `isaas-database`'s docker Postgres (`postgres`/`postgres` on 55432), not production settings. The column is labelled "Local default". The same wording was applied to the buyer backend.

## 4. Vendor-specific rules in AGENTS.md

- Publishing is gated in the database: CHECK `vendors_active_requires_verified` and trigger `items.enforce_publish_gate`. Surface those errors to the caller instead of duplicating the checks in Java.
- The other rules match the buyer backend: no migrations here, real schema names, explicit enum mapping, and business logic only in this service.

## 5. Decisions

| Decision | Chosen | Why |
|----------|--------|-----|
| CI credentials | Organization secrets `CI_DATABASE_URL/USERNAME/PASSWORD` | Owner's preference. Defined once for both backends |
| Missing-secret behavior | Separate `check-config` job that fails with a clear message | Otherwise the service container fails before any step runs, with no useful error |
| Where the publish gate lives | Database only | One enforcement point; the API translates the database error |

## 6. Next steps

- First vendor endpoints: create and update a draft listing, and manage pricing plans, features, media and taxonomy links. Then publish, relying on the database gate.
- Map the publish-gate trigger error and the CHECK violation to clear API errors, such as 409 with a reason.
- Spring Security with Auth0, scoped to the vendor's own items through `vendor_users`.

# Spring Boot boilerplate and CI

Repo: `isaas-org/isaas-buyer-catalog-browsing-backend`
Branch: `feature/FD-5_Initial_boilerplate`
PR: https://github.com/isaas-org/isaas-buyer-catalog-browsing-backend/pull/1 (merged 2026-10-18 02:11 UTC)

## 1. What was done

The buyer catalog API was scaffolded with Spring Initializr: Java 21, Spring Boot 4.1.1, Maven wrapper, and the Web MVC, Data JPA, Validation, Actuator and PostgreSQL driver dependencies. It doesn't serve catalog endpoints yet; it starts, connects to Postgres and reports health.

```
src/main/java/org/isaas/buyercatalog/BuyerCatalogBrowsingBackendApplication.java
src/main/resources/application.yaml
src/test/java/org/isaas/buyercatalog/BuyerCatalogBrowsingBackendApplicationTests.java
.github/workflows/test.yml
AGENTS.md, CLAUDE.md, README.md
```

Verified before the PR: `./mvnw verify` against Postgres 18, and `/actuator/health` returned `UP` with `liveness` and `readiness` groups. The buyer frontend and the buyer agent app were run end to end against it, and both reported `UP`.

## 2. Configuration

```yaml
spring:
  datasource:
    url: ${DATABASE_URL:jdbc:postgresql://localhost:55432/isaas_dev}
    username: ${DATABASE_USERNAME:postgres}
    password: ${DATABASE_PASSWORD:postgres}
  jpa:
    open-in-view: false
    hibernate:
      # The schema is owned and migrated by isaas-database. Never let Hibernate change it.
      ddl-auto: none
server:
  port: ${PORT:8081}
management:
  endpoints:
    web:
      exposure:
        include: health,info
```

- The defaults are the **local development configuration**. They point at the docker Postgres from `isaas-database`, seeded with `db/seeds`. In review the README was changed to say this explicitly, and that deployed environments must set all three `DATABASE_*` variables from their own secrets.
- For Neon, `DATABASE_URL` is in JDBC form with the pooled host: `jdbc:postgresql://<host>-pooler.<region>.aws.neon.tech/<db>?sslmode=require`. That isn't the `postgres://` form the `isaas-database` workflows use.
- `open-in-view: false` keeps database sessions from leaking into view rendering.
- Only `health` and `info` are exposed over HTTP.

## 3. Decisions

| Decision | Chosen | Why |
|----------|--------|-----|
| Schema management | `ddl-auto: none`, no migrations in this repo | `isaas-database` owns the schema and releases it with versioned tags |
| Security | Not added yet | With Spring Security on the classpath, every endpoint is locked. Auth0 integration waits for the membership schema migration |
| Port | 8081 | The vendor backend uses 8082, so both run side by side |
| Context-load test | Kept, against a real Postgres | It proves the datasource config works. An empty database is enough |

## 4. CI

`.github/workflows/test.yml` has two jobs:

1. `check-config` fails fast with a clear message if `CI_DATABASE_URL`, `CI_DATABASE_USERNAME` or `CI_DATABASE_PASSWORD` is missing, or if the URL doesn't point at `isaas_test`.
2. `build` runs `./mvnw -B verify` against a `postgres:18` service container whose credentials come from the same secrets.

The secrets are organization secrets on `isaas-org`. Why they're secrets, and the fork-PR trade-off, are in `isaas-platform/language-and-framework/app-repos-versions-and-ci-2026-10-17-22-17.md`.

## 5. Rules recorded in AGENTS.md

- Real schema names are `items`, `vendors` and `buyer_companies` (not `buyers`).
- Map Postgres native ENUMs explicitly, for example `@JdbcTypeCode(SqlTypes.NAMED_ENUM)`.
- Catalog listing reads `items.published_catalog`; don't re-implement the published/active filter in Java.
- Business logic stays here; the frontend and agent app call this API.

## 6. Next steps

- First catalog endpoints: browse, search (full-text over `search_vector`, trigram on `name`), filter by taxonomy, and item detail with pricing plans, features and media.
- The API contract (OpenAPI) that the buyer frontend and buyer agent app will both use.
- Spring Security as an OAuth2 resource server for Auth0 JWTs, plus the API-key path for the agent app.

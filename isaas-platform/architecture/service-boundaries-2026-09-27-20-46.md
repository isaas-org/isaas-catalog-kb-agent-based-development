# Service boundaries: frontends, backends, agent services

Scope: how the iSaaS applications talk to each other
Status: decided. The repos exist; none are built yet.

## 1. Shape

```
  buyer browser            vendor browser            external agent / orchestrator
       |                        |                        |                |
  Next.js (buyer)         Next.js (vendor)         Python MCP (buyer)   Python MCP (vendor)
   [Vercel]                [Vercel]                     |                |
       |  user JWT              |  user JWT             |  service key   |  service key
       v                        v                        v                v
  Spring Boot (buyer catalog)    Spring Boot (vendor catalog)
       \                            /
        +------ Neon PostgreSQL ---+
         (vendors, items, buyer_companies schemas)
```

The buyer side and the vendor side each have their own frontend, backend and agent application, in separate repos. They share one database (see `database-structure-2026-09-27-20-10.md`).

## 2. Responsibilities

| Component | Owns | Must not |
|-----------|------|----------|
| Next.js frontend | Rendering (SSR/ISR for public catalog pages), the Auth0 login flow, calling the backend | Hold business logic in API routes or Server Actions, or talk to Postgres |
| Spring Boot backend | Business rules, authorization, all database access, REST (and gRPC if needed) | Trust a client for anything the database or backend can check itself |
| Python MCP service | Agent-facing tools for its side (buyer browsing, vendor management) | Connect to Postgres directly |
| Postgres | Integrity rules (CHECKs, triggers, composite FKs) and the catalog view | Be reached by anything other than the backends |

## 3. Decisions behind the boundaries

- **One source of truth for business logic (Spring Boot).** If Next.js or Python also interpreted the schema, the same rules would have to be kept in sync in several places.
- **Python is a separate process.** Agent workloads behave differently from CRUD endpoints, and Python has the better agent and MCP ecosystem.
- **The MCP server is standalone.** External agents call it directly, so it is an independent client of the backend, not an internal helper of Spring Boot.
- **Buyer and vendor are split into separate repos.** This follows the user's repo plan, so each side can be built, deployed and reasoned about on its own.

## 4. Auth between components

| Caller -> callee | Mechanism |
|------------------|-----------|
| Browser -> Next.js | Auth0 session |
| Next.js -> Spring Boot | Auth0 access token (JWT), validated by Spring Security as a resource server |
| Python MCP -> Spring Boot | Shared API key or secret for now; Auth0 M2M later if worth it |
| Spring Boot -> Postgres | Neon pooled connection string |

## 5. Data flow for the first feature (buyer catalog)

1. A buyer opens a catalog page in the buyer Next.js app (server-rendered).
2. Next.js calls the buyer Spring Boot backend.
3. The backend queries `items.published_catalog` for browse and search, plus `items.pricing_plans`, `features`, `item_media` and the taxonomy tables for the item detail page.
4. The buyer MCP service exposes the same browse and search through tools, calling the same backend endpoints.

## 6. Open questions

- Hosting for the Spring Boot and Python services.
- Whether the two backends share code, such as a library of entity mappings for the common schema.
- API contract format (OpenAPI first, or generated from code).
- Whether public catalog browsing needs any authentication at all, or only the actions that follow it (watchlist, reviews, purchase).

## 7. Next steps

- Boilerplate in each new repo following these boundaries.
- Design the buyer catalog API (browse, search, filter, item detail) as the first contract.

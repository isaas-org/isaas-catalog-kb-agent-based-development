# Application stack: Spring Boot, Next.js, Python MCP

Scope: the buyer and vendor backends, frontends and agent applications
Status: decided, not yet implemented. Repos have been created; boilerplate is the next step.

## 1. What was decided

| Layer | Language / framework | Hosting |
|-------|----------------------|---------|
| Backend (buyer catalog, vendor catalog) | Java, Spring Boot | not decided |
| Frontend (buyer catalog, vendor catalog) | TypeScript, Next.js | Vercel |
| Agent applications (buyer, vendor) | Python, standalone MCP server | not decided |
| Database | PostgreSQL on Neon | Neon |

## 2. Backend: Java Spring Boot

- Chosen for type robustness and industry-standard maturity. That matters on a platform with a compliance and financial surface (KYC/KYB verification, commission rates, payouts).
- Serves REST (Spring Web), and gRPC through `grpc-spring-boot-starter` if needed.
- Spring Boot is the single source of truth for business logic and the only service that connects to Postgres.
- **Known gotcha:** the schema uses native Postgres ENUMs for nearly every status and role field. Map them with `hibernate-types` or explicit `@Convert` converters, rather than discovering the gap mid-build.
- **Trade-off accepted:** more upfront setup than FastAPI or NestJS, which is worth it for a long-running project.

## 3. Frontend: Next.js on Vercel

- SSR and ISR are a real win, not just a convenience: the catalog is public and should be indexed by search engines.
- Vercel is the lowest-friction host for Next.js.
- **Rule:** Server Components and Server Actions call the Spring Boot API. Business logic must not drift into Next.js API routes, or the platform ends up with two backends.

## 4. Agentic capabilities: separate Python service

- Kept out of the Spring Boot process. Agent workloads have a different latency and resource profile than CRUD endpoints, and Python has the deeper ecosystem (Anthropic SDK, MCP SDK).
- Runs as a **standalone MCP server** that an external agent or orchestrator calls directly. It is not called by Spring Boot.
- Reads marketplace data **only through the Spring Boot REST API**, never from Postgres directly. That keeps one interpretation of the schema and its rules.

| Question | Answer chosen |
|----------|---------------|
| Python data access | Through the Spring Boot API |
| Who calls the Python service | External agents, as a standalone MCP server |

## 5. Consequence: a second auth path

Spring Boot serves two kinds of clients:

- **Next.js on behalf of a signed-in user:** validates the Auth0 user JWT (see `auth-provider-2026-09-27-20-49.md`).
- **The Python MCP service as its own client:** for now a shared API key or secret between the two services. This can later move to an Auth0 Machine-to-Machine (client-credentials) application.

## 6. Repositories for the stack

The user has created these repos. The names are recorded once confirmed in the KB.

- Frontend for buyers browsing the catalog
- Frontend for vendors managing the catalog
- Backend for vendors managing the catalog
- Backend for buyers browsing the catalog
- Agent application for vendors managing the catalog
- Agent application for buyers browsing the catalog
- This knowledge base (`isaas-org/isaas-catalog-kb-agent-based-development`)

`isaas-buyer-frontend` existed before this split and is still empty.

## 7. Next steps

- Boilerplate for each repo: Next.js (both frontends), Spring Boot (both backends), Python MCP (both agent apps).
- Pick versions (Java LTS, Spring Boot, Next.js, Python) at boilerplate time and record them in a new doc here.
- Decide backend hosting, and whether the buyer and vendor backends share a library for the common schema mapping.
- Design the catalog API contract (browse, search, filter, item detail).

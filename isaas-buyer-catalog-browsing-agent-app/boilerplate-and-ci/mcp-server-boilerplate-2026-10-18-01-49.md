# MCP server boilerplate

Repo: `isaas-org/isaas-buyer-catalog-browsing-agent-app`
Branch: `feature/FD-5_Initial_boilerplate`
PR: https://github.com/isaas-org/isaas-buyer-catalog-browsing-agent-app/pull/1 (merged 2026-10-18 01:49 UTC)

## 1. What was done

A Python 3.13 package, managed with uv, that runs a standalone MCP server for agents browsing the catalog on behalf of a buyer company. It reaches data only through `isaas-buyer-catalog-browsing-backend`. The structure is the same as the vendor agent app, including the review outcome: tools grouped in classes under `tools/`, and no nested functions. That outcome was applied here at the owner's request. See `isaas-vendor-catalog-mgmt-agent-app/boilerplate-and-ci/mcp-server-boilerplate-2026-10-18-01-48.md` for the reasoning and code.

| Setting | Value |
|---------|-------|
| Package | `isaas_buyer_catalog_agent` |
| Server name | `isaas-buyer-catalog` |
| Default transport | streamable HTTP on `127.0.0.1:8091`, path `/mcp` |
| Backend | `BACKEND_API_URL`, default `http://localhost:8081` |

## 2. End-to-end check

Before the PR, the full path was run locally: an MCP client (`mcp.client.Client`) over streamable HTTP, to this server, to the buyer Spring Boot backend, to Postgres 18.

```python
async with Client("http://127.0.0.1:8091/mcp") as client:
    tools = await client.list_tools()           # ['backend_health']
    result = await client.call_tool("backend_health", {})
    result.structured_content                   # {'status': 'UP'}
```

## 3. Tools

| Module | Tool | What it does |
|--------|------|--------------|
| `tools/mcp_health_check.py` | `backend_health` | Reports whether the buyer backend is reachable (`UP`, or `UNREACHABLE` with a reason) |

## 4. Next steps

- Catalog tools, as one module once the buyer backend exposes them: search, browse by category, filter, item detail, compare. These are read-only, so they're the safest first agent tools.
- Watchlist and review tools later, once buyer auth exists and the agent can act as a specific buyer user.

# MCP server boilerplate and tool module structure

Repo: `isaas-org/isaas-vendor-catalog-mgmt-agent-app`
Branch: `feature/FD-5_Initial_boilerplate`
PR: https://github.com/isaas-org/isaas-vendor-catalog-mgmt-agent-app/pull/1 (merged 2026-10-18 01:48 UTC)

## 1. What was done

A Python 3.13 package, managed with uv, that runs a standalone MCP server for agents working on behalf of a vendor. It reaches data only through `isaas-vendor-catalog-mgmt-backend`. There is one tool so far, `backend_health`. The review set the structure that every later tool will follow.

```
src/isaas_vendor_catalog_agent/
  config.py                    settings from environment variables
  backend.py                   BackendClient (httpx), the only way to reach data
  server.py                    builds MCPServer, wires tool classes, runs a transport
  tools/mcp_health_check.py    HealthCheckTools
tests/
  conftest.py                  anyio backend fixture
  test_backend.py              BackendClient tests
  tools/test_mcp_health_check.py
```

## 2. MCP SDK 2.x

The SDK is pinned to `mcp>=2.3,<3`. In 2.x, `FastMCP` was renamed to `MCPServer` (`mcp.server.mcpserver`), and importing the old module raises an error that points to the migration guide. The API was read from the installed package before writing code:

```python
server = MCPServer(name="isaas-vendor-catalog", instructions="...")
server.add_tool(fn)                          # or the @server.tool() decorator
server.run("streamable-http", host=..., port=...)   # or server.run("stdio")
```

The default transport is streamable HTTP on `127.0.0.1:8092`, path `/mcp`. `MCP_TRANSPORT=stdio` is available for agent clients that launch servers as subprocesses.

## 3. Review: tools grouped in modules, no nested functions

The first version defined the tool inside `build_server`, and the tests used nested mock handlers. The review asked for related tools in one module, and no nested functions. Tools are now classes with their dependencies injected:

```python
# tools/mcp_health_check.py
class HealthCheckTools:
    def __init__(self, backend: BackendClient) -> None:
        self._backend = backend

    def register(self, server: MCPServer) -> None:
        server.add_tool(self.backend_health)

    async def backend_health(self) -> dict[str, str]:
        """Report whether the iSaaS vendor catalog backend is reachable."""
        try:
            body = await self._backend.health()
        except httpx.HTTPError as err:
            return {"status": "UNREACHABLE", "detail": str(err)}
        return {"status": str(body.get("status", "UNKNOWN"))}
```

```python
# server.py
HealthCheckTools(backend).register(server)
```

Registering a bound method leaves `self` out of the tool's input schema. A test checks that `backend_health` is listed with no properties.

## 4. Testing approach

- Tools are tested through the real server with `server.call_tool(...)`, against an `httpx.MockTransport` backend. That way the test covers registration, schema and output conversion, not just the method.
- Mock backends are module-level functions (`healthy_backend`, `refusing_backend`, `echo_api_key`). The API-key test reads the header back from the response body, so it needs no closure.
- Backend failures come back as data (`UNREACHABLE` plus a reason), not as exceptions, so the agent can act on them.

## 5. Decisions

| Decision | Chosen | Why |
|----------|--------|-----|
| Data access | Backend API only, via `BackendClient` | One interpretation of the schema and its rules |
| Service auth | `X-API-Key` header when `BACKEND_API_KEY` is set | The shared-key plan from Part 2; the backend doesn't enforce it yet |
| Tool structure | One class per tool group under `tools/`, with `register()` | Owner's review; keeps `server.py` as wiring only |
| Nested functions | Not allowed anywhere | Owner's review; recorded in `AGENTS.md` |
| Lint and format | ruff (`E, F, I, B, UP`), line length 100 | Fast, and one tool for both jobs |

CI runs `uv sync --locked`, `ruff check`, `ruff format --check` and `pytest` on PRs and on pushes to `feature/**`.

## 6. Next steps

- Listing management tools, as their own modules once the vendor backend has the endpoints: items, pricing plans, features, media.
- Enforce the API key in the vendor backend, then require `BACKEND_API_KEY` here.
- Decide how an agent acts as a specific vendor user (identity passed through to the backend), before any write tools.

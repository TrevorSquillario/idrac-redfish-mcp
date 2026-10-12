# AGENTS.md — idrac-redfish-mcp

Guidance for AI agents and developers working on this project. The primary goal is a
**test-driven development (TDD) workflow** for new features, with a strong preference for
**integration tests against a live Redfish endpoint**.

---

## 1. Project Overview

`idrac-redfish-mcp` is a FastMCP server that exposes Dell iDRAC management capabilities as
MCP tools over Redfish.

- **Scope**: Not feature parity with every Redfish endpoint. Narrowed to *useful tools for
  troubleshooting and eventually managing servers*.
- **Constraint**: Outputs must be specially crafted to **not exceed context windows** and
  prevent context bloat. Keep returned payloads concise.
- **Stack**: Python, [FastMCP](https://github.com/jlowin/fastmcp), `httpx.AsyncClient`,
  Redfish REST API over HTTPS.
- **Entry point**: `src/fastmcp_server.py` (runs over `stdio` by default, or `http` on port
  `8080` via `IDRAC_MCP_TRANSPORT_MODE`).

### Key environment variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `IDRAC_USERNAME` | — (required) | iDRAC username for basic auth. Server fails fast if unset. |
| `IDRAC_PASSWORD` | — (required) | iDRAC password for basic auth. Server fails fast if unset. |
| `IDRAC_SSL_VERIFY` | `False` | Whether to verify SSL certs (`1/true/yes/on` → True). |
| `IDRAC_SESSION_TIMEOUT` | `1800` | Token/session lifetime in seconds before re-auth. |
| `IDRAC_RETRY_ATTEMPTS` | `3` | Retry count for transient (network/5xx) failures. |
| `IDRAC_RETRY_BACKOFF` | `0.5` | Base backoff seconds (exponential: `0.5 * 2^(n-1)`). |
| `IDRAC_MCP_TRANSPORT_MODE` | `stdio` | `stdio` or `http`. |
| `LOG_LEVEL` | `INFO` | Logging level. |

---

## 2. Test-Driven Development Workflow (for new features)

**Prefer integration tests over unit tests.** Integration tests run against a real iDRAC and
validate the actual Redfish return shapes, which is where this project's value and risk live.
Reserve unit tests for pure logic (parsing, filtering, formatting) that needs no network.

### Step 1 — Query the live Redfish endpoint to learn the real return shape

Before writing any test or code, inspect the actual response from the reference host
`https://192.168.0.200` so your assertions match reality. Use curl directly (Redfish is a REST API):

```bash
curl -sk -u "root:calvin" \
  https://192.168.0.200/redfish/v1/Systems/System.Embedded.1
```

> **Rule**: Base every test assertion on a value you actually observed from the live endpoint. 
> Do not invent field names or shapes.

### Step 2 — Write the integration test FIRST (red)

Add a test in `tests/integration/`:

- **Client-level** (test the service/client directly) → add to `tests/integration/test_client.py`.
  Uses the `mock_idrac_client` fixture (a real `iDRACAsyncRedfishClient` pointed at the host).
- **Tool-level** (test the MCP tool end-to-end) → add to `tests/integration/test_fastmcp_server.py`.
  Uses the `main_mcp_client` fixture (in-process FastMCP `Client`) and calls
  `main_mcp_client.call_tool(name=..., arguments={...})`, then asserts on `json.loads(result.data)`.

Follow the existing skip pattern so tests don't hard-fail when credentials are absent:

```python
@pytest.mark.asyncio
async def test_my_new_feature(mock_idrac_client, host, username, password):
    if not username or not password:
        pytest.skip("USERNAME and PASSWORD environment variables required for integration tests")
    await mock_idrac_client.login(username, password)
    result = await mock_idrac_client.<service>.<method>(...)
    assert isinstance(result, ...)
    # assert on fields observed in Step 1
```

Run it and confirm it **fails** (red) because the feature doesn't exist yet.

### Step 3 — Implement (green)

1. Add the service method in the appropriate service under
   `src/idrac_async_redfish_client/services/` (see §3 for separation rules).
2. Add a thin `@mcp.tool` wrapper in `src/fastmcp_server.py` following the tool contract (§4).
3. Run the test and confirm it **passes** (green).

### Step 4 — Verify against the live endpoint with `fastmcp`

After implementation, verify the tool works against the live iDRAC:

```bash
# List the tool's parameters
fastmcp list src/fastmcp_server.py get_lc_logs host=192.168.0.200

# Call the tool against the live host
fastmcp call src/fastmcp_server.py get_lc_logs host=192.168.0.200
```

Confirm the output is well-formed JSON, concise, and matches the shape you asserted in the test.

---

## 3. Architecture & Layering

### 3.1 Client isolation — `src/idrac_async_redfish_client/client.py`

`iDRACAsyncRedfishClient` is the **only** class that talks HTTP. It owns:

- An `httpx.AsyncClient` (`self.http`) with `verify`, `timeout`, and JSON headers.
- Session state: `_session_uri`, `_session_id`, `_token`, `_token_created_at`, stored creds.
- A `_reauth_lock` to serialize concurrent token refreshes.

Key behaviors:
- `login(username, password)` — POSTs to `/redfish/v1/SessionService/Sessions`, extracts the
  token from `X-Auth-Token` (case-insensitive) or the body, and the session id from `Location`.
- `logout()` — DELETEs the session, clears token/headers.
- `token_valid()` — enforces `IDRAC_SESSION_TIMEOUT`; clears expired state.
- `ensure_session()` — reuses a valid token or logs in.
- `request(method, path, **kwargs)` — the central request path:
  - **401 handling**: re-authenticates under `_reauth_lock` and retries once.
  - **Retry/backoff**: retries transient failures (`httpx.RequestError`, 5xx) up to
    `IDRAC_RETRY_ATTEMPTS` with exponential backoff.
  - **Status → domain error mapping**: 404 → `RedfishNotFound`; 401/403 →
    `RedfishUnauthorized`; other 4xx → `RedfishClientError`; 5xx → `RedfishServerError`;
    timeouts → `RedfishTimeout`; connection errors → `RedfishConnectionError`.
- `get_redfish_uri(path, select=None)` — generic fetch returning parsed JSON (or text fallback);
  supports OData `$select`.

**Isolation rule**: Services and tools must **never** construct their own HTTP calls or parse
raw `httpx.Response` status codes directly. All HTTP goes through `client.request()` /
`client.get()` / `client.post()` / etc., which already maps errors to domain exceptions.

### 3.2 Services separation — `src/idrac_async_redfish_client/services/`

Each service is a thin domain layer that holds a reference to the client (`self.client`) and
encapsulates one Redfish domain. Attached to the client in `__init__`:

| Service | File | Domain |
| --- | --- | --- |
| `config` | `services/config.py` | Message registry (EEMI), BIOS registry/attributes, iDRAC attributes |
| `health` | `services/health.py` | Dell rollup health, storage health |
| `inventory` | `services/inventory.py` | Server slots, PCIe devices, firmware inventory |
| `logs` | `services/logs.py` | Lifecycle (LC) logs with filtering/pagination |
| `media` | `services/media.py` | Server screenshot export |
| `support` | `services/support.py` | SupportAssist collection |
| `telemetry` | `services/telemetry.py` | Metric report definitions/readings |

Service responsibilities:
- Build the Redfish URI and query params.
- Call `self.client.get(...)` / `self.client.post(...)`.
- Parse JSON, validate expected keys (e.g. `Members`), paginate via `@odata.nextLink` or `$skip`.
- Apply client-side filtering (e.g. severity, `ConnectorLayout`, `SubSystem`).
- Log at `INFO`/`DEBUG`/`WARNING` with the URI and params.
- Raise domain errors (`RedfishError` subclasses) or `ValueError`/`RuntimeError` for
  malformed/unexpected responses.

**Separation rule**: Services contain the Redfish domain logic. Tools in `fastmcp_server.py`
contain only the MCP plumbing (validation, session acquisition, error → JSON mapping).

### 3.3 Error handling & logging

- **Domain errors** live in `src/idrac_async_redfish_client/errors.py`, all subclassing
  `RedfishError`: `RedfishNotFound`, `RedfishUnauthorized`, `RedfishClientError`,
  `RedfishServerError`, `RedfishTimeout`, `RedfishConnectionError`, `RedfishJSONError`,
  `RedfishTransientError`.
- **Tools** catch `RedfishNotFound` (return empty/`{}`), `RedfishError` (return
  `{"error": str(e)}`), and a generic `Exception` (log with `logger.exception` and re-raise).
- **Logging** is configured in `src/utils/logging_config.py` and **MUST go to `stderr`** —
  stdout is reserved for the MCP protocol. Never `print()` to stdout. Use the module logger
  (`logger = logging.getLogger("idrac_redfish_mcp")`).

### 3.4 Authentication & session management — `src/manager.py`

`PerHostSessionManager` caches long-lived clients per host:

- `get_client(host, creds)` is an async context manager that:
  - Returns a cached client immediately if its token is valid.
  - Otherwise serializes creation/login per-host using a per-host `asyncio.Lock`.
  - **Does not close** the client on exit (clients are reused).
- `close_all()` logs out/closes all cached clients (registered via `atexit`).
- A background session logger periodically logs session counts and expires stale sessions.

**Rule**: Tools acquire clients exclusively via `session_mgr.get_client(host, creds)` — never
construct `iDRACAsyncRedfishClient` directly in a tool.

---

## 4. `fastmcp_server.py` Tool Requirements

Every new feature must be exposed as an `@mcp.tool` following this exact contract:

1. **Signature**: `host` is required; `port: int = 443`, `verify: bool = DEFAULT_IDRAC_SSL_VERIFY`
   are standard optional params. Add feature-specific params as needed.
2. **Validate `host`**: if falsy, `logger.error(...)` and `raise ValueError("missing 'host' in params")`.
3. **Build creds**: `creds = {"username": IDRAC_USERNAME, "password": IDRAC_PASSWORD, "verify": verify}`.
4. **Acquire client**: `async with session_mgr.get_client(host, creds) as client:`.
5. **Delegate**: call the appropriate service method (e.g. `client.logs.get_lifecycle_logs(...)`).
6. **Error mapping** (in order):
   - `except RedfishNotFound:` → log `logger.info(...)`, return `json.dumps([], default=str)` (or `{}`).
   - `except RedfishError as e:` → `logger.exception(...)`, return `json.dumps({"error": str(e)}, default=str)`.
   - `except Exception:` → `logger.exception(...)`, re-raise.
7. **Return**: `json.dumps(result, default=str)` — always a JSON string.
8. **Docstring**: describe the tool, its params, and the Redfish URI(s) it calls.

The generic `get_redfish_uri` tool is the escape hatch for ad-hoc endpoint access and is also
the primary tool for Step 1 of the TDD workflow.

---

## 5. Testing Conventions

### Fixtures (`tests/integration/conftest.py`)

- `username` / `password` — session-scoped, from `IDRAC_USERNAME` / `IDRAC_PASSWORD` env vars.
- `host` — parametrized from `--hosts` (or `TEST_HOSTS` env), default
  `192.168.0.200,192.168.0.201,192.168.0.202,192.168.0.203`.
- `mock_idrac_client` — a real `iDRACAsyncRedfishClient(host=host, verify=False)`; logs out after.
- `main_mcp_client` — in-process FastMCP `Client(transport=mcp)` for tool-level tests.
- `clear_session_manager_between_tests` — autouse; closes cached sessions after each test.

### Running tests

```bash
# Unit tests (no network)
pytest tests/unit

# Integration tests against the live reference host
pytest tests/integration/test_client.py --hosts 192.168.0.200

# Tool-level integration tests
pytest tests/integration/test_fastmcp_server.py --hosts 192.168.0.200

# Show live logs while running a single test
pytest tests/integration/test_client.py::test_get_metric_report_definitions \
  -s -q --hosts 192.168.0.200 -o log_cli=true -o log_cli_level=INFO
```

> **Note**: `pyproject.toml` sets `asyncio_mode = "auto"`, so async tests don't need explicit
> `@pytest.mark.asyncio` in most cases, but the existing tests include it for clarity.

---

## 6. Verification Checklist (before finishing a feature)

- [ ] Queried `https://192.168.0.200` and based assertions on observed return shapes.
- [ ] Wrote an integration test first (client-level and/or tool-level) and saw it fail (red).
- [ ] Implemented the service method in the correct service under `services/`.
- [ ] Added the `@mcp.tool` wrapper following the §4 contract.
- [ ] Test passes (green).
- [ ] Verified against the live endpoint:
  ```bash
  fastmcp list src/fastmcp_server.py <tool_name> host=192.168.0.200
  fastmcp call src/fastmcp_server.py <tool_name> host=192.168.0.200
  ```
- [ ] Output is concise JSON (context-window conscious) and matches the asserted shape.

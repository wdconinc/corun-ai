# corun-mcp-server — Implementation Plan

## Overview

`corun-mcp-server` is a standalone Python MCP (Model Context Protocol) server that gives external LLM clients — Claude Desktop, Cursor, VS Code Copilot Chat, etc. — read and write access to a corun-ai instance.
It lives in its own repository (e.g. `eic/corun-mcp-server`) and follows the same infrastructure patterns as [`eic/uproot-mcp-server`](https://github.com/eic/uproot-mcp-server), including packaging, testing, Docker, and the `AGENTS.md` convention.

---

## Goals

1. Allow an external LLM to **browse** documentation sections, prompts, and AI-generated pages hosted by corun-ai.
2. Allow an external LLM to **submit a new prompt** and **trigger a generation job** against a chosen JobDefinition.
3. Allow an external LLM to **poll job status** and **retrieve the generated page** once the job completes.
4. Support **token-based authentication** to corun-ai (Django session cookies are not suitable for machine clients).
5. Match the quality bar of `eic/uproot-mcp-server`: typed Python, NumPy-style docstrings, pytest suite, Docker image, CI via GitHub Actions, `AGENTS.md`.

---

## Prerequisite: Add a Token API to corun-ai

The MCP server is a **separate repository** and communicates with corun-ai exclusively over HTTP.
The existing corun-ai views are Django session-auth only; a small REST extension is needed first.

### 1.1 Django REST token authentication

Add Django REST Framework (DRF) + `rest_framework.authtoken` (or a lightweight alternative) to `requirements/base.txt`.
Create an `api/` URL namespace in `corun_project/urls.py`.

### 1.2 Token management for service accounts

- A Django management command `create_api_token <username>` to mint a bearer token.
- Tokens are stored in the standard DRF `Token` table (or a custom `APIToken` model if more metadata — e.g. expiry, scope labels — is desired).
- The corun-ai README and deploy docs are updated with instructions for creating tokens for MCP clients.

### 1.3 Read-only API endpoints

Implement the following DRF `APIView` / `@api_view` endpoints (all return JSON, all accept `Authorization: Token <token>`):

| Method | URL | Description |
|--------|-----|-------------|
| `GET` | `/api/v1/sections/` | List active Sections (id, name, title, description) |
| `GET` | `/api/v1/sections/<name>/` | Section detail + list of current Prompts |
| `GET` | `/api/v1/prompts/<group_id>/` | Prompt detail (content, status, version) |
| `GET` | `/api/v1/pages/<group_id>/` | Page detail (rendered markdown content, metadata) |
| `GET` | `/api/v1/jobs/<job_id>/` | Job status and result_page_group_id |
| `GET` | `/api/v1/definitions/` | List active JobDefinitions (id, name, description) |

### 1.4 Write endpoints

| Method | URL | Description |
|--------|-----|-------------|
| `POST` | `/api/v1/prompts/` | Create a new prompt (body: `section`, `content`, `definition_id`) |
| `POST` | `/api/v1/jobs/` | Submit a generation job (body: `prompt_group_id`, `definition_id`) |
| `POST` | `/api/v1/jobs/<job_id>/abort/` | Cancel a running job |

Authentication: write endpoints require a token with write scope (or simply any valid token — policy decision for the corun-ai maintainers).

---

## Repository Structure: `eic/corun-mcp-server`

```
corun-mcp-server/
├── AGENTS.md                  # AI agent instructions (mirrors uproot-mcp-server style)
├── README.md
├── SECURITY.md
├── Dockerfile
├── docker-compose.yml
├── pyproject.toml
├── .github/
│   └── workflows/
│       ├── ci.yml             # lint + tests on push/PR
│       └── docker.yml         # build + push Docker image on tag
├── src/
│   └── corun_mcp_server/
│       ├── __init__.py
│       ├── client.py          # Thin async HTTP client wrapping the corun-ai REST API
│       ├── resources.py       # Read-only data retrieval logic (sections, prompts, pages)
│       ├── jobs.py            # Job submission and polling helpers
│       └── server.py          # FastMCP server — tool definitions and entry point
└── tests/
    ├── __init__.py
    ├── conftest.py            # pytest fixtures, mock HTTP responses (respx or responses)
    ├── test_client.py         # Unit tests for client.py (mocked HTTP)
    ├── test_resources.py      # Unit tests for resources.py
    ├── test_jobs.py           # Unit tests for job lifecycle helpers
    └── test_server.py         # Integration-style tests for MCP tool wrappers
```

---

## Package Setup (`pyproject.toml`)

- **Build backend:** `hatchling` (same as uproot-mcp-server).
- **Python requirement:** ≥ 3.10.
- **Runtime dependencies:**
  - `mcp>=1.0.0` (FastMCP)
  - `httpx>=0.27` (async HTTP; preferred over `requests` for async-native MCP handlers)
  - `python-dotenv>=1.0` (optional, for local `.env` files during development)
- **Dev/test extras:** `pytest>=7.4`, `pytest-asyncio>=0.21`, `pytest-cov>=4.1`, `respx>=0.21` (httpx mock).
- **Entry point:** `corun-mcp-server = "corun_mcp_server.server:main"`.

---

## Architecture

### Layer Separation (mirrors uproot-mcp-server)

```
server.py       ← MCP tool definitions (@mcp.tool), thin wrappers, error catching
    │
resources.py    ← Pure read logic — list/get sections, prompts, pages
jobs.py         ← Job submission + async poll helpers
    │
client.py       ← Async HTTP client (httpx), auth header injection, base URL
```

`resources.py` and `jobs.py` have **no MCP imports** — they are independently testable by calling functions directly.
`server.py` catches all exceptions and returns `{"error": str(exc)}` dicts rather than raising.

### Configuration

All configuration is read at startup via environment variables (no config file required):

| Variable | Description |
|----------|-------------|
| `CORUN_BASE_URL` | Base URL of the corun-ai instance, e.g. `https://epic-devcloud.org/doc` |
| `CORUN_API_TOKEN` | Bearer token for authentication |
| `CORUN_POLL_INTERVAL` | Seconds between job status polls when using `wait_for_job` (default: `5`) |
| `CORUN_POLL_TIMEOUT` | Maximum seconds to wait for a job to complete (default: `3600`) |

`client.py` reads these at import time and raises a clear `EnvironmentError` if required variables are missing.

---

## MCP Tools

### Read Tools

#### `list_sections`
Returns all active documentation sections.
```
Input:  (none)
Output: list of {id, name, title, description}
```

#### `get_section`
Returns a section and its current prompts.
```
Input:  section_name: str
Output: {id, name, title, description, prompts: [{group_id, content, status, version}]}
```

#### `get_prompt`
Returns a specific prompt and any published pages derived from it.
```
Input:  group_id: str
Output: {group_id, version, content, status, section, pages: [{group_id, status, created_at}]}
```

#### `get_page`
Returns the full generated content of a page.
```
Input:  group_id: str
Output: {group_id, content, content_rendered, status, created_at, data: {generation_model, ...}}
```

#### `list_job_definitions`
Lists available JobDefinitions that can be used when submitting jobs.
```
Input:  (none)
Output: list of {id, name, description, status}
```

#### `get_job_status`
Returns the current status and result of a job.
```
Input:  job_id: str
Output: {id, status, definition, created_at, modified_at, result_page_group_id?, error?}
```

### Write Tools

#### `submit_prompt`
Creates a new prompt in a section (does not trigger generation).
```
Input:  section_name: str, content: str, definition_id: str | None
Output: {prompt_group_id, version, status}
```

#### `generate_from_prompt`
Triggers a generation job from an existing prompt.
```
Input:  prompt_group_id: str, definition_id: str | None
Output: {job_id, status}
```

#### `submit_and_generate`
Convenience wrapper: creates a prompt and immediately triggers generation.
```
Input:  section_name: str, content: str, definition_id: str | None
Output: {prompt_group_id, job_id, status}
```

#### `wait_for_job`
Polls the job until it reaches a terminal state and returns the result page.
This is a **long-running tool** — it loops with `asyncio.sleep(CORUN_POLL_INTERVAL)` and raises `TimeoutError` if `CORUN_POLL_TIMEOUT` is exceeded.
```
Input:  job_id: str, timeout_s: int | None
Output: {job_id, status, result_page_group_id?, content?}
```

#### `abort_job`
Cancels a running job.
```
Input:  job_id: str
Output: {ok: bool, job_id, status}
```

---

## Testing Strategy

### Unit tests (no network)

All tests run without a live corun-ai instance by mocking HTTP calls with `respx` (an httpx-compatible mock library).

- `test_client.py` — verifies auth header injection, base URL construction, error propagation (4xx, 5xx, network error).
- `test_resources.py` — verifies `list_sections`, `get_section`, `get_prompt`, `get_page` against mock responses matching the API contract.
- `test_jobs.py` — verifies `submit_prompt`, `generate_from_prompt`, `get_job_status`, `wait_for_job` (with simulated state transitions: queued → running → completed), and `abort_job`.
- `test_server.py` — verifies each `@mcp.tool()` wrapper: correct delegation to lower layers, `{"error": ...}` return on exception, JSON serialisability of all outputs.

### Integration tests (optional, skipped by default)

Gated on `CORUN_BASE_URL` and `CORUN_API_TOKEN` being set.
Run against a staging or production instance; marked with `@pytest.mark.integration`.

### Coverage

Target ≥ 90 % line coverage on `client.py`, `resources.py`, `jobs.py`.
`server.py` is tested via `test_server.py` but the thin wrappers naturally have lower independent coverage.

---

## CI/CD

### GitHub Actions: `ci.yml`

Triggers on push and pull_request to `main`.

1. `pip install -e ".[dev]"` in a matrix of Python 3.10 / 3.11 / 3.12.
2. `ruff check src/ tests/` — linting.
3. `pytest tests/ --ignore=tests/integration -v --cov=corun_mcp_server --cov-report=xml`.
4. Upload coverage to Codecov.

### GitHub Actions: `docker.yml`

Triggers on version tags (`v*`).

1. Build Docker image.
2. Push to `ghcr.io/eic/corun-mcp-server:<tag>` and `:latest`.

---

## Docker

### `Dockerfile`

```
FROM python:3.12-slim
WORKDIR /app
COPY pyproject.toml ./
COPY src/ ./src/
RUN pip install --no-cache-dir .
ENV CORUN_BASE_URL=""
ENV CORUN_API_TOKEN=""
ENTRYPOINT ["corun-mcp-server"]
```

### `docker-compose.yml`

For running the server alongside a local corun-ai instance during development (both services share a network; `CORUN_BASE_URL` points to the Django container).

---

## MCP Client Configuration

### Claude Desktop (`claude_desktop_config.json`)

```json
{
  "mcpServers": {
    "corun": {
      "command": "corun-mcp-server",
      "env": {
        "CORUN_BASE_URL": "https://epic-devcloud.org/doc",
        "CORUN_API_TOKEN": "<your-token>"
      }
    }
  }
}
```

### Docker-based (Streamable HTTP transport, future)

```json
{
  "mcpServers": {
    "corun": {
      "type": "http",
      "url": "http://localhost:8001/mcp"
    }
  }
}
```

---

## Development Sequence

The work splits naturally into two phases.

### Phase 1 — corun-ai REST API (in this repository)

1. Add DRF to dependencies and wire up `api/` URL namespace.
2. Implement the read-only API endpoints with token auth.
3. Implement the write endpoints (`POST /api/v1/prompts/`, `POST /api/v1/jobs/`).
4. Add `create_api_token` management command.
5. Add API endpoint tests (`tests/api/`).
6. Update deployment and README docs.

### Phase 2 — corun-mcp-server (new repository)

1. Bootstrap repo: `pyproject.toml`, `src/corun_mcp_server/`, `tests/`, `AGENTS.md`, `Dockerfile`, `docker-compose.yml`.
2. Implement `client.py` with `httpx.AsyncClient`, auth, and base URL handling.
3. Implement `resources.py` (read tools logic).
4. Implement `jobs.py` (write/poll logic).
5. Implement `server.py` — register all tools with `@mcp.tool()`, wire up `main()`.
6. Write full test suite; reach ≥ 90 % coverage.
7. Set up GitHub Actions CI and Docker publish workflows.
8. Write `README.md` and `AGENTS.md`.

---

## Security Considerations

- Tokens must be treated as secrets; never hardcode them or commit them to source control.
- The corun-ai API should enforce per-token rate limits to prevent runaway job submission.
- Write endpoints should validate that the target Section and JobDefinition are in `active` status before creating rows.
- The MCP server itself has no persistent state — it is stateless between tool calls.
- Future enhancement: token scopes (read-only vs. read-write) to allow clients that only need browsing to be issued restricted tokens.

---

## Resources

- MCP Specification: https://modelcontextprotocol.io/
- FastMCP (Python SDK): https://github.com/modelcontextprotocol/python-sdk
- uproot-mcp-server (reference): https://github.com/eic/uproot-mcp-server
- zenodo-mcp-server (reference, TypeScript): https://github.com/eic/zenodo-mcp-server
- Django REST Framework: https://www.django-rest-framework.org/
- httpx: https://www.python-httpx.org/
- respx (httpx mock): https://lundberg.github.io/respx/

# Claude Code Prompt: Revvity Signals Notebook MCP Server

## Project Overview

Build a Python MCP (Model Context Protocol) server that exposes Revvity Signals Notebook's REST API as named semantic tools. This allows AI agents (Claude Code, Claude Desktop, etc.) to read, search, create, and modify electronic lab notebook entities programmatically.

**Project name:** `signals-notebook-mcp`
**Location:** `~/projects/signals-notebook-mcp`
**Transport:** stdio (primary), streamable-http (secondary)
**Package manager:** uv

## Tech Stack

- **Python 3.12+**
- **MCP SDK:** `mcp[cli]` (latest, currently 1.27.x) — use `FastMCP` from `mcp.server.fastmcp`
- **HTTP client:** `httpx` (async)
- **Testing:** `pytest`, `pytest-asyncio`, `respx` (for httpx mocking)
- **Config:** Environment variables via `pydantic-settings`

## Project Structure

```
signals-notebook-mcp/
├── pyproject.toml
├── README.md
├── .env.example
├── src/
│   └── signals_notebook_mcp/
│       ├── __init__.py
│       ├── server.py          # FastMCP server instance + tool registrations
│       ├── client.py          # Async httpx wrapper for Signals Notebook API
│       ├── config.py          # Pydantic Settings for env vars
│       ├── models.py          # Pydantic models for API request/response shapes
│       └── tools/
│           ├── __init__.py
│           ├── entities.py    # Entity CRUD tools
│           ├── search.py      # Search tools (text + chemical)
│           ├── notifications.py  # Notification tools
│           └── users.py       # User lookup tools
└── tests/
    ├── __init__.py
    ├── conftest.py            # Shared fixtures (mock client, respx router)
    ├── test_client.py         # Tests for the HTTP client wrapper
    ├── test_entities.py       # Tests for entity tools
    ├── test_search.py         # Tests for search tools
    ├── test_notifications.py  # Tests for notification tools
    └── test_users.py          # Tests for user tools
```

## Configuration (config.py)

Use `pydantic-settings` `BaseSettings` with env prefix `SIGNALS_`:

```python
class SignalsConfig(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="SIGNALS_")

    base_url: str  # e.g. "https://snb.example.com"
    api_key: str   # The x-api-key value
    api_version: str = "v1.0"  # REST API version path segment
    timeout: float = 30.0  # HTTP request timeout in seconds
    rate_limit_rpm: int = 900  # Stay under 1000/min tenant limit
```

`.env.example`:
```
SIGNALS_BASE_URL=https://your-tenant.signals.revvitysignals.com
SIGNALS_API_KEY=your-api-key-here
SIGNALS_API_VERSION=v1.0
SIGNALS_TIMEOUT=30.0
```

## HTTP Client (client.py)

Async wrapper class `SignalsClient` around httpx:

```python
class SignalsClient:
    """Async HTTP client for Revvity Signals Notebook REST API."""

    def __init__(self, config: SignalsConfig) -> None: ...

    # Properties
    @property
    def api_base(self) -> str:
        """Base URL for API requests: {base_url}/api/rest/{api_version}"""

    # Core HTTP methods — all return parsed JSON dicts
    async def get(self, path: str, params: dict | None = None) -> dict: ...
    async def post(self, path: str, json: dict | None = None, params: dict | None = None) -> dict: ...
    async def patch(self, path: str, json: dict, params: dict | None = None) -> dict: ...
    async def delete(self, path: str, params: dict | None = None) -> dict: ...

    # Lifecycle
    async def close(self) -> None: ...
```

Requirements:
- Set `x-api-key` header on all requests from config
- Set `Content-Type: application/vnd.api+json` (JSON:API)
- Set `Accept: application/vnd.api+json`
- Raise a descriptive error on non-2xx responses, including the Signals error body (status, code, title, detail) when available
- Include `digest` as a query parameter when provided to methods (for concurrent edit safety)
- Support `force=true` query parameter option
- Handle rate limit awareness (log warnings as requests approach the limit)

## Pydantic Models (models.py)

Define models for the key request/response shapes:

```python
class SearchQuery(BaseModel):
    """A Signals entity search query body."""
    query: dict  # The $and/$match/$child/$chemsearch query tree
    options: dict | None = None  # sort, pagination

class EntitySummary(BaseModel):
    """Minimal entity info extracted from API responses."""
    eid: str
    name: str
    entity_type: str
    state: str | None = None
    created_at: str | None = None
    edited_at: str | None = None
    description: str | None = None

class NotificationSummary(BaseModel):
    """Minimal notification info."""
    id: str
    type: str
    is_dismissed: bool
    created_at: str
    entity_eid: str | None = None
    user_email: str | None = None
    comment: str | None = None

class SearchMatch(BaseModel):
    """A single $match clause."""
    field: str
    value: str | bool
    mode: str | None = None
    in_field: str | None = None  # maps to "in" key
    as_type: str | None = None   # maps to "as" key
```

These are for internal convenience — tools return raw JSON:API dicts to the agent so it has full access to relationships and included data.

## MCP Tools

### Server Entry Point (server.py)

```python
from mcp.server.fastmcp import FastMCP
from signals_notebook_mcp.config import SignalsConfig
from signals_notebook_mcp.client import SignalsClient

mcp = FastMCP("signals-notebook")

# Instantiate config and client at module level
config = SignalsConfig()
client = SignalsClient(config)

# Import tool modules to register them
import signals_notebook_mcp.tools.entities
import signals_notebook_mcp.tools.search
import signals_notebook_mcp.tools.notifications
import signals_notebook_mcp.tools.users

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

Each tool module imports `mcp` and `client` from `server.py` and registers tools via `@mcp.tool()`.

### Entity Tools (tools/entities.py)

Implement these tools:

#### `get_entity`
- **Description:** Retrieve a Signals Notebook entity by its entity ID (eid). Returns the full JSON:API response including attributes, relationships, and included resources.
- **Params:** `eid: str` — Entity ID, e.g. "experiment:03ae2d17-e94d-466a-ba83-94d89a3cea2f"
- **Returns:** Full JSON:API response dict
- **API call:** `GET /entities/{eid}`

#### `get_entity_children`
- **Description:** List the child entities of a given parent entity. Experiments contain children like text blocks, chemical drawings, tables, and sample containers.
- **Params:** `eid: str` — Parent entity ID
- **Returns:** JSON:API response with array of child entities
- **API call:** `GET /entities/{eid}/children`

#### `create_entity`
- **Description:** Create a new entity (experiment, notebook, etc.) as a child of a parent entity. Provide the entity type, name, and optional description.
- **Params:** `parent_eid: str`, `entity_type: str`, `name: str`, `description: str = ""`
- **Returns:** The created entity JSON:API response
- **API call:** `POST /entities/{parent_eid}/children` with JSON:API body
- **Note:** Construct JSON:API compliant request body:
  ```json
  {
    "data": {
      "type": "entity",
      "attributes": {
        "type": "<entity_type>",
        "name": "<name>",
        "description": "<description>"
      }
    }
  }
  ```

#### `update_entity`
- **Description:** Update an entity's attributes (name, description, or custom fields). Requires the entity's current digest value for concurrency control.
- **Params:** `eid: str`, `digest: str`, `name: str | None = None`, `description: str | None = None`, `fields: dict | None = None`, `force: bool = False`
- **Returns:** Updated entity JSON:API response
- **API call:** `PATCH /entities/{eid}?digest={digest}` (add `&force=true` if force=True)

#### `delete_entity`
- **Description:** Trash/delete an entity by its ID. Requires the entity's current digest.
- **Params:** `eid: str`, `digest: str`, `force: bool = False`
- **Returns:** Confirmation dict
- **API call:** `DELETE /entities/{eid}?digest={digest}`

#### `export_entity_pdf`
- **Description:** Get a PDF export URL for a top-level entity (experiment, notebook, etc.).
- **Params:** `eid: str`
- **Returns:** The PDF download URL from the entity's relationships.pdf.links.self
- **Implementation:** Call `get_entity`, extract the PDF link from `data.relationships.pdf.links.self`, return it. The agent or downstream tool can then fetch the PDF via that URL.

### Search Tools (tools/search.py)

#### `search_entities`
- **Description:** Search for entities in Signals Notebook using flexible query criteria. Supports filtering by type, name, state, custom fields, creation/modification dates, and more. Returns paginated results.
- **Params:**
  - `entity_type: str = "experiment"` — Entity type to search for
  - `name: str | None = None` — Filter by entity name (exact or partial)
  - `state: str | None = None` — Filter by state (e.g. "open", "closed")
  - `custom_fields: dict | None = None` — Dict of field_name: value for custom field matching
  - `sort_by: str = "modifiedAt"` — Sort field
  - `sort_order: str = "desc"` — "asc" or "desc"
  - `include_templates: bool = False` — Whether to include template entities
- **Returns:** JSON:API response with matching entities
- **Implementation:** Build the `$and` query array from provided params. Always include `$match` for `type` and `isTemplate`. Add additional `$match` clauses for each non-None filter param. POST to `/entities/search`.

#### `search_by_molecule`
- **Description:** Search for experiments containing a specific molecular structure in their chemical drawings. Uses SMILES notation for chemical structure matching.
- **Params:**
  - `smiles: str` — SMILES string representing the molecule (e.g. "C1=CC=CC=C1" for benzene)
  - `full_match: bool = True` — If True, require exact structure match; if False, allow substructure search
  - `entity_type: str = "experiment"` — Entity type to search within
- **Returns:** JSON:API response with matching entities
- **Implementation:** Build query with `$child` containing `$chemsearch`:
  ```json
  {
    "$child": {
      "$chemsearch": {
        "molecule": "<smiles>",
        "mime": "chemical/x-daylight-smiles",
        "options": "full=<true|false>"
      }
    }
  }
  ```

#### `raw_search`
- **Description:** Execute a raw search query against the Signals search API. Use this for complex queries that don't fit the structured search_entities tool. Provide the full query body as a JSON dict.
- **Params:** `query_body: dict` — Complete search request body matching the Signals search API spec
- **Returns:** JSON:API response
- **API call:** `POST /entities/search` with the provided body

### Notification Tools (tools/notifications.py)

#### `list_notifications`
- **Description:** Retrieve notifications from Signals Notebook. Supports filtering by dismissed/read status.
- **Params:**
  - `dismissed: bool | None = None` — Filter by dismissed status
  - `read: bool | None = None` — Filter by read status
  - `limit: int = 50` — Max notifications to return
- **Returns:** JSON:API response with notification array
- **API call:** `GET /notifications` with query params for filtering

#### `dismiss_notification`
- **Description:** Mark a notification as dismissed.
- **Params:** `notification_id: str`
- **Returns:** Updated notification
- **API call:** `PATCH /notifications/{notification_id}` with `{"data": {"attributes": {"isDismissed": true}}}`

#### `get_notification`
- **Description:** Get a single notification by ID.
- **Params:** `notification_id: str`
- **Returns:** Full notification JSON:API response
- **API call:** `GET /notifications/{notification_id}`

### User Tools (tools/users.py)

#### `list_users`
- **Description:** List users in the Signals Notebook tenant.
- **Params:** `limit: int = 50`, `offset: int = 0`
- **Returns:** JSON:API response with user array
- **API call:** `GET /users`

#### `get_user`
- **Description:** Get details for a specific user by user ID.
- **Params:** `user_id: str`
- **Returns:** JSON:API user response
- **API call:** `GET /users/{user_id}`

## Testing Requirements

**100% test coverage.** Use `pytest` + `pytest-asyncio` + `respx` (httpx mock).

### conftest.py Fixtures

```python
@pytest.fixture
def config():
    """Test config with fake values."""
    return SignalsConfig(
        base_url="https://test.signals.example.com",
        api_key="test-api-key-12345",
    )

@pytest.fixture
def mock_client(config):
    """SignalsClient with respx mocked transport."""
    # Use respx.mock to intercept all httpx requests
    ...

@pytest.fixture
def sample_experiment_response():
    """Full JSON:API response for a test experiment."""
    # Return the example from the Signals docs
    ...

@pytest.fixture
def sample_search_response():
    """Search results response."""
    ...

@pytest.fixture
def sample_notification_response():
    """Push notification response."""
    ...
```

### Test Categories

For each tool, test:
1. **Happy path** — correct API call made, response properly returned
2. **Parameter variations** — optional params included/excluded, defaults applied
3. **Error handling** — non-2xx responses raise descriptive errors with Signals error body
4. **Digest/concurrency** — update/delete tools include digest param correctly
5. **Force override** — force=True adds query param

For the client, test:
1. **Headers** — x-api-key, Content-Type, Accept set correctly
2. **URL construction** — base_url + api_version + path assembled correctly
3. **Error parsing** — Signals error JSON extracted and included in exception
4. **Timeout** — configured timeout applied

For search tools specifically, test:
1. **Query construction** — verify the $and/$match/$chemsearch body is built correctly from params
2. **Template exclusion** — isTemplate=false included by default
3. **Custom fields** — custom_fields dict maps to correct $match with `in: "tags"` and `as: "text"`
4. **SMILES search** — $child.$chemsearch body constructed correctly

## Documentation

### README.md

Include:
1. What this server does (one paragraph)
2. Prerequisites (Python 3.12+, uv, Signals Notebook tenant with API key)
3. Installation: `uv sync`
4. Configuration: copy `.env.example` to `.env`, fill in values
5. Running: `uv run python -m signals_notebook_mcp.server` (stdio) or with transport flag
6. Claude Desktop config snippet:
   ```json
   {
     "mcpServers": {
       "signals-notebook": {
         "command": "uv",
         "args": ["run", "--directory", "/path/to/signals-notebook-mcp", "python", "-m", "signals_notebook_mcp.server"],
         "env": {
           "SIGNALS_BASE_URL": "https://your-tenant.signals.revvitysignals.com",
           "SIGNALS_API_KEY": "your-api-key"
         }
       }
     }
   }
   ```
7. Claude Code config: `claude mcp add signals-notebook -- uv run --directory /path/to/signals-notebook-mcp python -m signals_notebook_mcp.server`
8. Available tools table with name, description, parameters
9. Running tests: `uv run pytest -v --cov`

### pyproject.toml

```toml
[project]
name = "signals-notebook-mcp"
version = "0.1.0"
description = "MCP server exposing Revvity Signals Notebook REST API as AI agent tools"
requires-python = ">=3.12"
dependencies = [
    "mcp[cli]>=1.20",
    "httpx>=0.27",
    "pydantic-settings>=2.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.24",
    "pytest-cov>=5.0",
    "respx>=0.22",
    "ruff>=0.8",
]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]

[tool.ruff]
line-length = 120
target-version = "py312"

[tool.coverage.run]
source = ["src/signals_notebook_mcp"]

[tool.coverage.report]
fail_under = 100
show_missing = true
```

## Signals Notebook API Reference (for implementation context)

### Base URL Pattern
`{base_url}/api/rest/{api_version}/`

### Authentication
Header: `x-api-key: {api_key}`

### Key Endpoints
- `GET /entities/{eid}` — Fetch entity by ID
- `GET /entities/{eid}/children` — List entity children
- `POST /entities/{parent_eid}/children` — Create child entity
- `PATCH /entities/{eid}?digest={digest}` — Update entity
- `DELETE /entities/{eid}?digest={digest}` — Delete entity
- `POST /entities/search` — Search entities (body contains query)
- `GET /notifications` — List notifications
- `GET /notifications/{id}` — Get notification
- `PATCH /notifications/{id}` — Update notification flags
- `GET /users` — List users
- `GET /users/{id}` — Get user

### Entity ID Format
`{type}:{uuid}` — e.g. `experiment:03ae2d17-e94d-466a-ba83-94d89a3cea2f`

Entity types: notebook, experiment, text, chemicalDrawing, grid (table), samplesContainer, request, parallelExperiment, image, file, excelDoc

### JSON:API Response Structure
All responses follow JSON:API spec with top-level `data`, `links`, `included` objects. The `data.attributes` contains entity fields. The `data.relationships` contains links to related resources. The `included` array contains the full objects referenced in relationships.

### Concurrency Control
Entities include a `digest` value in attributes. When updating/deleting, pass `digest` as query parameter. Server returns 428 if digest doesn't match (concurrent edit detected). Pass `force=true` to override.

### Search Query Format
POST body with `query` (containing `$and` array of `$match`/`$child`/`$chemsearch` clauses) and optional `options` (sort, pagination).

### Rate Limits
1000 requests/minute per tenant across all API keys and bearer tokens.

## Execution Instructions

1. Build the entire project in a single pass
2. All files complete — no TODOs, no placeholders, no stubs
3. Run `uv sync` to install dependencies
4. Run `uv run pytest -v --cov` — all tests must pass with 100% coverage
5. Run `uv run ruff check src/ tests/` — no lint errors
6. Verify the server starts: `uv run python -m signals_notebook_mcp.server`

# Server Stack Guide

Recommended Python stack for the backend server of a web app built with this kit.

**This guide only applies if you picked the `agent-sdk` or `claude-api` engine.** The `claude-code` engine doesn't use `app/server/` at all — agents run in the CLI and produce artifacts (e.g. the static site under `app/site/`), so there's nothing for a server to do.

For projects that *do* have a server, the server is the **data and API layer**. Agent runtime hosting belongs to the engine; the server stays orthogonal — it serves data, persists state, and exposes endpoints the runtime and the client call.

## At a glance

| Concern | Choice |
|---|---|
| Language | Python 3.14+ |
| Dependency management | `uv` |
| Web framework | FastAPI |
| ASGI server | `uvicorn[standard]` |
| Data layer | SQLAlchemy 2.x (async) + `asyncpg` |
| Migrations | Alembic (async mode) |
| Database | PostgreSQL |
| Validation / config | Pydantic v2 + `pydantic-settings` |
| Testing | `pytest` + `pytest-asyncio` |
| Lint + format | `ruff` |
| Type checking | `mypy` |
| Observability | OpenTelemetry + `structlog` |
| Auth | Out of scope — bring your own |

## What the server owns

- HTTP API surface (REST + JSON, plus SSE for streaming responses)
- Persistence (relational data, migrations)
- Domain logic and validation
- Outbound integrations with external services
- Structured logs, traces, and metrics

## What the server does *not* own

- **Agent runtime definition** — see `guides/engines/`. The runtime may execute inside the server process (it usually does for the `agent-sdk` engine, where a FastAPI handler invokes the SDK directly), but *defining* it — system prompts, tool registration, agent topology — belongs to the engine guide, not the server.
- **Client UI** — see `guides/client/`.
- **Auth** — bring your own IdP / token verifier. Wire it in as a FastAPI dependency at the router boundary.

## Stack

### Python 3.14+

Pin the Python version with a `.python-version` file. 3.14 is the current default; 3.13 is acceptable if a dependency forces it.

### `uv` for dependency management

Use [`uv`](https://docs.astral.sh/uv/) for environment and dependency management. It replaces `pip`, `pip-tools`, `virtualenv`, and `poetry` with a single fast tool, and produces a deterministic `uv.lock`. Declare dependencies in `pyproject.toml`; commit `uv.lock`.

```sh
uv sync                    # install/update from lock
uv run pytest              # run a command in the project env
uv add httpx               # add a runtime dep
uv add --dev mypy          # add a dev dep
uv tool install ruff       # install a CLI globally (isolated env, pipx-style)
```

Prefer `uv add --dev` for tools that need to be pinned per-project (e.g. `mypy`, `pytest`, `alembic`); reserve `uv tool install` for general-purpose CLIs you want available across all projects.

### FastAPI

[FastAPI](https://fastapi.tiangolo.com/) is the framework. Reasons:

- **Async-native**, which matters when the server fans out to LLM APIs and other slow I/O.
- **Pydantic-based** request/response models give you validation and OpenAPI docs for free.
- **Dependency injection** via `Depends(...)` is the right shape for things like database sessions, auth, and request-scoped resources.

Organize routes in `routers/`, one module per resource. Keep handlers thin; push logic into `services/`.

### `uvicorn[standard]`

Run FastAPI under `uvicorn[standard]` (which pulls in `httptools` and `uvloop`). Dev: `uv run uvicorn app.main:app --reload`. Prod: run multiple uvicorn workers behind your platform's load balancer; do **not** add gunicorn unless you have a specific reason.

### SQLAlchemy 2.x async + `asyncpg`

Use SQLAlchemy 2.x with the async API (`AsyncEngine`, `AsyncSession`) and the `asyncpg` driver.

**Why async, not sync.** FastAPI handlers are async; if the database layer is sync, every query blocks the event loop and you lose most of FastAPI's concurrency benefit. Agentic workloads are heavily I/O-bound (LLM calls, tool calls, external APIs), so the cost of mixing in a blocking DB call is high. The async API is mature in SQLAlchemy 2.x; pay the small upfront ergonomics cost once and stay consistent.

Yield `AsyncSession` instances via a FastAPI dependency. One session per request. Commit at the boundary, not inside services.

### Alembic (async mode)

[Alembic](https://alembic.sqlalchemy.org/) for migrations, configured for async (`env.py` uses `async_engine_from_config`). Migration files are committed; `alembic upgrade head` runs on deploy. Never edit applied migrations — write a new one.

### PostgreSQL

PostgreSQL is the only first-class database. SQLite is acceptable as a test fallback (in-memory, `StaticPool`) but production assumes Postgres. Don't write migrations or queries that depend on SQLite-only behavior.

### Pydantic v2 + `pydantic-settings`

- **Pydantic v2** for request/response schemas, distinct from SQLAlchemy ORM models. Keep them separate: ORM models in `models/`, Pydantic schemas in `schemas/`. Use `model_config = ConfigDict(from_attributes=True)` to map ORM → schema.
- **`pydantic-settings`** for configuration. One `Settings` class loaded from environment variables (with a project-specific prefix, e.g. `MYAPP_`). Read `.env` in development; in production, the platform injects env vars directly. Commit a `.env.example`; never commit `.env`.

### `pytest` + `pytest-asyncio`

- `asyncio_mode = "auto"` in `pyproject.toml` so async tests don't need decorators.
- Split tests into `tests/unit/` and `tests/integration/`. Unit tests don't touch the database or network; integration tests use a real Postgres (or a SQLite-in-memory fallback) and exercise full request cycles via FastAPI's `TestClient` / `httpx.AsyncClient`.
- Mock outbound HTTP with [`respx`](https://lundberg.github.io/respx/).
- Enforce a coverage floor (start at 80%) via `pytest-cov`.
- Fixtures in `conftest.py`: async DB engine, transactional session per test, app client, mocked external services.

### `ruff` for lint + format

[`ruff`](https://docs.astral.sh/ruff/) does both linting and formatting in one fast tool. No `black`, no `isort`, no `flake8`. Configure in `pyproject.toml`. Run via pre-commit and CI.

### `mypy` for type checking

[`mypy`](https://mypy.readthedocs.io/) in strict-ish mode (`strict = true`, with documented per-module relaxations). Annotate everything new; treat type errors as build failures. Pyright is faster but `mypy` has broader rank-and-file adoption, more pre-commit / CI tooling, and fewer surprises for contributors.

### OpenTelemetry + `structlog`

- **Structured logging** with [`structlog`](https://www.structlog.org/). Emit JSON in production; pretty-printed in dev. Log lines should be machine-parseable: include request IDs, user IDs (when authenticated), and operation names.
- **Tracing and metrics** with [OpenTelemetry](https://opentelemetry.io/). Use the FastAPI and SQLAlchemy auto-instrumentations; export OTLP to whatever backend you have (Honeycomb, Grafana Tempo, Jaeger, Datadog, etc.). Propagate trace context on outbound HTTP.

### Auth (out of scope)

This guide does not prescribe an auth solution. When you add one, wire it in as a FastAPI dependency on each protected router. Common options: JWT verification (Authlib, python-jose), Clerk/Auth0 backend SDKs, or a session middleware if the client is browser-only.

## Project layout

```
app/server/
  pyproject.toml
  uv.lock
  .python-version
  .env.example
  alembic.ini
  alembic/
    env.py
    versions/
  src/
    main.py            # FastAPI app, lifespan, exception handlers
    config.py          # Settings (pydantic-settings)
    database.py        # async engine + session dependency
    logging.py         # structlog + OpenTelemetry setup
    routers/           # one module per resource; thin handlers
    services/          # domain logic + external integrations
    models/            # SQLAlchemy ORM models
    schemas/           # Pydantic request/response models
    exceptions.py      # domain exceptions
  tests/
    conftest.py
    unit/
    integration/
```

## Patterns

### Lifespan context manager

Use FastAPI's `lifespan=` to set up and tear down resources (DB engine, HTTP clients, OTel providers). Don't use the deprecated `@app.on_event` hooks.

### Domain exceptions → HTTP responses

Define domain exceptions in `exceptions.py` (e.g. `EntryNotFoundError`, `QuotaExceededError`). Register exception handlers on the app that map each to a specific status code and response shape. Handlers stay free of HTTP concerns; routers stay free of error-shape concerns.

### Service layer for external integrations

Each external integration (LLM provider, third-party API, search index) gets a service class in `services/`. Construct once at startup (in the lifespan), inject via `Depends`, close on shutdown. Tests mock the service class, not the underlying HTTP library.

### Schema ↔ model separation

`models/` is SQLAlchemy. `schemas/` is Pydantic. Routers receive and return schemas; services accept and return models or domain types. Don't leak ORM rows out through the API.

### Settings as a single object

One `Settings` instance, loaded once, injected via `Depends(get_settings)` (cached with `lru_cache`). No reading `os.environ` directly outside `config.py`.

## Alternatives worth considering

- **Litestar** instead of FastAPI — more opinionated, dataclass-friendly, faster in some benchmarks. Smaller ecosystem.
- **Django + DRF** — pick this if you need the admin UI, batteries-included auth, and a more conventional sync model. Less natural fit for async LLM workloads.
- **Sync SQLAlchemy + `psycopg`** — simpler mental model. Acceptable if your handlers don't make outbound LLM/tool calls and aren't latency-sensitive. Not the default here because agentic apps almost always do.
- **`pyright`** instead of `mypy` — faster, what VS Code's Pylance uses under the hood. Choose this if your team is already on Pylance and prefers speed.
- **Poetry** or **PDM** instead of `uv` — both are mature. `uv` wins on speed and on consolidating multiple tools into one.
- **Celery / `arq`** if you need background work. Not in the default stack because most agentic apps start without it; add when you actually have long-running jobs that shouldn't block a request.

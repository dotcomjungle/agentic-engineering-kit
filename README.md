# Agentic Engineering Kit

A starting point / template for new agentic engineering projects.

This repo is an opinionated scaffold and playbook for building software where Claude is part of the runtime. Pick an engine, follow the matching guide, end up with a project that has a consistent shape and well-justified default choices.

## Pick an engine

Three mutually exclusive options for how the agent runs:

- **`claude-code`** — interact with Claude directly through the Claude Code CLI. Agents are defined as `app/agents/<name>/prompt.md`. Artifacts are published as a static Astro site under `app/site/`. No server, no SPA client.
- **`agent-sdk`** — Anthropic's Claude Agent SDK runs inside your application. Python backend, web UI.
- **`claude-api`** — custom runtime built directly on the Claude API. Python backend, web UI. Choose this when the SDK's shape doesn't fit and you need full control over the loop.

Each lives under `guides/engines/<choice>/`.

## Default stack

For the two engines that ship a web project (`agent-sdk` and `claude-api`):

- **Server** — Python 3.14+, FastAPI, SQLAlchemy 2.x async + asyncpg, PostgreSQL, `uv`, `ruff`, `mypy`, `pytest`, OpenTelemetry + `structlog`. Full rationale in `guides/server/README.md`.
- **Client** — see `guides/client/README.md`.

The `claude-code` engine doesn't use `app/server/` or `app/client/` at all, so neither stack guide applies to it.

## How to use this kit

1. Read `guides/engines/` and pick an engine.
2. Follow that engine's guide. For `agent-sdk` or `claude-api`, also read `guides/server/` and `guides/client/`.
3. Build out `app/` in the shape the engine prescribes.
4. Replace this `README.md` with your project's own when you're ready.

## Layout

`CLAUDE.md` at the repo root has the canonical directory layout and is what Claude Code reads when working in this repo.

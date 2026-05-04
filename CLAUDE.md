# Agentic Engineering Kit

A starting point / template for new agentic engineering projects.

## Layout

Two top-level concerns: **how to set the project up** (`guides/`) and **what the project becomes** (`app/`).

### `guides/` — setup playbooks and stack recommendations

- `guides/engines/` — three mutually exclusive playbooks for the agentic runtime. Pick one:
  - `guides/engines/claude-code/` — interact directly via the Claude Code CLI. Agents are defined as `app/agents/<name>/prompt.md`; artifacts are published as a static site under `app/site/` (Astro).
  - `guides/engines/agent-sdk/` — Anthropic Claude Agent SDK as the runtime, driven through an application UI. Agentic software written in python.
  - `guides/engines/claude-api/` — custom runtime built on the Claude API, driven through an application UI. Agentic software written in python.
- `guides/client/` — recommended (default) frontend tech stack and a discussion of alternatives. **Applies to `agent-sdk` and `claude-api` engines only.**
- `guides/server/` — recommended (default) backend tech stack (Python: FastAPI, SQLAlchemy 2.x async, Postgres, `uv`). **Applies to `agent-sdk` and `claude-api` engines only.**

### `app/` — the generated project

The shape of `app/` depends on which engine you picked:

**`claude-code` engine** — no server, no SPA client.

- `app/agents/<name>/prompt.md` — system prompt defining each agent's role, goals, and behavior.
- `app/site/` — Astro static site for agent-authored artifacts. Source under `src/content/`; build output `dist/` is gitignored and never hand-edited.

**`agent-sdk` and `claude-api` engines** — a web project with a server and a client.

- `app/server/CLAUDE.md` — describes the **role** of the server in the app (what it's responsible for).
- `app/client/CLAUDE.md` — describes the **role** of the client in the app (what it's responsible for).

The split: `guides/` answers *"what should we build it with, and why"*; the `CLAUDE.md` files under `app/` answer *"what is this part of the app for"*. Stack-specific working instructions for the generated code live under `app/` once the stack is chosen.

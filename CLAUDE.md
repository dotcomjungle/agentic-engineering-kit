# Agentic Engineering Kit

A starting point / template for new agentic engineering projects.

## Layout

Two top-level concerns: **how to set the project up** (`guides/`) and **what the project becomes** (`app/`).

### `guides/` — setup playbooks and stack recommendations

- `guides/engines/` — three mutually exclusive playbooks for the agentic runtime. Pick one:
  - `guides/engines/claude-code/` — interact directly via the Claude Code CLI.
  - `guides/engines/agent-sdk/` — Anthropic Claude Agent SDK as the runtime, driven through an application UI.
  - `guides/engines/claude-api/` — custom runtime built on the Claude API, driven through an application UI.
- `guides/client/` — recommended (default) frontend tech stack and a discussion of alternatives.
- `guides/server/` — recommended (default) backend tech stack and a discussion of alternatives.

### `app/` — the generated web app

- `app/server/CLAUDE.md` — describes the **role** of the server in the app (what it's responsible for).
- `app/client/CLAUDE.md` — describes the **role** of the client in the app (what it's responsible for).

The split: `guides/{client,server}/` answers *"what should we build it with, and why"*; `app/{client,server}/CLAUDE.md` answers *"what is this part of the app for"*. Stack-specific working instructions for the generated code live under `app/` once the stack is chosen.

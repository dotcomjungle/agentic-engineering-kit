# Agentic Engineering Kit

A starting point / template for new agentic engineering projects.

## Role

Your role is to facilitate the development of an agentic engineering project. This repo is a meta-template; the actual project lives under `app/` once an engine is chosen.

## Goals

### Phase 1 — engine selection

On first contact, don't wait for the user to lead — they may not know what this kit expects of them. Open by:

1. Briefly explaining what this kit is and that the first decision is which agentic runtime engine to use.
2. Walking the user through the three options (`claude-code`, `agent-sdk`, `claude-api`). Show a comparison table if it helps.
3. Asking what you need in order to recommend one:
   - The user's coding experience (hands-on developer vs. vibe-coding through the agent).
   - Project goals — what they're trying to build, and for whom.
   - Output shape — content generation (artifacts, documents, a static site) vs. a transactional application with end users.
4. If the answers point clearly to one engine, say so — be opinionated. If it's genuinely a toss-up, lay out the tradeoffs and let the user pick. Do not fall back to a default; no engine should be picked silently.

### Phase 2 — scaffolding

Once an engine is picked, collaborate with the user to flesh out `app/` per the chosen engine guide. Part of scaffolding is creating the three entry-point scripts under `scripts/` — see the Layout section below. The first agent to build is the `concierge` — see the MVP section below.

### Phase 3 — development

Once `app/` is scaffolded, your role shifts from kit facilitator to development collaborator on the project the user is now building.

## MVP — concierge agent

When scaffolding the project in Phase 2, build a single first agent: the `concierge`. It is a friendly conversationalist whose role will grow into coordinating and orchestrating other agents as the agent roster expands. Future agents are added as **subagents dispatched by the concierge**, not peers — the user always has one conversation, and the concierge orchestrates internally (Claude Code's Task tool for `claude-code`; the SDK's `agents={}` for `agent-sdk`; a hand-rolled equivalent for `claude-api`).

Acceptance criteria differ by engine:

- **`claude-code`** — the concierge is defined at `app/agents/concierge/prompt.md` (YAML frontmatter + system prompt body); `scripts/setup.sh` mirrors it to `.claude/agents/concierge.md` (gitignored) so Claude Code's subagent system finds it. The user opens `claude` and asks to use the concierge — Claude routes to the subagent, which then runs with its own system prompt and configured tools.
- **`agent-sdk` or `claude-api`** — the user can interact with a chat window on the homepage of the SPA in which they chat with the concierge. Assistant messages stream as they arrive (message-level streaming via the SSE event taxonomy in `guides/engines/agent-sdk/`).

### Starter tools

The concierge ships with two tools at MVP. Both are read-only and dependency-free, and they double as worked examples of the engine's tool-definition idiom and as a smoke test that tool wiring works end-to-end.

- **`list_agents`** — returns the project's agent roster (name + one-line description per agent), read from the engine-appropriate registry. Day one this returns just `[concierge]`; as the roster grows it becomes the basis for routing and coordination.
- **`get_current_time(timezone?)`** — returns the current time in the requested IANA timezone (default UTC). LLMs lack a clock; this is the textbook minimal tool and a useful sanity check.

Implementation idiom is engine-specific — built-in `Glob` + `Read` may suffice for `list_agents` in `claude-code`; `@tool` + `create_sdk_mcp_server` for `agent-sdk`; JSON tool specs for `claude-api`. See the engine guide.

### System prompt

The concierge's system prompt is authored during scaffolding (not shipped as a fixed template) so it can reflect the user's actual project. Compose it to cover these elements:

- **Identity** — name (Concierge), role (friendly conversationalist + future orchestrator of other agents in this project), tone.
- **Project context** — instruct the concierge to read the root `README.md` and `CLAUDE.md` at session start so it knows what this project is and what it's for.
- **Available agents** — instruct the concierge to call `list_agents` and incorporate the result into how it responds. Day one the only result is itself; as the roster grows this is how it learns who to route to.
- **Tools** — describe `list_agents` and `get_current_time` and when to call each.
- **When to defer** — answer directly when it can; dispatch a subagent when one fits the task description better; ask the user when intent is ambiguous; never fabricate capabilities or claim agents exist that don't.

Don't pad the prompt beyond these elements. A long system prompt is a slow, expensive prompt — extend it deliberately when a real need arises.

## Filename conventions

This repo uses two filename conventions deliberately. Know the difference:

- **`README.md`** — documentation. Audience is humans browsing the repo, plus you when you need context. *Reference material, not instructions.* The root `README.md` is the project's front door; the `README.md` files under `guides/` explain the **rationale** behind each stack/engine choice (the *why*, plus alternatives considered). Read them for context. Don't treat them as a checklist to execute.
- **`CLAUDE.md`** — instructions to you. Loaded into your context automatically (this root one eagerly, nested ones lazily when you touch files in their subtree). Written as live rules for how to work in that part of the codebase. *Follow them.*

If a `README.md` and a `CLAUDE.md` appear to disagree on what you should do, `CLAUDE.md` wins.

### Implication for scaffolding

During Phase 2, part of your job is to distill the prescriptive bits of the relevant guide(s) into the new `app/*/CLAUDE.md` files. The guide stays as rationale; the `CLAUDE.md` becomes the live rulebook future-you reads when working in that subtree. Don't copy the guide wholesale — extract the rules, drop the alternatives discussion, and reference the guide for the *why*.

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

- `app/server/CLAUDE.md` — live rules for working in the server: role, conventions, working instructions distilled from `guides/server/`.
- `app/client/CLAUDE.md` — live rules for working in the client: role, conventions, working instructions distilled from `guides/client/`.

### `scripts/` — developer entry points

The scripts here are the canonical entry points for working with the project — for humans and for you. They abstract over the engine choice so anyone can set up, run, and validate the project without knowing whether it's `uv` + `npm`, Astro, or something else underneath. **Create all three when scaffolding the project in Phase 2** and tailor each to the chosen engine — every engine has meaningful work for setup, start, and validate, so don't omit any of the three. When the run/test/build story changes in Phase 3, keep them in sync — a stale `validate.sh` is worse than none.

- **`scripts/setup.sh`** — install dependencies and run idempotent seeds. Must be safe to re-run: `uv sync`, `npm install`, `alembic upgrade head`, and any seed data that's keyed/upserted. A new contributor should be one command from a working environment.
- **`scripts/start.sh`** — spin up all dev services and, when running interactively, open the app in the browser (skip the open if `$CI` is set or stdout isn't a TTY; use `open` on macOS, `xdg-open` elsewhere). For multi-service projects (`agent-sdk`, `claude-api`), start everything in parallel, **prefix each child's output with its service name** so interleaved logs stay readable (e.g. `npx concurrently --names server,client --prefix-colors blue,green ...`, or a `sed`-prefix per child), and ensure Ctrl-C cleans up children (`trap` + `wait`). For single-service projects (`claude-code`), this may just serve the static site.
- **`scripts/validate.sh`** — the pre-PR / pre-commit check. Run all linters, type checkers, test suites, and any production build. This is the correctness gate CI uses to decide whether a change can merge — not the full CI pipeline (deploy, secret scanning, etc. live in CI config, not here). If it's too slow for developers to run locally before pushing, it's the wrong check — split or speed it up rather than skip steps.

Conventions for every script in this directory:

- `#!/usr/bin/env bash` and `set -euo pipefail`. Fail fast.
- Resolve paths from the repo root so scripts work from any cwd. Standard idiom at the top of each script:

  ```bash
  cd "$(dirname "${BASH_SOURCE[0]}")/.."
  ```

- Exit non-zero on any failure. Don't swallow errors to keep going.
- Print a short banner before each step (`=== step name ===`) so failures are easy to locate in the output.

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

Once an engine is picked, collaborate with the user to flesh out `app/` per the chosen engine guide.

### Phase 3 — development

Once `app/` is scaffolded, your role shifts from kit facilitator to development collaborator on the project the user is now building.

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

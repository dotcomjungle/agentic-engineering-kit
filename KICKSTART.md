# Kickstart playbook

This file is the kit's kickstart playbook — the instructions Claude follows when a fresh clone of this repo has not yet been built out into a real project. Once `app/` contains implementation code, this file is no longer needed in context (the root `CLAUDE.md` only references it conditionally).

If you are reading this, the project has not been kickstarted yet. Work through the phases below with the user.

## Phase 1 — engine selection

On first contact, don't wait for the user to lead — they may not know what this kit expects of them. Open by:

1. Briefly explaining what this kit is and that the first decision is which agentic runtime engine to use.
2. Walking the user through the three options (`claude-code`, `agent-sdk`, `claude-api`). Show a comparison table if it helps.
3. Asking what you need in order to recommend one:
   - The user's coding experience (hands-on developer vs. vibe-coding through the agent).
   - Project goals — what they're trying to build, and for whom.
   - Output shape — content generation (artifacts, documents, a static site) vs. a transactional application with end users.
4. If the answers point clearly to one engine, say so — be opinionated. If it's genuinely a toss-up, lay out the tradeoffs and let the user pick. Do not fall back to a default; no engine should be picked silently.

## Phase 2 — kickstart

Once an engine is picked, collaborate with the user to build out `app/` per the chosen engine guide. Part of the kickstart is creating the three entry-point scripts under `scripts/` — see the Layout section in the root `CLAUDE.md`. The first agent to build is the `concierge` — see the MVP section below.

During the kickstart, also distill the prescriptive bits of the relevant guide(s) into the `app/*/CLAUDE.md` files. The guide stays as rationale; the `CLAUDE.md` becomes the live rulebook future-you reads when working in that subtree. Don't copy the guide wholesale — extract the rules, drop the alternatives discussion, and reference the guide for the *why*.

## MVP — concierge agent

When kickstarting the project, build a single first agent: the `concierge`. It is a friendly conversationalist whose role will grow into coordinating and orchestrating other agents as the agent roster expands. Future agents are added as **subagents dispatched by the concierge**, not peers — the user always has one conversation, and the concierge orchestrates internally (Claude Code's Task tool for `claude-code`; the SDK's `agents={}` for `agent-sdk`; a hand-rolled equivalent for `claude-api`).

Acceptance criteria differ by engine:

- **`claude-code`** — the concierge is defined at `app/agents/concierge/prompt.md` (YAML frontmatter + system prompt body); `scripts/setup.sh` mirrors it to `.claude/agents/concierge.md` (gitignored) so Claude Code's subagent system finds it. The user opens `claude` and asks to use the concierge — Claude routes to the subagent, which then runs with its own system prompt and configured tools.
- **`agent-sdk` or `claude-api`** — the user can interact with a chat window on the homepage of the SPA in which they chat with the concierge. Assistant messages stream as they arrive (message-level streaming via the SSE event taxonomy in `guides/engines/agent-sdk/`).

### Starter tools

The concierge ships with two tools at MVP. Both are read-only and dependency-free, and they double as worked examples of the engine's tool-definition idiom and as a smoke test that tool wiring works end-to-end.

- **`list_agents`** — returns the project's agent roster (name + one-line description per agent), read from the engine-appropriate registry. Day one this returns just `[concierge]`; as the roster grows it becomes the basis for routing and coordination.
- **`get_current_time(timezone?)`** — returns the current time in the requested IANA timezone (default UTC). LLMs lack a clock; this is the textbook minimal tool and a useful sanity check.

Implementation idiom is engine-specific — built-in `Glob` + `Read` may suffice for `list_agents` in `claude-code`; `@tool` + `create_sdk_mcp_server` for `agent-sdk`; JSON tool specs for `claude-api`. See the engine guide.

### System prompt

The concierge's system prompt is authored during the kickstart (not shipped as a fixed template) so it can reflect the user's actual project. Compose it to cover these elements:

- **Identity** — name (Concierge), role (friendly conversationalist + future orchestrator of other agents in this project), tone.
- **Project context** — instruct the concierge to read the root `README.md` and `CLAUDE.md` at session start so it knows what this project is and what it's for.
- **Available agents** — instruct the concierge to call `list_agents` and incorporate the result into how it responds. Day one the only result is itself; as the roster grows this is how it learns who to route to.
- **Tools** — describe `list_agents` and `get_current_time` and when to call each.
- **When to defer** — answer directly when it can; dispatch a subagent when one fits the task description better; ask the user when intent is ambiguous; never fabricate capabilities or claim agents exist that don't.

Don't pad the prompt beyond these elements. A long system prompt is a slow, expensive prompt — extend it deliberately when a real need arises.

## After kickstart

Once `app/` contains implementation code, the kickstart is done. The root `CLAUDE.md` becomes the steady-state instruction file; this `KICKSTART.md` is no longer loaded into context on subsequent sessions. Your role shifts from kit facilitator to development collaborator on the project the user is now building.

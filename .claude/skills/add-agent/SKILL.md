---
name: add-agent
description: Add a new agent definition to the project. Use when the user asks to create, add, or scaffold a new agent (e.g., "please add a UX Designer agent", "create a code reviewer agent", "add an agent for security review").
---

When invoked, create a new agent under `app/agents/<slug>/prompt.md`, drafting a role-specific system prompt from the requested role name.

## 1. Verify the engine

This skill supports the `claude-code` engine only. Detect:

- `app/agents/` exists (even if empty) → claude-code engine. Proceed.
- `app/agents/` does not exist, but `app/server/` contains Python source (`pyproject.toml`, `*.py`) → `agent-sdk` or `claude-api` engine. Refuse: tell the user this skill only supports `claude-code` today and that adding agents to those engines is still manual.
- `app/` contains only documentation stubs (`app/server/CLAUDE.md`, `app/client/CLAUDE.md`) and no implementation → project hasn't been kickstarted. Tell the user to read `KICKSTART.md` and kickstart first.

## 2. Determine the slug

Convert the role name to kebab-case: lowercase, hyphens between words, strip punctuation. Examples:

- "UX Designer" → `ux-designer`
- "Code Reviewer" → `code-reviewer`
- "Security Auditor" → `security-auditor`

If `app/agents/<slug>/` already exists, stop and ask the user before overwriting.

## 3. Write `app/agents/<slug>/prompt.md`

Frontmatter:

- `name` — must equal the directory slug.
- `description` — a one-sentence capability statement written as routing copy. Claude routes to the agent when a task matches this string, so describe what the agent is good at, not who it is. Avoid biographical bios. Good: "Reviews UI flows and proposes user-experience improvements grounded in usability heuristics." Bad: "A UX designer with ten years of experience."
- Omit `tools` and `model` unless the user specified them. The user can add later if they want to restrict tools or override the model.

Body — draft a 6–15 line system prompt tailored to the role. Cover:

- The agent's primary responsibility (one paragraph).
- Concrete outputs or behaviors expected (e.g. "produces accessibility findings as a bulleted list with WCAG references").
- Working style or tone if relevant.
- A short "do" / "don't" list when it adds clarity.

Draft from the role name — a UX Designer's prompt must differ meaningfully from a Security Auditor's. Don't ship a generic empty stub.

## 4. Mirror into `.claude/agents/`

Run `scripts/setup.sh` if it exists. The script mirrors `app/agents/<slug>/prompt.md` to `.claude/agents/<slug>.md`, which is what Claude Code's subagent system reads.

If `scripts/setup.sh` does not exist yet, do the mirror manually:

```bash
mkdir -p .claude/agents
cp "app/agents/<slug>/prompt.md" ".claude/agents/<slug>.md"
```

…and tell the user that `scripts/setup.sh` is missing and should be created (point them at `CLAUDE.md` and the engine guide).

## 5. Report back

In one short message, tell the user:

- The path you created.
- A one-sentence summary of the drafted prompt.
- A nudge to read and refine — the draft is a starting point.

Do not commit the change unless the user explicitly asks.

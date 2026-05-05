# Claude Code Harness Guide

Playbook for initializing an agentic engineering project that defines system prompts and tools but is interacted with directly in the Claude Code CLI.

## Agent definitions

Each agent lives in its own directory under `app/agents/`. The directory holds the agent's source — at minimum a `prompt.md` with YAML frontmatter + system-prompt body, plus per-agent assets (custom tool implementations, fixtures, docs) as they accrue.

Example: `app/agents/a11y-advocate/prompt.md`

```markdown
---
name: a11y-advocate
description: Reviews UI work for accessibility issues and suggests WCAG-aligned fixes.
tools: Read, Glob, Grep
model: sonnet
---

You are the accessibility advocate for this project. Your job is to...
```

Frontmatter fields match Claude Code's subagent metadata:

- `name` (required) — must match the directory name.
- `description` (required) — Claude routes to this agent when a task matches the description, so write it as a clear capability statement, not a bio.
- `tools` (optional) — comma-separated list of allowed tools. Omit to inherit the parent's full toolset.
- `model` (optional) — `sonnet`, `opus`, or `haiku` to override the default.

Refer to Claude Code's current subagent docs for the authoritative frontmatter spec — fields evolve.

### How invocation works

Claude Code's subagent system reads from `.claude/agents/<name>.md` (project-level) or `~/.claude/agents/<name>.md` (user-level). The kit's source of truth lives at `app/agents/<name>/prompt.md`; `.claude/agents/<name>.md` is **generated** from it. The reason for the indirection: `app/agents/<name>/` is a directory, which gives each agent a place to host custom tools, fixtures, and docs alongside its prompt — Claude Code's subagent location is a flat single-file format that doesn't.

`scripts/setup.sh` mirrors every agent into `.claude/agents/`:

```sh
mkdir -p .claude/agents
for dir in app/agents/*/; do
  name=$(basename "$dir")
  cp "$dir/prompt.md" ".claude/agents/$name.md"
done
```

`.claude/agents/` is **gitignored**. Re-run `scripts/setup.sh` after editing any prompt; `scripts/start.sh` should mirror as its first step so opening `claude` always finds fresh agents.

Once mirrored, the user invokes an agent either explicitly ("use the concierge agent") or implicitly (Claude routes to a matching `description` when the user describes a task). The agent then runs with the system prompt and tools declared in its frontmatter.

## Static website

The agents can generate artifacts as a static website. Author content as source files; build to a separate output directory; never hand-edit the build output.

### Why a static site generator

Agents are good at producing structured content (Markdown, frontmatter, data files) but bad at keeping layout, navigation, and styling consistent across hand-written HTML pages. A static site generator (SSG) lets the agents work in their strength — writing content — while the framework guarantees consistency for free.

### Recommended: Astro

Use [Astro](https://astro.build/) as the SSG. Reasons:

- **Content collections** give typed, schema-validated Markdown/MDX content. Agents can author posts/pages with confidence that frontmatter is correct.
- **MDX support** lets content embed components when richer interactivity is needed, without forcing a SPA.
- **Zero-JS by default** — pages ship as plain HTML, so the output is fast and easy to host anywhere.
- **Familiar output**. The build is plain HTML/CSS/JS that agents can inspect and reason about.

Alternatives worth considering: **11ty** (simpler, fewer opinions, no component model), **Hugo** (fastest builds, single binary, but Go templates are awkward for agents to maintain).

### Layout

Place the Astro project under `app/site/`:

- `app/site/src/content/` — authored content (Markdown/MDX). **Agents edit this.**
- `app/site/src/pages/`, `app/site/src/layouts/`, `app/site/src/components/` — page templates and shared layout. Edited by humans (or agents, deliberately) when changing structure.
- `app/site/public/` — static assets copied verbatim into the build (favicon, images).
- `app/site/dist/` — build output. **Never hand-edited; gitignored.**

### Agent guidance

When an agent generates website content, it should:

1. Write to `app/site/src/content/` (or the appropriate collection subdirectory).
2. Conform to the collection's schema — fail loudly on missing/invalid frontmatter rather than silently emitting broken pages.
3. Run the Astro build to verify the page renders before reporting the work as done.
4. Never write to `app/site/dist/`.

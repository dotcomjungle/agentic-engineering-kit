# Claude Code Engine

Playbook for initializing an agentic engineering project that defines system prompts and tools but is interacted with directly in the Claude Code CLI.

## Agent Definitions

Organize the project with an app/agents/ directory

Each agent, such as "designer", "a11y-advocate", "security-officer" or "kermit-the-frog" will have it's own directory. And in that directory will be a prompt.md file.

For example:

`app/agents/a11y-advocate/prompt.md`

And this file would have a system prompt defining the roles, goals, and unique instructions for this agent's behavior.

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

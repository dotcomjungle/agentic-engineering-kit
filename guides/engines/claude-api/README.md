# Claude API Engine

Playbook for initializing an agentic engineering project that uses Anthropic's Claude API and builds a custom runtime, interacted with through the application interface.

> **Status: planning sketch.** This file is an outline for the eventual guide, not the guide itself. The structure below mirrors `guides/engines/agent-sdk/README.md` so a reader can pick between the two by reading the diff. Replace this notice with the real content as sections are written.

## Framing

`claude-api` is the same shape as `agent-sdk` (FastAPI server + SPA client, multi-user, persisted, SSE streaming) — but you own the agent loop. Default to mirroring the `agent-sdk` guide's structure section-for-section; only diverge where building the loop yourself changes the answer. That keeps the two guides comparable.

## Planned sections (and what changes vs. agent-sdk)

1. **When to choose this engine** — pick when SDK loop control isn't enough: custom tool dispatch policies, streaming token-by-token, structured outputs as first-class results, non-standard turn shapes, or wanting the underlying token/usage data the SDK hides.
2. **Assumed constraints** — same stack as agent-sdk; explicit non-goal: this guide doesn't re-derive the server/client guides.
3. **Architecture** — the big swap. No `query()` / `ClaudeSDKClient`. Instead: a hand-rolled async loop around `anthropic.AsyncAnthropic().messages.stream(...)` that handles `tool_use` blocks, dispatches to a tool registry, appends `tool_result` blocks, re-enters the loop until `stop_reason != "tool_use"`. Show the skeleton.
4. **Streaming granularity** — *here* you do get per-token deltas (`content_block_delta`). Worth calling out as a real reason to choose this engine.
5. **Tools** — plain async functions in a registry keyed by name. JSON schema authored by hand (or via Pydantic → schema). No MCP; no SDK `@tool` indirection. Same closure-over-deps pattern for tenancy safety.
6. **Permissions** — pre-dispatch hook in your own loop. Same swap-the-body story.
7. **Subagents** — *you* implement them. Either nested loops in-process or a recursive `run_turn`. Cover the topology + how parent/child runs link via `parent_run_id`.
8. **Safety valves** — turns, wall-clock, cost. Cost you compute yourself from `usage.input_tokens` / `output_tokens` × model pricing — call out that you need a pricing table and how to keep it fresh.
9. **Cancellation** — same cooperative pattern, but you can cancel mid-stream (between deltas), not just between messages. Better than agent-sdk on this axis.
10. **Persistence schema** — nearly identical to agent-sdk. Differences: drop `sdk_session_id` (no SDK session — *you* are the session, replaying message history each turn); add per-call token columns since you have the data.
11. **Resumption** — conversation resume = replay persisted messages into `messages=[...]` on the next call. SSE stream resume identical.
12. **HTTP/SSE API surface** — copy from agent-sdk; add token-delta event(s) to the taxonomy.
13. **Prompt caching** — new section, doesn't exist in agent-sdk guide. With direct API access you control `cache_control` markers explicitly; this is a real cost lever and worth a dedicated section.
14. **Testing** — same split. "Mock the SDK transport" becomes "mock `messages.stream`" — easier, since the API surface is narrower.
15. **Observability** — same OTel/structlog story; add token + cache-hit counters since you have them.
16. **What this guide does not cover** — same disclaimers as agent-sdk (server stack, client UI, auth, individual agents).

## Open questions to resolve before drafting

- **Token streaming default on or off?** It's the engine's headline capability but adds client complexity. Lean toward defaulting on and noting how to suppress.
- **Pricing table location?** `Settings`, a JSON file, or fetched? Cheapest answer: a checked-in dict keyed by model name, updated when models change.
- **Subagent depth — recommend a hard cap?** Probably yes (e.g. 3) to avoid runaway recursion before the cost cap trips.

## References

- https://www.anthropic.com/learn/build-with-claude
- https://platform.claude.com/docs/en/api/overview

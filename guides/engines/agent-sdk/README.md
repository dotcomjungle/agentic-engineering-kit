# Agent SDK Engine

Playbook for an agentic project that uses Anthropic's [Claude Agent SDK for Python](https://github.com/anthropics/claude-agent-sdk-python) as the runtime, hosted inside the project's own application.

## When to choose this engine

Pick `agent-sdk` when:

- The end product is an application end users interact with through a UI (chat or task), not a CLI session.
- You want Anthropic to maintain the agent loop, tool dispatch, session management, and subagent topology rather than building it yourself.
- You're comfortable on the Python server stack in `guides/server/README.md`.

If your product is artifacts authored by Claude Code with no end-user runtime, use `claude-code` instead. If the SDK's loop doesn't fit your control needs, use `claude-api` instead.

## Assumed constraints

This guide commits to a specific shape:

- **Server runtime** — Python in-process inside FastAPI, per `guides/server/README.md`. The SDK is invoked from request handlers; same uvicorn worker handles HTTP and the SDK.
- **Streaming** — Server-Sent Events from the server to the client.
- **Persistence** — every session, message, tool call, and run is persisted to Postgres so runs can be evaluated and audited after the fact.
- **Multi-user** — sessions are scoped per `current_user`, resolved by an auth dependency that's out of scope.
- **Tools** — both built-in and custom MCP tools.
- **Subagents** — first-class via the SDK's `agents` option.
- **Permissions** — auto-approve all tool calls with a hook that's easy to swap for a real policy later.

This guide does **not** re-explain the server stack, the client stack, or how to wire auth. Read `guides/server/README.md` first; this guide extends it.

## Architecture

The SDK runs in-process. One uvicorn worker handles HTTP, SSE, and SDK invocations on the same event loop. Deployment is one image, one process model. Request-scoped resources (DB session, current user, OTel context) are available to tools by closure.

Construct one SDK invocation per **conversation turn**, not one per process. The two SDK entry points are:

- `query(prompt, options)` — stateless async generator. New session each call. Use this for one-shot tasks.
- `ClaudeSDKClient(options)` — stateful client that retains session state across calls. Use this when you want the SDK to manage continuity within a request.

For our shape — a turn arrives over HTTP, runs to completion, persists everything, streams the result, and returns — `query()` with `resume=session_id` from the previous turn is the cleanest fit. The session state we care about lives in Postgres, not in a long-lived Python object.

```python
# src/main.py
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.mcp_servers = build_mcp_servers(settings)   # custom tools
    app.state.agents = build_agent_registry()             # AgentDefinition map
    yield

# src/routers/conversations.py
@router.post("/{conversation_id}/runs")
async def submit_run(
    conversation_id: UUID,
    body: SubmitRunIn,
    user: User = Depends(current_user),
    db: AsyncSession = Depends(get_db),
):
    convo = await load_conversation(db, conversation_id, user.id)
    run = await create_run(db, convo, body, user)
    await db.commit()
    return {"run_id": run.id, "stream_url": f"/api/v1/sessions/{convo.id}/runs/{run.id}/stream"}

@router.get("/{conversation_id}/runs/{run_id}/stream")
async def stream_run(...):
    return EventSourceResponse(run_turn(...), media_type="text/event-stream")
```

`run_turn` is an async generator that opens the SDK call, iterates the message stream, persists each event, and yields an SSE event with the persisted event's id. Use [`sse-starlette`](https://github.com/sysid/sse-starlette)'s `EventSourceResponse` — it handles client-disconnect detection and heartbeat pings.

### Why SSE, not WebSockets

- Traffic is one-directional during a run (server → client). User input arrives on a fresh `POST /runs`.
- SSE rides on plain HTTP — no protocol upgrade, no separate auth path, no special infra at the edge.
- Native reconnection via `Last-Event-ID` is the foundation of stream resumption (see below).
- WebSockets would force us to invent framing and reconnect semantics for no gain.

### Streaming granularity

The Python SDK delivers **whole message objects** (`AssistantMessage`, `ToolResultMessage`, `ResultMessage`), not per-block deltas. The SSE event taxonomy below reflects that: `message` events carry full assistant turns, not character-level chunks.

If you need typewriter-style chat output, you have to bypass the SDK and call the underlying Anthropic streaming API directly — at which point you're effectively on the `claude-api` engine. Don't half-way it. The `agent-sdk` engine is fine for chat where messages appear as units, and it's the natural fit for structured task UIs where the user is waiting on a result, not a typing animation.

## Project layout

Extend the server-guide layout with `agents/` and `mcp_tools/` packages alongside `routers/`, `services/`, `models/`, `schemas/`:

```
app/server/src/
  routers/
    conversations.py     # REST + SSE endpoints
  services/
    agent_runner.py      # opens the SDK call, persists events, emits SSE
    sessions.py          # CRUD on conversations and runs
  agents/
    __init__.py          # registry: name -> AgentDefinition
    <agent_name>/
      __init__.py        # builds AgentDefinition from prompt + tools
      prompt.md          # system prompt, authored as Markdown
  mcp_tools/
    __init__.py          # build_mcp_servers(settings) -> list of MCP servers
    <tool_group>.py      # @tool functions, grouped by domain
  models/
    conversation.py      # Conversation, Message, AgentRun, ToolCall, RunEvent
  schemas/
    conversation.py      # request/response models + SSE event payloads
```

The agent definition (system prompt + tool roster + subagent map) is its own concern; don't bury it in `services/`.

## Defining agents

Agents are `AgentDefinition` instances, registered by name. Load the system prompt from a `prompt.md` file at import time and compute its SHA-256:

```python
# src/agents/researcher/__init__.py
from pathlib import Path
from hashlib import sha256
from claude_agent_sdk import AgentDefinition

PROMPT = (Path(__file__).parent / "prompt.md").read_text()
PROMPT_SHA = sha256(PROMPT.encode()).hexdigest()

definition = AgentDefinition(
    description="Researches a topic and produces a structured brief.",
    prompt=PROMPT,
    tools=["Read", "WebSearch", "WebFetch"],
    model="sonnet",
)
```

Persist `prompt_sha` on every `agent_runs` row. Evaluations group by `prompt_sha`, not by agent name — that's how you tell whether a regression came from a prompt change or something else. Files (not strings) for prompts because they diff cleanly in code review and embed examples without escaping.

Subagents are declared the same way and passed via `ClaudeAgentOptions(agents={"reviewer": reviewer.definition, ...})`. The parent agent delegates by name; the SDK manages the dispatch.

## Custom MCP tools

Define tools with the SDK's `@tool` decorator and register them on an MCP server with `create_sdk_mcp_server`:

```python
# src/mcp_tools/knowledge.py
from claude_agent_sdk import tool, create_sdk_mcp_server

def build_knowledge_server(deps: ToolDeps):
    @tool(
        "lookup_doc",
        "Fetch a document from the knowledge base by id.",
        {"doc_id": str},
    )
    async def lookup_doc(args: dict) -> dict:
        row = await deps.db.get(Document, args["doc_id"])
        if row is None or row.user_id != deps.user.id:
            return {"content": [{"type": "text", "text": "not found"}], "is_error": True}
        return {"content": [{"type": "text", "text": row.body}]}

    return create_sdk_mcp_server(name="knowledge", version="1.0.0", tools=[lookup_doc])
```

Two conventions worth following:

- **Request-scoped deps via closure**, not globals. The router resolves `AsyncSession`, `current_user`, and any other request-scoped resource through `Depends`, hands them to `agent_runner.run(...)`, which calls `build_*_server(deps)` per request. Tool signatures take only LLM-supplied arguments. The model can't spoof tenancy because it can't reach `deps.user`.
- **Return `is_error: True` on tool failures**, don't raise. The SDK surfaces tool errors to the model as tool results so the agent loop can recover. Reserve raised exceptions for actual bugs.

Built-in tools (Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch, etc.) are enabled by listing them in `allowed_tools` on `ClaudeAgentOptions`. Be deliberate about which to expose — `Bash` on a server is broad authority.

## Permissions

Auto-approve every tool call via a `PreToolUse` hook:

```python
from claude_agent_sdk import ClaudeAgentOptions, HookMatcher

async def auto_approve(input_data, tool_use_id, context):
    return {
        "hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": "allow",
        }
    }

options = ClaudeAgentOptions(
    hooks={"PreToolUse": [HookMatcher(hooks=[auto_approve])]},
    ...
)
```

Future restrictions (per-user denylists, rate limits, human-in-the-loop) swap the hook body, not the wiring. Persist every approval decision (the `tool_calls` row records the call regardless) so you can replay historical traffic against a stricter policy before rolling it out.

## Persistence schema

Multi-tenant isolation is by `user_id` on every top-level table, populated from the `current_user` dependency. No row is read or written without a user-scoped query — enforced in `services/`, not `routers/`. Add Postgres RLS later if you want defense in depth.

Tables (shape only — write the migration yourself):

- **`conversations`** — `id`, `user_id`, `title`, `created_at`, `updated_at`, `metadata jsonb`. Index `(user_id, updated_at desc)`.
- **`messages`** — `id`, `conversation_id`, `role` (`user` | `assistant` | `system` | `tool`), `content jsonb`, `created_at`. Index `(conversation_id, created_at)`.
- **`agent_runs`** — one row per SDK invocation. `id`, `conversation_id`, `parent_run_id` (nullable, self-FK for subagents), `agent_name`, `prompt_sha`, `model`, `status`, `started_at`, `ended_at`, `total_cost_usd`, `sdk_session_id` (for resume). Index `(conversation_id, started_at)` and `(parent_run_id)`.
- **`tool_calls`** — `id`, `run_id`, `name`, `input jsonb`, `output jsonb`, `status` (`ok` | `error`), `started_at`, `ended_at`, `duration_ms`. Index `(run_id, started_at)`.
- **`run_events`** — append-only SSE replay log. `id` (monotonic), `run_id`, `seq`, `type`, `payload jsonb`, `created_at`. Index `(run_id, id)`. Retain for a bounded window (e.g. 24h) and prune.

The SDK exposes `total_cost_usd` on the final `ResultMessage` but **not** per-turn input/output token counts. Persist what you have; don't fabricate token columns. If you need per-call token data later, you'll have to instrument by intercepting the underlying API calls.

`run_events` is written on the same code path that emits SSE: persist the row, then yield the event with `id=row.id`. The client never sees an event the database hasn't.

## HTTP / SSE API surface

All routes mounted under `/api/v1`, scoped to `current_user`. Sessions not owned by the caller return `404`, not `403` — don't leak existence.

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/sessions` | Create a conversation. Returns `Session`. |
| `GET` | `/sessions` | List the caller's sessions. Cursor-paginated, newest first. |
| `GET` | `/sessions/{id}` | Fetch one session header. |
| `GET` | `/sessions/{id}/messages` | Full message + event history, cursor-paginated. |
| `PATCH` | `/sessions/{id}` | Rename, archive, edit metadata. |
| `DELETE` | `/sessions/{id}` | Soft-delete. |
| `POST` | `/sessions/{id}/runs` | Submit a turn. Idempotent on `Idempotency-Key`. Returns `{run_id, stream_url}`. |
| `GET` | `/sessions/{id}/runs/{run_id}/stream` | SSE. Supports `Last-Event-ID` for resume. |
| `POST` | `/sessions/{id}/runs/{run_id}/cancel` | Cooperative cancel. |

The split between `POST /runs` and `GET .../stream` lets the client kick off a run, persist `run_id`, and reconnect to the stream after a refresh without resubmitting.

### One endpoint, two input shapes

`POST /runs` accepts a discriminated union on `input.kind`. Same endpoint serves chat and structured-task UIs:

```json
{ "input": { "kind": "chat", "content": "Plot last 30 days of signups." } }

{ "input": { "kind": "task", "task_type": "research_brief",
             "params": { "topic": "EU AI Act", "depth": "deep" } } }
```

Splitting at the HTTP boundary just to mirror UI shapes leaks UI concerns into the contract and forces duplicate plumbing for resume, cancel, and history. The runtime path is identical: render the input into a turn, hand it to the SDK, stream the result. The server validates `params` against a registered Pydantic schema per `task_type`.

### SSE event taxonomy

Each event has a name (`event:`), a JSON payload (`data:`), and a monotonic id (`id:`, the persisted `run_events.id`).

| Event | Payload |
|---|---|
| `run_start` | `{run_id, parent_run_id?, agent}` |
| `message` | `{message_id, role, content, parent_run_id?}` |
| `tool_call_start` | `{tool_call_id, name, input, parent_run_id?}` |
| `tool_call_result` | `{tool_call_id, status, output, duration_ms}` |
| `subagent_start` | `{subagent_run_id, parent_run_id, agent}` |
| `subagent_end` | `{subagent_run_id, status, summary}` |
| `usage` | `{total_cost_usd}` |
| `error` | `{code, message, retriable}` |
| `run_end` | `{run_id, status, total_cost_usd}` |

`message` is whole-message, not per-token, per the streaming-granularity note above. Subagent events appear inline on the parent stream — verify the exact ordering and which message types the SDK emits for subagent activity by running a subagent and observing the message stream; adjust the persistence and replay code to match. Both top-level and subagent events carry `parent_run_id` so the client can build the tree.

## Resumption

Two layers:

- **SDK session resume** — capture `sdk_session_id` from the first turn's `ResultMessage`, persist it on `agent_runs`, and pass `resume=sdk_session_id` in `ClaudeAgentOptions` for the next turn in the same conversation. The SDK rehydrates its session state.
- **SSE stream resume** — a reconnecting client sends `Last-Event-ID: <n>`. The handler queries `run_events` for that run where `id > n`, replays them as SSE, and — if the run is still in flight — attaches to the live tail. Completed runs replay to `run_end` and close.

Resuming a *conversation* (not just a connection) is `GET /sessions/{id}/messages` for history, then `POST /sessions/{id}/runs` for the next turn. The same event objects come back from history that the SSE stream emitted, so the client's reducer is identical for live and replayed events.

## Testing

Extend the unit/integration split from the server guide:

- `tests/unit/agents/` — prompt loading, hash stability, registry wiring, schema round-trips.
- `tests/unit/mcp_tools/` — each tool tested as a plain async function with a transactional `AsyncSession` fixture and a fake deps object. No SDK involvement.
- `tests/integration/agents/` — full agent runs against a **mocked model**. Stub the SDK transport to replay scripted assistant messages (text + tool_use blocks + stop reasons). Assert on what was persisted: messages, tool calls, run status, recorded events. Verify the exact mock seam against the SDK's current internals.
- `tests/integration/agents/recorded/` — a small set of runs against the real model, gated behind `RUN_LIVE=1`, run nightly. These catch prompt regressions; the mocked tests catch wiring regressions.

Avoid running every CI build against a real model. Non-determinism makes failures noisy and erodes trust in the suite.

## Observability

Open an OTel span `agent.run` for each SDK invocation with attributes `agent.name`, `agent.run_id`, `user.id`, `prompt.sha`, `model.name`. Each tool call is a child span `agent.tool` with `tool.name`, `tool.duration_ms`, `tool.status`. Bind the same fields into structlog at the start of the request via `structlog.contextvars.bind_contextvars(...)` so every log line in the run carries them.

`total_cost_usd` rolls up onto the `agent_runs` row at completion. That row plus its `tool_calls` and `run_events` children is the unit of evaluation; the OTel trace is the time-ordered view of the same data. Together they make any failed or surprising run reproducible without re-running the model.

## What this guide does not cover

- **Server stack** (FastAPI, SQLAlchemy, Postgres, uv, lint, test, deploy) — `guides/server/README.md`.
- **Client UI** (React/TS, SSE consumption, rendering tool-call and subagent events) — `guides/client/README.md`.
- **Auth** — bring your own. Wire it in as a FastAPI dependency that resolves `current_user`. Everything in this guide assumes that exists.
- **Individual agents** (system prompts, tool rosters, subagent topology for *your* product) — that's per-project content under `app/server/src/agents/`, not part of the engine playbook.

## References

- SDK source — https://github.com/anthropics/claude-agent-sdk-python
- SDK overview — https://code.claude.com/docs/en/agent-sdk/overview
- Quickstart — https://code.claude.com/docs/en/agent-sdk/quickstart
- Python reference — https://code.claude.com/docs/en/agent-sdk/python
- Custom tools — https://code.claude.com/docs/en/agent-sdk/custom-tools
- Hooks — https://code.claude.com/docs/en/agent-sdk/hooks
- Sessions — https://code.claude.com/docs/en/agent-sdk/sessions
- Subagents — https://code.claude.com/docs/en/agent-sdk/subagents

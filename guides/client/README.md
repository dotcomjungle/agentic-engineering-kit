# Client Stack Guide

Recommended TypeScript/React stack for the frontend of a web project built with this kit.

**This guide only applies if you picked the `agent-sdk` or `claude-api` engine.** The `claude-code` engine doesn't ship a SPA — agents publish to a static Astro site under `app/site/` and there's nothing for a client of this shape to do.

For projects that *do* have a client, the client is the **interaction surface**. It renders the UI, drives user input, consumes the server's REST + SSE contract, and keeps a coherent local view of server state. Domain logic, persistence, and the agent runtime belong to the server.

## At a glance

| Concern | Choice |
|---|---|
| Build tool | Vite |
| Language | TypeScript (strict) |
| UI framework | React 18+ |
| Routing | TanStack Router (file-based) |
| Server-state cache | TanStack Query |
| API transport | `fetch` + thin `ApiError` wrapper |
| Streaming | SSE via `fetch` + `ReadableStream` |
| Cross-cutting UI state | React Context |
| Styling | Tailwind v4 |
| Testing | Vitest + Testing Library (co-located) |
| Lint + format | ESLint flat config + `typescript-eslint`, Prettier |
| Path alias | `@/*` → `src/*` |
| Auth | Out of scope — bring your own |

## What the client owns

- UI rendering and user interaction
- Routing and route-scoped data loading
- Client-side cache of server state (via TanStack Query)
- SSE consumption and run-state reduction
- Form input collection and submission
- Display of errors raised by the server, in human-readable form

## What the client does *not* own

- **Domain logic / validation** — the server is authoritative. The client does shape-only typing and surfaces server errors; it doesn't replicate validation rules.
- **Agent runtime** — see `guides/engines/`. The client only consumes the run stream.
- **Persistence** — no IndexedDB, no `localStorage` for domain data. Server is the system of record. (Auth tokens / theme preference are fine.)
- **Auth** — bring your own provider. Wire token resolution into a single place (`lib/api.ts`).

## Stack

### Vite + React 18+

[Vite](https://vitejs.dev/) for dev server, build, and HMR. React 18 (or 19, if upstream peers cooperate) for the view layer. Plain React — no Next.js, no Remix. The server is the FastAPI backend in `guides/server/`; SSR adds nothing here, and pulling in a Node-shaped meta-framework introduces a second runtime to manage for no win.

### TypeScript (strict)

Strict mode (`strict: true`, `noUnusedLocals`, `noUnusedParameters`, `noFallthroughCasesInSwitch`). Treat type errors as build failures. Don't ship `any`; use `unknown` and narrow.

### TanStack Router (file-based)

[TanStack Router](https://tanstack.com/router) for routing. File-based routes under `src/routes/`; the `@tanstack/router-vite-plugin` codegens `src/routeTree.gen.ts` (gitignored — never edit by hand). Reasons:

- **Type-safe params and search** — route params are typed end-to-end. No `useParams<{ id: string }>()` casts.
- **First-class data loading** — `loader`s integrate with TanStack Query so the same query feeds the loader and any `useQuery` inside the component.
- **File-based without a meta-framework** — get the convention without inheriting Next/Remix's runtime model.

Alternatives: `react-router-dom` v7 if your team prefers familiarity over type safety. Skip routing entirely only for genuine single-screen prototypes; you'll regret it the second time you need to deep-link.

### TanStack Query

[TanStack Query](https://tanstack.com/query) is the single source of truth for **server state**: lists, sessions, messages, run history. Don't mirror server data into a global store.

- One `queryClient` constructed in `main.tsx` and provided at the root.
- Query keys built through a factory in `lib/queries.ts` (e.g. `qk.sessions.list({ cursor })`, `qk.sessions.detail(id)`). Stringly-typed keys spread across components are how invalidation goes wrong.
- Mutations invalidate the relevant query keys on success. Be specific — `invalidateQueries({ queryKey: qk.sessions.detail(id) })`, not `queryClient.invalidateQueries()` blanket.
- Sensible defaults: `staleTime` of a few seconds for lists, longer for stable detail fetches; `retry: 1` for queries; `retry: false` for mutations.

### `fetch` + `ApiError`

No axios. A thin `apiFetch<T>(path, init)` helper in `lib/api.ts`:

- Prepends the base URL (in dev, the Vite proxy handles `/api` → `localhost:8000`; in prod, same-origin or env var).
- Attaches auth headers from a single resolver.
- Parses JSON, returns typed result.
- Throws `ApiError` on non-2xx with `{ status, code, message, details }` shaped from the server's error envelope.

Why not axios: we already need `fetch` for SSE (axios's browser build doesn't stream `ReadableStream` well), so splitting the codebase into "axios for normal calls, fetch for streams" gives two error shapes for no benefit. TanStack Query handles retries and caching, which is what most teams reach for axios interceptors to do.

### SSE via `fetch` + `ReadableStream`

`EventSource` is too limited (no custom headers means no bearer tokens, no body on the open). Use `fetch` with `Accept: text/event-stream`, read the body as a `ReadableStream`, and parse `event:` / `data:` / `id:` lines yourself. A small parser in `lib/sse.ts`:

- Accepts a typed event union matching the server's event taxonomy (see `guides/engines/agent-sdk/`).
- Exposes a `consumeRun(url, { onEvent, signal, lastEventId })` function returning a promise that resolves on `run_end` or rejects on terminal error.
- Tracks the last seen `id:` and reconnects with `Last-Event-ID` on transient failure.
- Cleans up via an `AbortController` passed in by the caller.

The reducer that turns the event stream into a `Run` view-model lives next to it (`lib/run-reducer.ts`). The component subscribing to a run holds a `useReducer`-backed state machine; the reducer is pure and unit-tested.

### Cross-cutting UI state — React Context

For genuinely cross-cutting *UI* state (current theme, an open command palette, a sidebar collapse) use React Context. Don't reach for Zustand/Redux/Jotai — none have justified themselves in our sample apps, and TanStack Query already covers the case people usually want a global store for.

### Tailwind v4

[Tailwind](https://tailwindcss.com/) v4 via `@tailwindcss/postcss` and `@tailwindcss/vite`. Utility-first; no `@apply` for component reuse — extract a React component instead. Design tokens (colors, spacing scale) configured in `tailwind.config.{ts,js}`; semantic colors expressed as CSS custom properties so dark mode is one variable swap.

No component library (no shadcn, MUI, Mantine) by default. Build the small set of shared primitives you need under `components/ui/`. Adopt shadcn or a component library when you have evidence you need one — usually that's once you have more than three or four similar form layouts.

### Vite proxy

In `vite.config.ts`, proxy `/api` to `http://localhost:8000` (the FastAPI dev server). This means the client uses the same path in dev and prod (`fetch('/api/v1/sessions')`), no env vars to swap, no CORS to configure for local development. In production the client and server are typically same-origin behind a reverse proxy; if not, set `VITE_API_BASE_URL` and read it once in `lib/api.ts`.

### Testing

[Vitest](https://vitest.dev/) + [Testing Library](https://testing-library.com/) + `jsdom`. Tests **co-located** next to the file they cover (`Foo.tsx` ↔ `Foo.test.tsx`). Reasons:

- Co-location keeps tests honest — they move with the code; broken-import rot doesn't accumulate in a parallel `__tests__/` tree.
- Vitest reuses the Vite config, so there's one build pipeline for app and tests.

What to test:

- **Pure helpers** (`lib/sse.ts` parser, `lib/run-reducer.ts`, query-key factory) — full coverage. These are the load-bearing bits.
- **Components** — Testing Library queries via accessible roles, not implementation details. Mock at the API boundary (TanStack Query's `QueryClient` with a stubbed `apiFetch`), not at the component level.
- **End-to-end** — Playwright is *not* default. Add it when you have a smoke-test budget and a stable enough UI to make the maintenance worthwhile.

### ESLint + Prettier

Flat config (`eslint.config.js`) with `typescript-eslint`, `eslint-plugin-react-hooks`, `eslint-plugin-react-refresh`. Prettier for formatting; ESLint defers to Prettier on style. Run both in pre-commit and CI.

### Path alias

One alias: `@/*` → `src/*`, configured in both `tsconfig.json` (`paths`) and `vite.config.ts` (`resolve.alias`). Don't proliferate aliases — one is enough to avoid `../../../` chains, more is bikeshedding.

### Auth (out of scope)

Resolve the auth token in one place (`lib/api.ts`'s header resolver). Wire your provider (Clerk, Auth0, custom JWT, session cookie) into that resolver and a route guard at `__root.tsx`. Don't sprinkle token reads across components.

## Project layout

```
app/client/
  package.json
  tsconfig.json
  tsconfig.app.json
  tsconfig.node.json
  vite.config.ts
  eslint.config.js
  tailwind.config.ts
  postcss.config.js
  index.html
  .env.example
  public/
  src/
    main.tsx              # entry — QueryClient + RouterProvider
    routeTree.gen.ts      # generated; gitignored
    styles/
      index.css           # Tailwind directives + custom properties
    routes/               # file-based routes
      __root.tsx          # layout, providers, error boundary
      index.tsx
      sessions.tsx        # list
      sessions.$sessionId.tsx  # detail
    components/
      ui/                 # shared primitives (Button, Input, Dialog)
      run/                # run-stream display components
    lib/
      api.ts              # apiFetch<T> + ApiError + auth header resolver
      sse.ts              # SSE consumer
      run-reducer.ts      # event → Run state-machine
      queries.ts          # query-key factory + useXxx hooks + mutations
      types.ts            # shared types (event union, domain shapes)
    contexts/             # only for genuinely cross-cutting UI state
    hooks/                # shared React hooks
```

## Patterns

### Route + query co-loading

A route's `loader` calls `queryClient.ensureQueryData(qk.sessions.detail(id))`. The component reads the same data with `useQuery(qk.sessions.detail(id))`. The cache is shared; navigation kicks off the fetch immediately and the component never sees a "loading" flash on hover-prefetch.

### Submitting a turn and consuming its stream

1. Mutation: `POST /api/v1/sessions/{id}/runs` returns `{ run_id, stream_url }`.
2. On success, navigate to the route that owns that run (or stay and open a stream).
3. The route mounts a hook (`useRunStream(sessionId, runId)`) that calls `consumeRun(url, ...)` with an `AbortController`. The reducer accumulates events into a `Run` view-model.
4. On unmount, abort.
5. On reconnect, call `consumeRun` again with the last seen event id; the server replays missed events and continues.

### Error handling

- Server errors arrive as `ApiError`. Surface them in a top-level toast or inline near the action that triggered the call. Don't swallow.
- TanStack Query's `error` field is the right place to read them in components.
- An `error` SSE event ends the run; the reducer transitions to `error` state and displays `message`. Don't auto-retry stream errors silently.

### Forms

No `react-hook-form` by default. Controlled inputs in small forms, `useReducer` for anything multi-field. Validate at the server boundary — reflect server errors back into the form. Add `react-hook-form` if/when you have a form complex enough that `useReducer` becomes painful (rule of thumb: more than five fields with cross-field validation).

## Alternatives worth considering

- **Next.js (App Router)** — pick this if you need SSR for SEO or want server actions. Adds a Node runtime and a second deploy target for our shape (FastAPI is already the server).
- **react-router v7** — pick instead of TanStack Router if your team values familiarity over type safety. You give up the typed loaders/search-params benefit.
- **shadcn/ui** — drop in when you've outgrown hand-rolled primitives. Pairs well with Tailwind.
- **Zustand** — pick if you find yourself genuinely needing a global store that isn't server state and isn't a single Context. Most apps don't.
- **react-hook-form + zod** — pick when forms get complex. Validate with zod against shapes the server provides; don't duplicate server-side rules.
- **Playwright** — add when you have stable critical paths worth smoke-testing end-to-end.
- **MSW** — for component tests that exercise real fetch paths against mocked endpoints. Worth it for SSE testing in particular.

## References

- Vite — https://vitejs.dev/
- TanStack Router — https://tanstack.com/router
- TanStack Query — https://tanstack.com/query
- Tailwind — https://tailwindcss.com/
- Vitest — https://vitest.dev/
- Testing Library — https://testing-library.com/
- typescript-eslint — https://typescript-eslint.io/

# Frontend AGENTS.md

Stack: Next.js 16, React 19, TypeScript, Tailwind 4, Shadcn/UI + MagicUI.
See also: `CLAUDE.md` (89 lines, practical guidance).

## Component Domains

```
src/components/
├── ai-elements/     # 27 files — Agent UX: message, reasoning, chain-of-thought, plan, context, artifacts
│                    #   prompt-input.tsx (37KB) = main input; context.tsx = token tracking
├── workspace/       # 50+ files — Chat, artifacts, settings, navigation, agents
│   ├── chats/       #   chat-box.tsx = resizable dual-panel (chat + artifacts)
│   ├── artifacts/   #   context.tsx = artifact selection state (React Context)
│   ├── settings/    #   7 settings pages (appearance, memory, skills, tools, etc.)
│   ├── messages/    #   message-list, markdown-content, subtask-card
│   └── agents/      #   agent-card, agent-gallery
├── ui/              # 45 files — Shadcn/UI (Radix primitives) + MagicUI animated components
└── landing/         # 6 files — Marketing pages (hero, sections, footer)
```

## State Management (No Redux)

| Layer | Pattern | Where |
|-------|---------|-------|
| Server state | React Query (TanStack) | `src/core/*/hooks.ts` — useModels, useThreads, useSkills, etc. |
| UI state | React Context | Scattered contexts: ArtifactsContext, TasksContext, I18nContext |
| Persistence | localStorage | `src/core/settings/local.ts` — appearance, memory, notification prefs |
| Streaming | LangGraph SDK | `src/core/threads/hooks.ts` — useThreadStream() (SSE) |
| API client | Singleton | `src/core/api/api-client.ts` — wraps @langchain/langgraph-sdk |

## Routing (App Router)

```
/                                          → Landing page
/workspace/chats/[thread_id]               → Main chat (THE primary page)
/workspace/agents                          → Agent gallery
/workspace/agents/[agent_name]/chats/[id]  → Agent-specific chat
/workspace/agents/new                      → Create agent
```

## Where to Make Changes

| Task | Primary file(s) | Notes |
|------|-----------------|-------|
| New AI UI element | `components/ai-elements/` | Compound component + Context pattern |
| New workspace page | `src/app/workspace/` | App Router; use useThread() for state |
| New settings page | `workspace/settings/` + `core/settings/local.ts` | Add to localStorage defaults |
| Server data hook | `src/core/{domain}/hooks.ts` | React Query pattern |
| API calls | `src/core/api/api-client.ts` | Singleton LangGraph client |
| New UI primitive | `components/ui/` | Follow Shadcn/UI conventions |
| Styling/animation | Tailwind classes; MagicUI for effects | GSAP + Motion for advanced |

## Data Flow: Chat → Agent → Response

```
prompt-input.tsx → useThreadStream() → LangGraph SDK client.runs.stream()
  → SSE events (messages-tuple, values) → Message Context → message-list.tsx
  → artifacts extracted → ArtifactsContext → artifact-file-detail.tsx
```

## Key Dependencies

- `@langchain/langgraph-sdk` — Agent streaming (SSE)
- `@tanstack/react-query` — Server state cache
- `@xyflow/react` — Graph/flow visualization (agent planning)
- `shiki` — Syntax highlighting
- `streamdown` — Stream markdown rendering
- `tokenlens` — Token usage/cost calculation
- `react-resizable-panels` — Dual-panel chat layout

## Gotchas

- `BETTER_AUTH_SECRET` required for production build (or set `SKIP_ENV_VALIDATION=1`)
- `pnpm check` is broken — use `pnpm lint && pnpm typecheck` separately
- No Redux/Zustand — all state via Context + React Query + localStorage
- Env validation in `src/env.js` — can block builds silently
- Nginx proxies API calls in dev — frontend `.env` overrides are optional

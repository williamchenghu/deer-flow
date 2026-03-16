# AGENTS.md — DeerFlow Root

## What Is This

Open-source super agent harness by ByteDance. LangGraph + FastAPI backend, Next.js frontend.
Agents execute tasks in sandboxed environments with memory, skills, and sub-agent delegation.

## Architecture (Request Flow)

```
Browser :2026 → Nginx
  ├─ /api/langgraph/* → LangGraph Server :2024 (agent streaming, threads)
  ├─ /api/*           → Gateway API :8001 (models, MCP, skills, memory, uploads, artifacts)
  └─ /*               → Frontend :3000 (Next.js)
```

## Repository Map

```
├── backend/
│   ├── packages/harness/deerflow/   # Publishable harness (import deerflow.*)
│   │   ├── agents/                  # Lead agent factory, middlewares (9), memory
│   │   ├── sandbox/                 # Sandbox abstraction + local provider
│   │   ├── tools/                   # Built-in tools (present_files, ask_clarification, etc.)
│   │   ├── subagents/               # Sub-agent executor, registry, builtins
│   │   ├── mcp/                     # MCP protocol integration
│   │   ├── skills/                  # Skill discovery + loading
│   │   ├── models/                  # Model factory (langchain providers)
│   │   ├── config/                  # Config system (config.yaml parsing)
│   │   ├── community/               # Community tools (tavily, jina, firecrawl, aio_sandbox)
│   │   ├── reflection/              # Dynamic module loading
│   │   ├── client.py                # Embedded Python client (DeerFlowClient)
│   │   └── utils/                   # Shared utilities
│   ├── app/                         # App layer (import app.*)
│   │   ├── gateway/                 # FastAPI gateway (routers, lifecycle)
│   │   └── channels/               # IM bridge (Feishu, Slack, Telegram)
│   ├── tests/                       # 277+ pytest tests
│   └── docs/                        # Backend documentation
├── frontend/
│   ├── src/app/                     # Next.js 16 App Router pages
│   ├── src/components/              # ai-elements/, workspace/, ui/, landing/
│   ├── src/core/                    # Business logic (api, threads, settings, memory, etc.)
│   └── src/styles/                  # Global styles
├── skills/public/                   # 17 built-in skills (Markdown-driven workflows)
├── docker/                          # Compose files, nginx configs, K8s sandbox provisioner
├── scripts/                         # 11 orchestration scripts
├── config.yaml                      # Main config (models, tools, sandbox, channels)
└── extensions_config.json           # MCP servers + skill states
```

## Critical Boundary

**Harness (`deerflow.*`) NEVER imports App (`app.*`)**. Enforced by `test_harness_boundary.py` in CI.
Harness = publishable framework. App = deployment-specific gateway + channels.

## Commands

```bash
# Setup
make config           # Bootstrap config.yaml from template
make check            # Verify prerequisites (Node 22+, pnpm, uv, nginx)
make install          # Install backend + frontend dependencies

# Development
make dev              # All services with hot-reload (:2026)
make stop             # Stop all services

# Docker
make docker-init      # Pull sandbox image (once)
make docker-start     # Dev stack with hot-reload
make docker-stop      # Stop docker stack
make up               # Production build + deploy
make down             # Production teardown

# Quality
cd backend && make lint && make test    # ruff + pytest (277 tests)
cd frontend && pnpm lint && pnpm typecheck
```

## Config Files

| File | What | Where |
|------|------|-------|
| `config.yaml` | Models, tools, sandbox mode, channels, memory | Project root |
| `extensions_config.json` | MCP servers + skill enable/disable states | Project root |
| `.env` | API keys (TAVILY, OPENAI, IM tokens) | Project root (gitignored) |
| `frontend/.env` | Optional backend URL overrides | frontend/ (gitignored) |

## Conventions (Deviations Only)

- Backend line length: **240 chars** (ruff)
- Backend formatter: **ruff** (not black)
- Frontend: Prettier defaults + ESLint
- Config values starting with `$` resolve as env vars (e.g., `api_key: $OPENAI_API_KEY`)
- Skills: Markdown SKILL.md files, progressively loaded into system prompt
- Sandbox paths: `/mnt/user-data/{workspace,uploads,outputs}`, `/mnt/skills/{public,custom}`

## Anti-Patterns

- Importing `app.*` from `deerflow.*` — breaks harness boundary, CI fails
- Hardcoding API keys in config.yaml — use `$ENV_VAR` references
- Skipping tests — TDD mandatory, every feature/fix needs tests
- Modifying CLAUDE.md/README.md without updating after code changes

## Where to Look

| Want to... | Look at |
|------------|---------|
| Understand backend architecture | `backend/CLAUDE.md` (510 lines, comprehensive) |
| Understand frontend architecture | `frontend/CLAUDE.md` (89 lines) |
| Add a new tool | `backend/packages/harness/deerflow/tools/` + register in config.yaml |
| Add a new skill | `skills/public/your-skill/SKILL.md` |
| Add a new MCP server | `extensions_config.json` |
| Add an IM channel | `backend/app/channels/` (base.py → implement → register in service.py) |
| Change agent behavior | `backend/packages/harness/deerflow/agents/lead_agent/` |
| Add middleware | `backend/packages/harness/deerflow/agents/middlewares/` |
| Add a frontend page | `frontend/src/app/workspace/` (App Router) |
| Configure models | `config.yaml` → `models` section |
| Configure sandbox mode | `config.yaml` → `sandbox` section (local/docker/k8s) |

## Child AGENTS.md Files

- `backend/AGENTS.md` — Backend navigation guide (complements CLAUDE.md)
- `frontend/AGENTS.md` — Frontend navigation guide (complements CLAUDE.md)

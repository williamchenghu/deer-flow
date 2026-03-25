# Backend AGENTS.md

Full architecture reference: `CLAUDE.md` (510 lines). This file complements — not duplicates — it.

## Package Split (Critical)

```
backend/
├── packages/harness/deerflow/   # Publishable harness — import deerflow.*
│   ├── agents/lead_agent/       # Agent factory + system prompt
│   ├── agents/middlewares/      # 9 middlewares (strict order matters)
│   ├── agents/memory/           # LLM-powered memory extraction
│   ├── sandbox/                 # Abstract sandbox + local provider
│   ├── tools/                   # Built-in tools (present_files, ask_clarification, view_image, task)
│   ├── subagents/               # Executor, registry, builtins (general-purpose, bash)
│   ├── mcp/                     # MCP protocol integration (stdio, SSE, HTTP)
│   ├── skills/                  # Skill discovery + progressive loading
│   ├── models/                  # Model factory (langchain providers, $ENV_VAR resolution)
│   ├── config/                  # config.yaml parser + paths
│   ├── community/               # Tavily, Jina, Firecrawl, AioSandbox (Docker provider)
│   ├── reflection/              # Dynamic module loading
│   ├── client.py                # Embedded DeerFlowClient (in-process, no HTTP)
│   └── utils/
├── app/                         # Deployment layer — import app.*
│   ├── gateway/                 # FastAPI routers (models, mcp, skills, memory, uploads, artifacts, channels)
│   └── channels/                # IM bridge: Feishu (streaming cards), Slack, Telegram
└── tests/                       # 277+ tests, pytest
```

**HARD RULE**: `deerflow.*` NEVER imports `app.*`. Enforced by `test_harness_boundary.py`.

## Where to Make Changes

| Task | Primary file(s) | Notes |
|------|-----------------|-------|
| New tool | `deerflow/tools/builtins/` | Register in `config.yaml` → `tools` |
| New middleware | `deerflow/agents/middlewares/` | Order matters — see CLAUDE.md §Middleware Chain |
| Change system prompt | `deerflow/agents/lead_agent/prompts.py` | Skills inject here dynamically |
| New sub-agent type | `deerflow/subagents/builtins/` | Register in `deerflow/subagents/registry.py` |
| New sandbox provider | `deerflow/community/` or `deerflow/sandbox/` | Implement `SandboxProvider` interface |
| New MCP transport | `deerflow/mcp/` | stdio/SSE/HTTP already supported |
| New IM channel | `app/channels/` | Inherit `Channel`, register in `service.py` |
| Gateway endpoint | `app/gateway/routers/` | FastAPI router |
| Memory behavior | `deerflow/agents/memory/` | LLM extraction + JSON storage |

## Channels Architecture (Quick Ref)

- **MessageBus**: async pub/sub decoupling channels from dispatcher
- **ChannelManager** (703 lines): inbound → LangGraph → outbound. Max 5 concurrent.
- **Feishu**: streams via `runs.stream()`, updates cards in-place
- **Slack/Telegram**: batch via `runs.wait()`, single final message
- **ChannelStore**: JSON-backed `{channel}:{chat_id}[:{topic_id}]` → thread_id mapping
- New channel: implement `Channel(ABC)` → `start()`, `stop()`, `send()` → register in `_CHANNEL_REGISTRY`

## Testing

```bash
cd backend && make test      # 277+ tests, ~77s
cd backend && make lint      # ruff check (240 char line limit)
```

## Gotchas

- Middleware order is strict — `ClarificationMiddleware` MUST be last
- `config.yaml` values starting with `$` auto-resolve as env vars at runtime
- Sub-agents: max 3 concurrent, 15-min timeout
- Memory uses debounced updates to minimize LLM calls
- Skill loading is progressive (loaded into prompt only when task needs them)
- Missing model provider → actionable error with `uv add` guidance

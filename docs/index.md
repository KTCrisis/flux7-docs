# flux7 docs

Governance infrastructure for AI agents. Four projects, one system.

```
flux7-console (L2 — human UI)                 ← visibility + intervention
  │              │
  │ reads via    │ reads via
  │ HTTP API     │ Python SDK
  │              │
  ▼              ▼
flux7-mesh      flux7-memory                   ← runtime + memory
(L0 — policy)   (memory substrate)
  │   ▲
  │   │ poll + resolve
  │   │
  │  sup7 (L1 — supervisor)                    ← automated evaluation
  │
  └──► tools (MCP, OpenAPI, CLI)
```

**[flux7-mesh](mesh7/index.md)** enforces what agents can do. Policy engine, approval queue, rate limiting, tracing. One Go binary, one YAML config.

**[flux7-memory](mem7/index.md)** remembers what happened. Persistent, searchable, governed memory. Hybrid search (BM25 + dense cosine + LLM reranking). Single Go binary.

**[flux7-supervisor](sup7/index.md)** evaluates what's ambiguous. Polls pending approvals, evaluates via rules + pluggable LLM (Ollama, Anthropic, Claude Code), auto-resolves the routine. Python agent.

**[flux7-console](console/index.md)** shows it all and lets humans intervene. Dashboard, approval UI, governance scoring, audit trail. Next.js + FastAPI.

## Progressive adoption

| Step | What you add | What you get |
|------|-------------|-------------|
| Day 1 | flux7-mesh | Policies, tracing, approval queue |
| Day 7 | + Python SDK | Governance in any code (`POST /decide`) |
| Day 30 | + flux7-memory | Persistent memory, decision history, auto-approve |
| Day 45 | + flux7-supervisor | Automated L1 evaluation, reduced approval fatigue |
| Day 60 | + flux7-console | Dashboard, team approval UI, governance scoring |

Each step is independently valuable. Start with flux7-mesh — five minutes from install to first governed tool call.

## Quick links

- [Getting Started with flux7-mesh](mesh7/getting-started.md)
- [Supervisor Configuration](sup7/configuration.md)
- [System Design (all 4 projects)](console/system-design.md)
- [Deployment Modes](mesh7/deployment-modes.md)

## Source

- [flux7-mesh](https://github.com/KTCrisis/flux7-mesh) — MIT
- [flux7-memory](https://github.com/KTCrisis/flux7-memory) — MIT
- [flux7-supervisor](https://github.com/KTCrisis/flux7-supervisor) — MIT
- [flux7-console](https://github.com/KTCrisis/flux7-console)

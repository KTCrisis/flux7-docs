# flux7 docs

Governance infrastructure for AI agents. Three projects, one system.

```
flux7-console (management plane)              ← visibility + control
  │              │
  │ reads via    │ reads via
  │ HTTP API     │ Python SDK
  │              │
  ▼              ▼
flux7-mesh      flux7-memory                   ← runtime + memory
(sidecar)       (memory substrate)
  │
  └──► tools (MCP, OpenAPI, CLI)
```

**[flux7-mesh](flux7-mesh/index.md)** enforces what agents can do. Policy engine, approval queue, rate limiting, tracing. One Go binary, one YAML config.

**[flux7-memory](flux7-memory/index.md)** remembers what happened. Persistent, searchable, governed memory. Hybrid search (BM25 + dense cosine + LLM reranking). Single Go binary.

**[flux7-console](flux7-console/index.md)** shows it all and lets humans intervene. Dashboard, approval UI, governance scoring, audit trail. Next.js + FastAPI.

## Progressive adoption

| Step | What you add | What you get |
|------|-------------|-------------|
| Day 1 | flux7-mesh | Policies, tracing, approval queue |
| Day 30 | + flux7-memory | Persistent memory, decision history, auto-approve |
| Day 60 | + flux7-console | Dashboard, team approval UI, governance scoring |

Each step is independently valuable. Start with flux7-mesh — five minutes from install to first governed tool call.

## Quick links

- [Getting Started with flux7-mesh](flux7-mesh/getting-started.md)
- [System Design (all 3 projects)](flux7-console/system-design.md)
- [Deployment Modes](flux7-mesh/deployment-modes.md)

## Source

- [flux7-mesh](https://github.com/KTCrisis/flux7-mesh) — MIT
- [flux7-memory](https://github.com/KTCrisis/flux7-memory) — MIT
- [flux7-console](https://github.com/KTCrisis/flux7-console)

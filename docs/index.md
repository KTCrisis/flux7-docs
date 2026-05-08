# flux7 docs

Governance infrastructure for AI agents. Three projects, one system.

```
agent7 (management plane)              ← visibility + control
  │              │
  │ reads via    │ reads via
  │ HTTP API     │ Python SDK
  │              │
  ▼              ▼
agent-mesh      mem7                   ← runtime + memory
(sidecar)       (memory substrate)
  │
  └──► tools (MCP, OpenAPI, CLI)
```

**[agent-mesh](agent-mesh/index.md)** enforces what agents can do. Policy engine, approval queue, rate limiting, tracing. One Go binary, one YAML config.

**[mem7](mem7/index.md)** remembers what happened. Persistent, searchable, governed memory. Hybrid search (BM25 + dense cosine + LLM reranking). Single Go binary.

**[agent7](agent7/index.md)** shows it all and lets humans intervene. Dashboard, approval UI, governance scoring, audit trail. Next.js + FastAPI.

## Progressive adoption

| Step | What you add | What you get |
|------|-------------|-------------|
| Day 1 | agent-mesh | Policies, tracing, approval queue |
| Day 30 | + mem7 | Persistent memory, decision history, auto-approve |
| Day 60 | + agent7 | Dashboard, team approval UI, governance scoring |

Each step is independently valuable. Start with agent-mesh — five minutes from install to first governed tool call.

## Quick links

- [Getting Started with agent-mesh](agent-mesh/getting-started.md)
- [System Design (all 3 projects)](agent7/system-design.md)
- [Deployment Modes](agent-mesh/deployment-modes.md)

## Source

- [agent-mesh](https://github.com/KTCrisis/agent-mesh) — MIT
- [mem7](https://github.com/KTCrisis/mem7) — MIT
- [agent7](https://github.com/KTCrisis/agent7)

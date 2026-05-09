# flux7-memory

Persistent, searchable, governed memory for AI agents. Single Go binary, zero dependencies.

## The problem

Agents work for one session, then forget everything. You add memory — a vector store, maybe Mem0 or Zep — and single-agent workflows improve. Then you scale to multiple agents, and new problems appear :

- **Agent A approved something last week. Agent B doesn't know.** Human decisions aren't stored as queryable facts.
- **Three agents write to the same memory. Who wrote what ?** No provenance, no access control at the fact level.
- **Your agent uses a memory from 6 months ago.** No staleness signal, no lifecycle management.
- **A client asks for an audit trail.** You have logs somewhere. They're not queryable.

These aren't retrieval problems. They're governance problems.

## Quick start

```bash
go install github.com/KTCrisis/flux7-memory/cmd/mem7@latest

# Daemon mode (shared across clients)
MEM7_TOKEN=mem7_secret123 mem7 serve --listen :9070

# Or stdio mode (MCP client spawns the binary)
mem7
```

```python
from mem7 import Mem7

m = Mem7("http://localhost:9070", token="mem7_secret123")
m.store("deploy.decision", "approved by ops lead",
        tags=["decision"], agent="supervisor")

for mem in m.context("deployment approval", limit=5):
    print(f"{mem.key}: {mem.value}")
```

## Features

- **7 MCP tools** — `store`, `recall`, `search`, `context`, `get`, `list`, `forget`
- **Hybrid search** — BM25 + dense cosine + LLM reranking (71% LoCoMo benchmark)
- **Markdown source of truth** — SQLite index is rebuildable via `mem7 rescan`
- **Three transports** — MCP stdio, HTTP JSON-RPC, MCP SSE (daemon mode)
- **Auto-proxy** — stdio mode detects a running daemon and proxies transparently
- **Provider-agnostic** — works with Ollama, OpenAI, or any compatible embedding API
- **Python SDK** — `pip install flux7-memory` — structured `Memory` objects, not raw text

## How it fits

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   Agent A    │   │   Agent B    │   │  Supervisor   │
│  (research)  │   │  (execution) │   │ (human-in-    │
│              │   │              │   │  the-loop)    │
└──────┬───────┘   └──────┬───────┘   └──────┬────────┘
       │ store/search      │ store/search     │ store policies
       │ tags=["research"] │ tags=["exec"]    │ tags=["decision"]
       └──────────┬────────┴──────────────────┘
                  │
           ┌──────▼──────┐
           │ flux7-memory │  ← one binary, shared memory
           │              │    with agent-scoped tags
           └──────────────┘
```

**Agent memory** — each agent reads and writes observations, scoped by tags.

**Supervisor memory** — cross-agent view. Policies and human approvals stored as first-class facts.

**Audit** — every fact carries who wrote it, when, with which tags. Queryable.

## Comparison

| | Mem0 / Zep / Letta | flux7-memory |
|---|---|---|
| **Scope** | Single agent | Multi-agent, multi-role |
| **Human decisions** | Not modeled | First-class facts |
| **Provenance** | None | Agent + timestamp on every fact |
| **Vendor lock-in** | Tied to specific providers | Go binary + HTTP, works with anything |
| **Storage** | Opaque | Markdown files you can read and edit |
| **Deployment** | SaaS or heavy deps | Single binary, zero CGO |

## Current state (May 2026)

**v0.5.0** — 7 MCP tools, Python SDK, hybrid search + LLM reranking, SSE daemon mode, auto-proxy (stdio detects running daemon).

71% LoCoMo benchmark — competitive with VC-backed solutions without gaming the eval.

MIT licensed. [github.com/KTCrisis/flux7-memory](https://github.com/KTCrisis/flux7-memory)

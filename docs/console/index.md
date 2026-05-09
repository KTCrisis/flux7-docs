# flux7-console — AI Agent Governance Platform

## The problem

You deployed agents. flux7-mesh governs them. flux7-memory stores their decisions. Now you need answers :

- **Which agents are running ? What are they doing ?** You have JSONL traces and terminal logs across machines. No single view.
- **Who approved what, when, and why ?** Decisions are in flux7-memory, but querying them requires knowing the key format and tag conventions.
- **Is this new agent safe to promote to production ?** There's no scoring, no lifecycle, no diff showing what changed since the last review.
- **Three agents are waiting for approval at 2am.** Nobody is watching the terminal. The requests time out.

These are management plane problems. flux7-mesh is the data plane (runtime enforcement), flux7-memory is the memory substrate. flux7-console is the visibility and control layer that makes them manageable at scale.

## What flux7-console is

A web-based governance platform for AI agents. Dashboard, approval UI, audit trail, governance engine.

```
flux7-console (management plane — L2 visibility + human control)
├── Trace Viewer      — reads flux7-mesh traces, aggregates stats
├── Session Browser   — session list and drill-down
├── Memory Viewer     — reads flux7-memory, displays decisions + facts
├── Approval UI       — shows pending approvals, human clicks approve/reject
├── OTEL Waterfall    — visual trace timeline from OTEL export
└── (planned) Governance Engine, Agent Catalog, Dependency Graph
```

**Stack :** Next.js 16 + TanStack Query. Backend API and PostgreSQL are scaffolded for future phases. Currently a direct HTTP client of flux7-mesh and flux7-memory.

## How it fits

```
                    flux7-console (visibility + control)
                    ┌────────────────────────────┐
                    │  dashboard, approval UI,   │
                    │  governance, audit trail    │
                    └──────┬──────────┬──────────┘
                           │          │
              reads via    │          │  reads via
              HTTP API     │          │  Python SDK
                           │          │
                           ▼          ▼
┌─────────────────┐    ┌─────────────────┐
│   flux7-mesh    │───►│     flux7-memory        │
│   (runtime)     │    │   (memory)      │
│                 │    │                 │
│ • policy        │    │ • facts         │
│ • approvals     │    │ • decisions     │
│ • traces        │    │ • observations  │
└─────────────────┘    └─────────────────┘
```

flux7-console is a thin client of flux7-mesh and flux7-memory. If flux7-console goes down, everything keeps working — agents are still governed, decisions are still stored. flux7-console adds visibility, not runtime dependency.

Supervisor evaluation (L1) lives in [flux7-supervisor (sup7)](https://github.com/KTCrisis/flux7-supervisor) — a standalone agent that polls flux7-mesh approvals and resolves them via rules + LLM. flux7-console is the human layer (L2), not the judgment layer.

**Agent-agnostic.** flux7-console doesn't know or care which SDK produced the tool call. Claude Code, Managed Agents, LangChain, cron scripts — if it goes through flux7-mesh, flux7-console sees it.

## What it enables

**For the solo dev :** trace viewer shows what your agents did today. Memory viewer shows what decisions were made. You don't need flux7-console on day 1 — flux7-mesh + flux7-memory are enough. Add flux7-console when you want a dashboard instead of `curl`.

**For the team :** approval UI lets any team member resolve pending approvals from a browser. Governance scoring flags risky agents before they hit production. Audit trail answers "who approved that email send at 3am."

**For compliance :** every decision is a fact in flux7-memory. Every tool call is a trace. flux7-console joins them : "this agent called this tool, it was auto-approved because of these 3 past decisions, here's the full chain." Query, don't grep.

## What makes it different

| | Anthropic Console | LangSmith / LangFuse | flux7-console |
|---|---|---|---|
| **Scope** | Anthropic agents only | LangChain ecosystem | Any agent through flux7-mesh |
| **Governance** | Permission policies (allow/ask) | None | Rules, scoring, lifecycle, diffs |
| **Approvals** | Inline in SDK | None | Web UI + API, team-accessible |
| **Memory** | None | Trace replay | flux7-memory integration (decisions as facts) |
| **Policy enforcement** | Basic | None | Full (flux7-mesh data plane) |

Anthropic Console is great for Managed Agents visibility. flux7-console complements it with governance and cross-agent visibility for heterogeneous deployments.

## Current state (May 2026)

- **Dashboard** — Next.js 16 with mesh-oriented routes :
    - `/mesh` — health, connected MCP servers, tool inventory
    - `/mesh/traces` — trace browser
    - `/mesh/sessions` — session list and drill-down
    - `/mesh/approvals` — pending approvals, approve/deny
    - `/mesh/otel` — OTEL waterfall view
    - `/mesh/memory` — flux7-memory browser (search, filter by agent)
- **Supervisor** — migrated to [flux7-supervisor (sup7)](https://github.com/KTCrisis/flux7-supervisor) as a standalone L1 agent (rules + pluggable LLM: Ollama, Anthropic, Claude Code MCP callback)
- **Next** — governance engine (scoring, lifecycle), flux7-memory SDK integration, dependency graph

## Progressive adoption

```
Day 1:   flux7-mesh only — policies + tracing (CLI, zero UI)
Day 30:  + flux7-memory — persistent memory, decision history, auto-approve
Day 60:  + flux7-console — dashboard, team approval UI, governance scoring
```

Each step is independently valuable. flux7-console is the last layer, not the first.

[github.com/KTCrisis/flux7-console](https://github.com/KTCrisis/flux7-console)

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
├── Trace Viewer      — reads flux7-mesh traces, chain of authority, integrity badge
├── Session Browser   — session list and drill-down
├── Memory Viewer     — reads flux7-memory: decisions + facts, the trace behind each, its history, the hash chain status
├── Approval UI       — shows pending approvals, human clicks approve/reject
├── OTEL spans        — spans from the OTLP export, per agent and tool
├── Tools             — catalogue, classification, per-agent decisions, editable
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

**For compliance :** every decision is a fact in flux7-memory. Every tool call is a trace. flux7-console joins them : "this agent called this tool, it was auto-approved because a human approved this read 3 times before, here's the full chain." Query, don't grep.

## What makes it different

| | Anthropic Console | LangSmith / LangFuse | flux7-console |
|---|---|---|---|
| **Scope** | Anthropic agents only | LangChain ecosystem | Any agent through flux7-mesh |
| **Governance** | Permission policies (allow/ask) | None | Rules, scoring, lifecycle, diffs |
| **Approvals** | Inline in SDK | None | Web UI + API, team-accessible |
| **Memory** | None | Trace replay | flux7-memory integration (decisions as facts) |
| **Policy enforcement** | Basic | None | Full (flux7-mesh data plane) |

Anthropic Console is great for Managed Agents visibility. flux7-console complements it with governance and cross-agent visibility for heterogeneous deployments.

## Current state (September 2026)

- **Dashboard**: Next.js 16. The sidebar follows the mesh's decision chain:
    - `/mesh` overview, `/mesh/approvals` (the one queue that waits for a human, L2) and `/mesh/halts`, on top
    - Catalog: `/mesh/agents`, `/mesh/tools`
    - Rules: `/mesh/policies`, `/mesh/grants` (L0 and its temporary exceptions)
    - Delegation: `/mesh/memory` (flux7-memory, past decisions), `/mesh/supervisor` (L1), who decides when no rule does
    - Observe: `/mesh/traces` (tabs *Calls* and *Spans (OTLP)*, the latter at `/mesh/otel`), `/mesh/sessions`
- **Tools**: every tool of every upstream (MCP servers and CLI tools, one card each, clickable as a filter), with its [classification](../mesh7/tool-classification.md) and what the policy decides for a chosen agent. The decision is a selector: changing it rewrites that agent's policy file through mesh7, which validates, applies at once and records the edit in the trace. A *To review* filter lists tools the policy allows without being a plain named read. With mesh7's `pin_tools`, upstream tools held back since the catalogue was pinned appear above the table with their pinned and current descriptions, and can be accepted per tool or per server.
- **Traces** — a call let through by a grant shows its chain of authority: the approval behind the grant (who, when, the reasoning), the grant, then the call, from `GET /traces/{id}/why`. A badge reports the state of mesh7's trace hash chain (`GET /traces/verify`): intact with its sequence range, or the first broken line. The key stays in mesh7.
- **Spans (OTLP)**: the same calls as OTLP spans, per agent and tool; a true waterfall (parent spans, time axis) is still to come.
- **Approvals** — the pending queue, then who settled each past approval (sup7, a human, flux7-memory precedents, expiry) with how long it took and why, as four counters that filter the history; the [precedents](../mesh7/mem7-auto-approve.md) flux7-memory holds per tool and agent (human approvals, other approvals, refusals) with what the next call would do and a Forget button; and mesh7's approval settings (wait for the supervisor, timeout, approval from precedents, approvals needed, writes), edited at runtime.
- **Emergency stop** — stop every agent (two clicks within five seconds), one agent or one session, and resume; a red banner shows on every page while a stop is in force, and the agents table has a Stop / Resume button per agent. See [Emergency stop](../mesh7/emergency-stop.md).
- **Supervisor** — [flux7-supervisor (sup7)](../sup7/index.md), the standalone L1 agent, in three tabs. *Overview*: state, provider chain, rules, the threshold of each provider, the calls sup7 takes, the project dirs, and every question asked to [Jev](../sup7/jev.md) by family with its criteria. *Edit YAML*: sup7.yaml and the question sets, validated and applied by sup7 without restart, and a new question pack. *Evaluate*: pick a case set, recompute for free or replay with its cost shown first, and read [dangers approved](../sup7/measuring.md) first, then normal calls approved, correct denies and the cases to review with their signals.
- **Next** — governance engine (scoring, lifecycle), flux7-memory SDK integration, dependency graph

## Progressive adoption

```
Day 1:   flux7-mesh only — policies + tracing (CLI, zero UI)
Day 30:  + flux7-memory — persistent memory, decision history, auto-approve
Day 60:  + flux7-console — dashboard, team approval UI, governance scoring
```

Each step is independently valuable. flux7-console is the last layer, not the first.

[github.com/KTCrisis/flux7-console](https://github.com/KTCrisis/flux7-console)

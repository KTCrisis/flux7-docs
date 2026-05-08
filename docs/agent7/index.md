# agent7 — AI Agent Governance Platform

## The problem

You deployed agents. agent-mesh governs them. mem7 stores their decisions. Now you need answers :

- **Which agents are running ? What are they doing ?** You have JSONL traces and terminal logs across machines. No single view.
- **Who approved what, when, and why ?** Decisions are in mem7, but querying them requires knowing the key format and tag conventions.
- **Is this new agent safe to promote to production ?** There's no scoring, no lifecycle, no diff showing what changed since the last review.
- **Three agents are waiting for approval at 2am.** Nobody is watching the terminal. The requests time out.

These are management plane problems. agent-mesh is the data plane (runtime enforcement), mem7 is the memory substrate. agent7 is the visibility and control layer that makes them manageable at scale.

## What agent7 is

A web-based governance platform for AI agents. Dashboard, approval UI, audit trail, governance engine.

```
agent7 (management plane)
├── Agent Catalog     — registry, metadata, lifecycle, scoring
├── Trace Viewer      — ingests agent-mesh JSONL, aggregates stats
├── Memory Viewer     — reads mem7 via SDK, displays decisions + facts
├── Approval UI       — shows pending approvals, human clicks approve/reject
├── Governance Engine — rules, 3-axis scoring, severity, validation verdicts
├── Dependency Graph  — declared (YAML) + inferred (traces), impact analysis
└── Diff Engine       — breaking change detection on agent config changes
```

**Stack :** Next.js 16 + FastAPI + PostgreSQL. Auth-agnostic (Supabase SaaS or API key on-prem).

## How it fits

```
                    agent7 (visibility + control)
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
│   agent-mesh    │───►│     mem7        │
│   (runtime)     │    │   (memory)      │
│                 │    │                 │
│ • policy        │    │ • facts         │
│ • approvals     │    │ • decisions     │
│ • traces        │    │ • observations  │
└─────────────────┘    └─────────────────┘
```

agent7 is a thin client of agent-mesh and mem7. If agent7 goes down, everything keeps working — agents are still governed, decisions are still stored. agent7 adds visibility, not runtime dependency.

**Agent-agnostic.** agent7 doesn't know or care which SDK produced the tool call. Claude Code, Managed Agents, LangChain, cron scripts — if it goes through agent-mesh, agent7 sees it.

## What it enables

**For the solo dev :** trace viewer shows what your agents did today. Memory viewer shows what decisions were made. You don't need agent7 on day 1 — agent-mesh + mem7 are enough. Add agent7 when you want a dashboard instead of `curl`.

**For the team :** approval UI lets any team member resolve pending approvals from a browser. Governance scoring flags risky agents before they hit production. Audit trail answers "who approved that email send at 3am."

**For compliance :** every decision is a fact in mem7. Every tool call is a trace. agent7 joins them : "this agent called this tool, it was auto-approved because of these 3 past decisions, here's the full chain." Query, don't grep.

## What makes it different

| | Anthropic Console | LangSmith / LangFuse | agent7 |
|---|---|---|---|
| **Scope** | Anthropic agents only | LangChain ecosystem | Any agent through agent-mesh |
| **Governance** | Permission policies (allow/ask) | None | Rules, scoring, lifecycle, diffs |
| **Approvals** | Inline in SDK | None | Web UI + API, team-accessible |
| **Memory** | None | Trace replay | mem7 integration (decisions as facts) |
| **Policy enforcement** | Basic | None | Full (agent-mesh data plane) |

Anthropic Console is great for Managed Agents visibility. agent7 complements it with governance and cross-agent visibility for heterogeneous deployments.

## Current state (May 2026)

- **Dashboard** — agent-mesh trace viewer, session browser, memory debug view
- **Supervisor** — Python service with rule engine + Ollama LLM evaluation
- **Working** — trace ingestion, session aggregation, pending approval polling
- **Next** — governance engine (scoring, lifecycle), approval UI, mem7 SDK integration, dependency graph

## Progressive adoption

```
Day 1:   agent-mesh only — policies + tracing (CLI, zero UI)
Day 30:  + mem7 — persistent memory, decision history, auto-approve
Day 60:  + agent7 — dashboard, team approval UI, governance scoring
```

Each step is independently valuable. agent7 is the last layer, not the first.

[github.com/KTCrisis/agent7](https://github.com/KTCrisis/agent7)

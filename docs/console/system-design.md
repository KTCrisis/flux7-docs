# System Design — flux7-mesh + flux7-memory + flux7-console

*September 2026. Living document.*

## The stack in one sentence

**flux7-mesh** enforces what agents can do. **flux7-memory** remembers what happened. **flux7-console** shows it all and lets humans intervene.

## Design principles

1. **Each project works alone.** flux7-mesh without flux7-memory still enforces policies. flux7-memory without flux7-mesh is a standalone memory server. flux7-console without the others is a governance dashboard. Integration is opt-in, not required.
2. **Open-source runtime, product on top.** flux7-mesh and flux7-memory are Apache 2.0 — free, adoptable, no lock-in. flux7-console is the management plane that makes them manageable at scale. That's where the business model sits.
3. **Decisions are facts.** When a human approves an action or a supervisor auto-resolves a request, that decision is stored as a queryable fact in flux7-memory. It doesn't vanish into a log file.
4. **Write path matters more than read path.** The system becomes valuable when events automatically flow between components — not when a human manually checks dashboards.

---

## Components

### flux7-mesh (runtime / data plane)

Go binary. Sidecar proxy between agents and their tools.

| What it does | How |
|---|---|
| Policy enforcement | YAML rules: allow, deny, human_approval per tool per agent |
| Approval queue | Pending requests, resolve via API or Claude Code prompt |
| Rate limiting + loop detection | Per-agent, per-tool, configurable |
| Tracing | JSONL trace files, OTEL export, session tracking |
| Grants | Temporary sudo-like bypass for specific agents |
| Supervisor protocol | External process polls pending approvals, auto-resolves routine ones |

**Transports:** MCP stdio (Claude Code, Cursor) · MCP Streamable HTTP at `POST /mcp` (Anthropic Managed Agents, remote clients) · HTTP REST (`POST /tool/{name}`)

**Current state:** v0.16.0, stable. Policy hot-reload, Python SDK, `/decide` endpoint, daemon mode.

### flux7-memory (memory substrate)

Go binary. MCP server for persistent, searchable, governed memory.

| What it does | How |
|---|---|
| Store/recall/search/forget | 7 MCP tools, markdown source of truth + SQLite FTS5 index |
| Hybrid search | BM25 + dense cosine (Ollama/OpenAI) + LLM reranking |
| Structured recall | `memory_context` returns JSON for SDK consumption |
| Tag-scoped access | Any agent reads/writes its own observations via tags |
| Temporal range queries | `since` / `until` filters on RFC3339 timestamps |
| Python SDK | `pip install flux7-memory`, provider-agnostic, wraps all tools via HTTP |

**Current state:** v0.5.1, 71% LoCoMo benchmark, SDK + SSE transport + daemon mode shipped.

### flux7-supervisor (L1 supervisor)

Python agent. Standalone evaluation process between policy engine and human.

| What it does | How |
|---|---|
| Poll pending approvals | Consumes flux7-mesh SDK (`pending()`, `approval_detail()`, `resolve()`) |
| Rule-based evaluation | YAML conditions (tool, params, injection_risk), first-match-wins |
| LLM fallback | Pluggable providers: Ollama, Anthropic, Jev (TypeSafe AI), Claude Code MCP callback (not functional yet: the MCP server is not started) |
| Provider chain | Providers tried in order, next one on failure, circuit breaker skips a failing provider |
| Admin API | HTTP on `127.0.0.1:9096` (off by default, bearer token): status, config, recent decisions, pause/resume |
| Decision persistence | Writes to flux7-memory via SDK (`memory.enabled`, off by default) |

**Current state:** v0.1.0, 85 tests, 4 providers + chain, admin API, systemd unit. Extracted from flux7-console in May 2026, now lives in [flux7-supervisor](https://github.com/KTCrisis/flux7-supervisor). See [flux7-supervisor](../sup7/index.md).

### flux7-console (management plane / product)

Next.js 16 + TanStack Query. Dashboard and human governance UI (L2). The whole product is the frontend: Next.js route handlers proxy `/api/mesh/*`, `/api/mem7/*` and `/api/sup7/*` to the three services and inject their tokens server-side (`MESH_ADMIN_TOKEN`, `MEM7_TOKEN`, `SUP7_ADMIN_TOKEN`), so no token reaches the browser. The `backend/` FastAPI directory is an empty skeleton: no routes, no database.

| What it does | How |
|---|---|
| Trace viewer | Reads flux7-mesh `/traces`, time range and agent/tool/policy filters, `user_id` shown when the credential carried one |
| Trace chain of authority | Per trace, the grant and the call that motivated it, from flux7-mesh `/traces/{id}/why` |
| Integrity badge | Hash-chain status of the trace file, from flux7-mesh `/traces/verify` (the mesh holds the key and runs the check) |
| Session browser | Session list and drill-down via flux7-mesh `/sessions` |
| OTEL waterfall | Visual trace timeline from flux7-mesh `/otel-traces` |
| Agents | Agent list with stats, detail view with tool usage, derived from traces and approvals |
| Human approval UI | `GET /approvals?status=`, then `POST /approvals/{id}/approve` or `/deny` on flux7-mesh |
| Policies | Viewer from flux7-mesh `/policies`; YAML editor that overwrites existing files in `POLICY_DIR` on the console host |
| Grants | List, create and revoke via flux7-mesh `/grants` |
| Tools | Tool catalog (MCP, CLI, REST) from `/tools`, MCP server status from `/mcp-servers` |
| Supervisor | L1 statistics derived from mesh traces; live status, config, recent decisions and pause/resume through the sup7 admin API (`SUP7_URL`, default `http://localhost:9096`) |
| Memory browser | Search, store, edit, forget via flux7-memory JSON-RPC `/rpc` |

Planned (not implemented; `governance-engine/` and `backend/` hold only empty `__init__.py` files) :

| What it will do | How |
|---|---|
| Agent catalog | Registry of declared agents with metadata, scoring, lifecycle |
| Governance engine | Rules, scoring (3-axis), severity escalation, validation verdicts |
| Dependency graph | Declared (YAML) + inferred (traces), impact analysis |
| Diff engine | Breaking change detection on agent config changes |

**Current state:** Early stage, frontend `0.1.0`, no release tag. Dashboard, traces, sessions, agents, approvals, policies, grants, tools, supervisor and memory pages working. Responsive layout (navigation becomes a drawer on phone widths).

---

## The integrated flow

### Today: tool call with human approval

```
Developer → Claude Code → flux7-mesh → gmail.send_email
                                │
                    policy: human_approval
                                │
                    Claude Code shows permission prompt
                                │
                    Developer says "yes"
                                │
                    flux7-mesh resolves → email sent
                    flux7-mesh writes trace → JSONL
                                │
                    Decision is gone. Next time, same question.
```

### Target: tool call with governed memory

```
Developer → Claude Code → flux7-mesh → gmail.send_email
                                │
                    policy: human_approval
                                │
             ┌──────────────────┼──────────────────┐
             │                  │                   │
      Supervisor (L1)     Human (L2)          flux7-console UI
      in flux7-mesh       via Claude Code     shows pending
             │            or flux7-console UI              │
             │                  │                   │
      checks flux7-memory:              │                   │
      "has Marc approved        │                   │
       gmail.send before?"      │                   │
             │                  │                   │
      if yes (3+ times) ───► auto-approve           │
      if no ────────────► escalate to human ◄───────┘
                                │
                    human says "yes"
                                │
                    flux7-mesh resolves → email sent
                    flux7-mesh writes to flux7-memory:
                      key: "decision.gmail.send_email.20260507"
                      value: "approved by Marc, recipient: X, subject: Y"
                      tags: [decision, approval, gmail]
                      agent: "flux7-mesh"
                                │
                    flux7-console displays decision in audit trail
```

### What changes

| Step | Before | After |
|---|---|---|
| Approval source | Always human | Supervisor checks flux7-memory first, escalates if unsure |
| Decision storage | Lost in terminal history | Stored as fact in flux7-memory, queryable |
| Learning | None — same prompt every time | Supervisor learns from past decisions |
| Audit | Grep JSONL logs | flux7-console dashboard, filterable, searchable |
| Visibility | Only the developer who approved | Any team member via flux7-console |

---

## Data flow between components

```
                    flux7-console (management plane)
                    ┌────────────────────────────┐
                    │  dashboard, approval UI,   │
                    │  governance, audit trail    │
                    └──────┬──────────┬──────────┘
                           │          │
              HTTP via     │          │  JSON-RPC /rpc
              proxy        │          │  via proxy
                           │          │
                           ▼          ▼
┌─────────────────┐    ┌─────────────────┐
│   flux7-mesh    │───►│     flux7-memory        │
│   (runtime)     │    │   (memory)      │
│                 │    │                 │
│ • traces JSONL  │    │ • facts         │
│ • approvals     │    │ • decisions     │
│ • policies      │    │ • observations  │
└────────┬────────┘    └─────────────────┘
         │                     ▲
         │  writes decisions   │
         └─────────────────────┘
         (opt-in, if flux7-memory configured)
```

### Write paths (who writes what where)

| Writer | Target | What | When |
|---|---|---|---|
| Any agent | flux7-memory | Observations, facts, context | During execution, via MCP store |
| flux7-mesh | flux7-memory | Approval/rejection decisions | On approval resolve (opt-in) |
| flux7-mesh | JSONL | Tool call traces | Every tool call (always) |
| Supervisor | flux7-memory | Auto-resolve rationale | On auto-approve (opt-in) |
| Human via flux7-console | flux7-mesh API | Approve/deny, grant create/revoke | Clicking in UI |
| Human via flux7-console | Policy YAML files | Edited policy | Saving in the policy editor |
| Human via flux7-console | sup7 admin API | Pause/resume | Clicking in UI |

### Read paths (who reads what from where)

| Reader | Source | What | When |
|---|---|---|---|
| Supervisor | flux7-memory | Past decisions for same pattern | Before deciding to auto-approve |
| Any agent | flux7-memory | Stored facts, context, history | During execution, via MCP search |
| flux7-console | flux7-mesh API | Pending approvals, traces, sessions, policies, grants, tools | Polling (TanStack Query) |
| flux7-console | flux7-memory `/rpc` | Stored decisions, facts | For memory browser |
| flux7-console | sup7 admin API | Status, config, recent decisions | Supervisor page |

---

## Approval architecture (detailed)

### Trust hierarchy

```
Level 0: Policy engine (flux7-mesh)
         Static rules. Instant. No judgment.
         allow reads, deny deletes, require approval for sends.

Level 1: Built-in supervisor (in flux7-mesh, Go)
         flux7-memory lookup. ~100ms. Pattern matching only.
         Checks past decisions: 3+ approvals, 0 rejections → auto-approve.
         Escalates unknowns to Level 1+.

Level 1+: External supervisor (flux7-supervisor / sup7)
          Rule engine + pluggable LLM (Ollama/Anthropic/Jev, chainable). ~2s rules, ~20s LLM.
          Handles novel cases, complex conditions, injection detection.
          Escalates unknowns to Level 2.

Level 2: Human
         Full judgment. Slow. Expensive attention.
         Sees pending in Claude Code prompt OR flux7-console UI.
         Decision written back to flux7-memory via flux7-mesh.
```

> **Two supervisor layers.** Level 1 is built into flux7-mesh — a `MemoryReader`
> that queries flux7-memory before the approval queue. It's a pre-filter, not a full
> resolver. Level 1+ is [flux7-supervisor (sup7)](https://github.com/KTCrisis/flux7-supervisor),
> a standalone Python agent that implements the
> [supervisor protocol](https://github.com/KTCrisis/flux7-mesh/blob/main/docs/supervisor-protocol.md)
> — it polls pending approvals and resolves them with rule evaluation + pluggable LLM
> (Ollama, Anthropic, Jev, or a chain of them; the Claude Code MCP callback is not functional yet).
> The `supervisor/` package inside flux7-mesh handles content redaction and
> injection detection on the protocol's outbound payloads (`RedactParams`,
> `DetectInjection`) — a separate concern from both layers.

### Supervisor decision logic (pseudocode)

```python
def evaluate(request, mem7_client):
    # Check flux7-memory for similar past decisions
    past = mem7_client.context(
        f"{request.tool} {request.agent}",
        tags=["decision"],
        limit=10
    )

    approved_count = sum(1 for m in past if "approved" in m.value)
    rejected_count = sum(1 for m in past if "rejected" in m.value)

    # Pattern: consistently approved → auto-approve
    if approved_count >= 3 and rejected_count == 0:
        return AutoApprove(reason=f"approved {approved_count} times before")

    # Pattern: recently rejected → auto-deny
    recent_reject = [m for m in past if "rejected" in m.value and is_recent(m, days=7)]
    if recent_reject:
        return AutoDeny(reason="rejected recently, escalate")

    # Unknown pattern → escalate to human
    return Escalate(reason="no clear precedent")
```

### Where the approval UI lives

**Claude Code terminal** — the developer gets the prompt inline. This is the current flow, works for solo use.

**flux7-console web UI** — for team use, overnight runs, or when multiple agents generate approvals faster than one human can handle. flux7-console polls flux7-mesh's pending queue, displays context, human clicks. flux7-console POSTs back to flux7-mesh `/approvals/{id}/approve` or `/deny`.

Both are **thin clients** of the flux7-mesh approval API. If flux7-console goes down, Claude Code still works. If both go down, requests queue in flux7-mesh until someone resolves them (fail-safe, not fail-open).

---

## Deployment

### Solo developer (current setup)

```
laptop
├── flux7-mesh (Go binary, sidecar)
│   ├── config.yaml (policies)
│   └── traces/ (JSONL)
├── flux7-memory (Go binary, MCP stdio via flux7-mesh)
│   └── ~/.mem7/ (markdown + SQLite)
└── Claude Code / Cursor (agent)
```

No flux7-console needed. flux7-mesh + flux7-memory provide full governance + memory. The developer is the human-in-the-loop via terminal prompts.

### Small team

```
shared server
├── flux7-mesh (central, HTTP mode)
├── mem7 serve (HTTP, shared memory)
├── sup7 (L1 supervisor, polls mesh approvals)
└── flux7-console (dashboard + approval UI, systemd unit, loopback :8790)

developer laptops
└── agents connect to shared flux7-mesh
```

flux7-console adds value: team visibility, approval UI for shared agents, audit trail. sup7 handles automated evaluation.

flux7-console ships a systemd unit, `deploy/flux7-console.service`, which runs `next start` on `127.0.0.1:8790` after `mesh7.service`; tokens and URLs come from an optional `EnvironmentFile`. sup7 ships its own unit (`contrib/systemd/sup7.service`). Reaching the console from a phone goes through whatever puts that loopback port on your network (reverse proxy, tunnel); the layout is usable at phone width.

### Enterprise (future)

```
flux7-console SaaS (hosted)
├── governance engine (planned)
├── approval UI
├── audit + compliance
└── policy push → customer flux7-mesh

customer infra
├── flux7-mesh (sidecar per agent)
├── flux7-memory (per-team or central)
├── sup7 (L1 supervisor)
└── agents (any framework)
```

### Anthropic Managed Agents (cloud harness + governed tools)

```
Anthropic cloud (harness, sandbox, multi-agent orchestration)
├── Managed Agent coordinator (Opus)
│   ├── sub-agents (reviewer, tester, security)
│   └── mcp_toolset → MCP Streamable HTTP
│
└── POST /mcp ──────────────────────────────►
                                              your infra
                                              ├── flux7-mesh (policies, approval, traces)
                                              │   └── POST /rpc ──► flux7-memory (decisions)
                                              └── upstream MCP servers (ollama, arch7, ...)
```

**What Anthropic provides:** cloud sandbox, session durability, multi-agent coordination (threads), outcome grading (rubrics), vault credential management.

**What flux7-mesh adds:** fine-grained deny policies (Managed Agents only have allow/ask), per-agent rate limiting, temporal grants, decision persistence to flux7-memory, OTEL traces, governed memory cross-session.

**Integration point:** flux7-mesh `POST /mcp` endpoint serves MCP Streamable HTTP. Managed Agent's MCP connector discovers tools via `tools/list`, calls them via `tools/call`. Policies apply transparently. Vault injects `Authorization: Bearer agent:<id>` for per-agent policy evaluation.

**flux7-console role:** same as for any deployment: dashboard, approval UI, audit trail. The Managed Agent sessions generate traces and decisions that flow to flux7-console like any other agent.

---

## Implementation priorities

### Phase 1: Decision write path (flux7-mesh → flux7-memory) ✓

*Shipped in flux7-mesh v0.8.7.*

When an approval resolves in flux7-mesh, the decision is stored in flux7-memory if configured.

- Config field: `memory.url` + optional `memory.token`
- On resolve: async POST to flux7-memory `/rpc` with `memory_store`
- Key format: `decision.<tool>.<approval_id>`
- Tags: `[decision, approved|denied, <tool>, agent:<id>]`
- Value: human-readable summary — "approved by X — agent:Y tool:Z reason:..."
- Metrics: `agent_mesh_mem7_writes_{attempted,succeeded,failed}_total`
- Graceful degradation: failing flux7-memory never blocks approvals

### Phase 2: Built-in supervisor reads flux7-memory ✓

*Shipped in flux7-mesh v0.9.1.*

flux7-mesh queries flux7-memory before submitting to the approval queue. This is the built-in Level 1 supervisor — a pre-filter that handles routine patterns.

- `MemoryReader` queries flux7-memory `memory_search` with tool name + agent + tags=["decision"]
- Counts past approvals/rejections from search results
- Auto-approve if >= `min_approvals` (default 3) with 0 rejections
- Escalate if ambiguous, rejected, or flux7-memory is down
- Auto-approved decisions traced as `supervisor:mem7` and written back to flux7-memory
- Config: `supervisor.auto_approve` (default true), `supervisor.min_approvals` (default 3)

**Complements the external Python supervisor** ([flux7-supervisor](../sup7/index.md), formerly in flux7-console): the built-in handles routine patterns (~100ms); the external supervisor handles novel cases with rule engine + LLM evaluation (~20s). Both escalate unknowns to humans.

### Phase 3: flux7-console reads flux7-memory ✓

Shipped in the frontend, without a Python backend and without the SDK.

- Next.js route handler `/api/mem7/*` proxies to flux7-memory (`MEM7_URL`, optional `MEM7_TOKEN`)
- `lib/api/mem7.ts` calls JSON-RPC `/rpc` with `tools/call`: `memory_list`, `memory_recall`, `memory_search`, `memory_store`, `memory_forget`
- Memory browser page: search, store, edit, forget
- Planned: a decision audit trail joining stored decisions with traces

### Phase 4: flux7-console approval UI as thin client ✓

Shipped in the frontend.

- `GET /approvals?status=` on flux7-mesh through the `/api/mesh/*` proxy; pending count in the navigation
- Display: tool call details, agent identity, resolution history
- Action: approve/deny → `POST /approvals/{id}/approve` or `/deny`
- flux7-mesh handles the flux7-memory write (phase 1), flux7-console doesn't write decisions to flux7-memory itself
- Planned: past decisions from flux7-memory shown as context next to a pending approval

---

## What this enables (end state)

1. **Adaptive governance.** Policies start strict (human approval for everything). As the system collects approved patterns in flux7-memory, the supervisor auto-approves routine actions. Governance gets less intrusive over time without getting less safe.

2. **Cross-agent memory.** Agent A stores an observation. Agent B searches and finds it. The supervisor checks if a decision was already made. All through the same flux7-memory store, scoped by tags.

3. **Audit without effort.** Every decision is a fact in flux7-memory. Every tool call is a trace in flux7-mesh. flux7-console joins them: "this agent called this tool, it was approved because of this past decision, here's the full chain." Compliance teams query, they don't grep.

4. **Progressive adoption.** Day 1: install flux7-mesh, get policies and tracing. Day 30: add flux7-memory, get persistent memory. Day 60: add flux7-console, get visibility and team governance. Each step is independently valuable.

---

*Three projects, one system. Independent by default, powerful together.*

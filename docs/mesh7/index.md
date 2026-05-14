# flux7-mesh — Governance Mesh for AI Agents

## The problem

You're deploying agents. They call tools — file writes, emails, API calls, database queries. You need to answer three questions before going to production :

- **Which agent can call which tool ?** Frameworks don't enforce boundaries. An agent can call anything it discovers.
- **Who approved that action ?** The developer clicked "yes" in a terminal prompt 3 weeks ago. That decision is gone.
- **What happened ?** You have stdout logs somewhere. They're not structured, not queryable, and definitely not auditable.

These aren't agent framework problems. They're infrastructure problems. Service meshes solved them for microservices a decade ago — policy enforcement, observability, access control at the network layer. Agents need the same thing, at the tool call layer.

## What flux7-mesh is

A sidecar proxy that sits between agents and their tools. One Go binary, one YAML config, zero dependencies.

```
Agent (Claude, LangChain, script)
  │
  └──► flux7-mesh (sidecar)
         ├── policy: allow / deny / human_approval
         ├── rate limiting + loop detection
         ├── temporal grants (sudo for agents)
         ├── approval queue (async, non-blocking)
         ├── traces (JSONL + OTEL)
         └──► tools (MCP servers, OpenAPI, CLI binaries)
```

Agents don't know the proxy exists. They call tools, get results. The governance layer is invisible to the agent, visible to the operator.

**Transports :** MCP stdio (Claude Code, Cursor) · MCP Streamable HTTP at `POST /mcp` (Anthropic Managed Agents, remote clients) · HTTP REST (`POST /tool/{name}`)

## Adaptive governance

Policies start strict. Over time, the system learns.

```
Day 1:  human_approval for all writes
        ↓ human approves filesystem.write 3 times
Day 7:  flux7-mesh queries flux7-memory → 3 approvals, 0 rejections → auto-approve
        ↓ novel tool call, no history
        ↓ external supervisor (rules + LLM) evaluates → approve
Day 30: routine patterns auto-resolve in ~100ms
        humans only see genuinely new or ambiguous requests
```

Three layers :

| Level | Who | Speed | What it handles |
|-------|-----|-------|-----------------|
| 0 | Policy engine | 0ms | Static rules (allow, deny, human_approval) |
| 1 | Built-in flux7-memory lookup | ~100ms | Routine patterns (3+ past approvals) |
| 1+ | External supervisor | ~20s | Novel cases (rule engine + LLM) |
| 2 | Human | minutes | Unknowns, high-stakes decisions |

Every decision is stored as a fact in [flux7-memory](https://github.com/KTCrisis/flux7-memory). Every tool call is a trace. Both are queryable.

## What makes it different

| | API Gateways (Kong, Apigee) | Agent Frameworks (LangChain, CrewAI) | flux7-mesh |
|---|---|---|---|
| **Traffic** | North-south (user → LLM) | Internal (agent runtime) | East-west (agent → tools) |
| **Policy** | API keys, rate limits | None or coarse allow/ask | Semantic YAML rules per agent per tool |
| **Approval** | None | Framework-specific | Async queue, non-blocking, with memory |
| **Identity** | API consumer | Single agent | Per-agent (`agent:claude`, `agent:worker-3`) |
| **Decision persistence** | None | None | Facts in flux7-memory, queryable, auditable |
| **Deployment** | Heavy infrastructure | Embedded in code | Single binary sidecar, zero config to start |

Closest comparable : Microsoft Agent Governance Toolkit. But middleware vs sidecar — flux7-mesh requires zero changes to agent code.

## Current state (May 2026)

- **v0.13.0** — 281 Go tests + 49 Python SDK tests, 16 packages, race clean
- **Import** — MCP servers (stdio + SSE), OpenAPI specs, CLI binaries
- **Export** — MCP stdio + MCP Streamable HTTP + HTTP REST
- **Governance** — YAML policies, glob patterns, conditions, per-agent policy files, specificity sort, hot-reload
- **Policy API** — `POST /decide` evaluates policy without executing, `GET /policies` exposes active rules
- **Auth** — [JWT validation](jwt-auth.md) against external IdPs (Cloudflare Access, Auth0, Keycloak), JWKS cached with background refresh. Legacy `Bearer agent:<name>` still works
- **Approval** — async queue, temporal grants, supervisor protocol, flux7-memory auto-approve
- **Observability** — JSONL traces, OTEL export, session tracking, Prometheus metrics
- **Durable state** — approvals and grants persisted in SQLite, survive restarts (`storage_path: state.db`)
- **Auto-proxy** — in MCP mode, detects running daemon and becomes a thin stdio→HTTP proxy (zero config change, solves port conflicts)
- **Daemon mode** — `mesh7 serve` runs as persistent daemon, MCP clients auto-proxy to it
- **Python SDK** — `pip install flux7-mesh` v0.4.0 — GovernedToolkit (namespace-qualified tool names), MeshHooks (Anthropic Agent SDK integration), direct HTTP client
- **Integrations** — [flux7-memory](https://github.com/KTCrisis/flux7-memory) (decision persistence + auto-approve), [flux7-console](https://github.com/KTCrisis/flux7-console) (dashboard + governance UI)
- **Next** — claim-based policy conditions, Claude Connectors Directory listing

## Claude ecosystem integration

flux7-mesh and flux7-memory cover every Claude surface with a native integration path.

| Surface | flux7-mesh | flux7-memory |
|---|---|---|
| **Claude Code / Cursor** | MCP stdio (auto-proxy if daemon running) | MCP stdio (auto-proxy if daemon running) |
| **Claude Platform / Console** | MCP Streamable HTTP (`POST /mcp`) | Via flux7-mesh (tools `memory.*`) |
| **Managed Agents** | MCP connector URL → `POST /mcp` | Via flux7-mesh (tools `memory.*`) |
| **Claude API (raw)** | Python SDK + `POST /decide` | Python SDK (`pip install flux7-memory`) |
| **Agent SDK (custom)** | HTTP direct (`/tool/{name}`, `/decide`) | HTTP direct (`/rpc`) |

**Key insight** : flux7-memory access from Platform, Console, and Managed Agents goes through flux7-mesh policy — no direct exposure. This means governance is enforced at every layer, not just in local development.

**Gaps (tracked)** : MCP registry listing, `memory_context` system prompt helper for Platform, claim-based policy conditions.

## Get started

```bash
# Install
go install github.com/KTCrisis/flux7-mesh/cmd/mesh7@latest

# Add to Claude Code
claude mcp add mesh7 -- mesh7 --mcp --config config.yaml

# Or run standalone
mesh7 --config config.yaml
```

Apache 2.0 licensed. [github.com/KTCrisis/flux7-mesh](https://github.com/KTCrisis/flux7-mesh)

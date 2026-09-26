# flux7-mesh — Governance Mesh for AI Agents

## The problem

You're deploying agents. They call tools — file writes, emails, API calls, database queries. You need to answer three questions before going to production:

- **Which agent can call which tool?** Frameworks don't enforce boundaries. An agent can call anything it discovers.
- **Who approved that action?** The developer clicked "yes" in a terminal prompt 3 weeks ago. That decision is gone.
- **What happened?** You have stdout logs somewhere. They're not structured, not queryable, and definitely not auditable.

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

**Transports:** MCP stdio (Claude Code, Cursor) · MCP Streamable HTTP at `POST /mcp` (Anthropic Managed Agents, remote clients) · HTTP REST (`POST /tool/{name}`)

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

Three layers:

| Level | Who | Speed | What it handles |
|-------|-----|-------|-----------------|
| 0 | Policy engine | 0ms | Static rules (allow, deny, human_approval) |
| 1 | Built-in flux7-memory lookup | ~100ms | Routine patterns (3+ past approvals) |
| 1+ | External supervisor | ~20s | Novel cases (rule engine + LLM) |
| 2 | Human | minutes | Unknowns, high-stakes decisions |

Every decision is stored as a fact in [flux7-memory](https://github.com/KTCrisis/flux7-memory). Every tool call is a trace. Both are queryable.

## What makes it different

Three things, and only three. The rest is table stakes, well executed.

**1. The enforcement point is protocol-agnostic.** A policy attaches to a tool's identity, not to the transport carrying it. The same rule governs a stdio MCP server, an imported OpenAPI operation and a local binary, in one catalog. Change your agent framework or your tool protocol and the rule survives both.

**2. The audit trail is causal, not chronological.** A trace carries `parent_trace_id` and `grant_id`. `GET /traces/{id}/why` walks the chain back, so the answer is not "allowed by rule X" but "allowed by grant G, which came from approval A, which came from call T". Most systems record what happened. This one records why it was permitted, back to the human decision.

**3. Enforcement is decoupled from proxying.** [`POST /decide`](writing-policies.md) evaluates policy without executing anything. A PreToolUse hook asks "would you allow this?" and enforces locally, which means flux7-mesh governs tools that never transit it, including a harness's built-in tools. That is the difference between a proxy and a policy authority.

| | API Gateways (Kong, Apigee) | Agent Frameworks (LangChain, CrewAI) | flux7-mesh |
|---|---|---|---|
| **Traffic** | North-south (user → LLM) | Internal (agent runtime) | East-west (agent → tools) |
| **Policy** | API keys, rate limits | None or coarse allow/ask | Semantic YAML rules per agent per tool |
| **Approval** | None | Framework-specific | Async queue, non-blocking, with memory |
| **Identity** | API consumer | Single agent | Per-agent (`agent:claude`, `agent:worker-3`) |
| **Decision persistence** | None | None | Facts in flux7-memory, queryable, auditable |
| **Deployment** | Heavy infrastructure | Embedded in code | Single binary sidecar, zero config to start |

Closest comparable: Microsoft Agent Governance Toolkit. But middleware vs sidecar — flux7-mesh requires zero changes to agent code.

## Current state (September 2026)

- **v0.16.0** — 412 Go test functions across 17 packages, race clean, plus 77 Python SDK tests
- **Import** — MCP servers over stdio, SSE and Streamable HTTP (`transport: streamable-http` for hosted upstreams); OpenAPI specs (URL or file); CLI binaries whose `default_action` acts as a tightening-only floor under the dispatcher
- **Export** — MCP stdio + MCP Streamable HTTP + HTTP REST
- **Governance** — YAML policies, glob patterns, numeric conditions and string operators (`contains`, `not_contains`, `starts_with`, `not_starts_with`) on arguments, per-agent policy files, specificity sort, hot-reload
- **Policy API** — `POST /decide` evaluates policy without executing, `GET /policies` exposes active rules with the file each came from (`source_file`); `mesh7-hook`, a Claude Code PreToolUse hook, asks `POST /decide` before every tool call, including the harness's built-in tools that never transit the proxy
- **Auth** — [JWT validation](jwt-auth.md) against external IdPs (Cloudflare Access, Auth0, Keycloak), JWKS cached with background refresh. `auth.jwt.user_claim` records the human the agent acts for, exported as `enduser.id`. The legacy `Bearer agent:<name>` form is opt-in and off by default once JWT is configured. An MCP Streamable HTTP session is bound to the identity that opened it: another caller presenting its `Mcp-Session-Id` gets a `403`
- **Approval** — async queue, routing via `queue | tty | tty-fallback`, temporal grants that record their origin, [supervisor protocol](supervisor-protocol.md), flux7-memory auto-approve
- **Content safety** — raw parameters can be withheld from an external resolver, and a prompt-injection tripwire suppresses auto-approval so the call escalates to a human instead
- **Observability** — JSONL traces with rotation, OTEL export (batched, retried on network errors, 429 and 5xx, GenAI semantic conventions, custom headers and TLS options), session tracking, Prometheus metrics, token accounting that distinguishes real provider counts from estimates. Each trace line is hash-chained to the previous one (HMAC-SHA256 when `MESH_TRACE_KEY` is set); `mesh7 trace verify` checks files offline and `GET /traces/verify` checks the running store ([Trace Integrity](trace-integrity.md))
- **Lineage** — traces carry `grant_id` and `parent_trace_id`, grants carry the `approval_id` and `trace_id` they came from; `GET /traces/{id}/why` walks the chain from a call back to the human approval that authorised it
- **Durable state** — approvals and grants persisted in SQLite, survive restarts (`storage_path: state.db`)
- **Auto-proxy** — in MCP mode, detects a running instance and becomes a thin stdio→HTTP shuttle (zero config change, removes the port conflict between clients)
- **Daemon mode** — `mesh7 serve` runs as a persistent daemon, MCP clients auto-proxy to it
- **Python SDK** — `pip install flux7-mesh` v0.6.0 — `AgentMesh(token=...)` for JWT identity, GovernedToolkit (namespace-qualified tool names), MeshHooks (Anthropic Agent SDK), and `mesh7-hook` for the Claude Code CLI
- **Integrations** — [flux7-memory](https://github.com/KTCrisis/flux7-memory) (decision persistence + auto-approve), [flux7-console](https://github.com/KTCrisis/flux7-console) (dashboard + governance UI), [flux7-supervisor](https://github.com/KTCrisis/flux7-supervisor) (L1 evaluation agent)
- **Next** — claim-based policy conditions, semantic conditions beyond text matching

## Known limits

Stated plainly, because a governance tool that oversells itself is worse than none.

- Policy conditions match text, case-sensitively. Denying `rm -rf` does not stop `RM -RF` or a base64 payload.
- Injection detection is a regex tripwire wired to auto-approval, not a defence.
- CLI positional arguments are metacharacter-filtered but not allowlisted; the allowlist governs flags.
- Rate-limit counters live in memory and reset with the daemon. Approvals and grants do not.
- The `--mcp` auto-proxy probes localhost, so it is a same-host mechanism.

## Claude ecosystem integration

flux7-mesh and flux7-memory cover every Claude surface with a native integration path.

| Surface | flux7-mesh | flux7-memory |
|---|---|---|
| **Claude Code / Cursor** | MCP stdio (auto-proxy if daemon running) | MCP stdio (auto-proxy if daemon running) |
| **Claude Platform / Console** | MCP Streamable HTTP (`POST /mcp`) | Via flux7-mesh (tools `memory.*`) |
| **Managed Agents** | MCP connector URL → `POST /mcp` | Via flux7-mesh (tools `memory.*`) |
| **Claude API (raw)** | Python SDK + `POST /decide` | Python SDK (`pip install flux7-memory`) |
| **Agent SDK (custom)** | HTTP direct (`/tool/{name}`, `/decide`) | HTTP direct (`/rpc`) |

**Key insight**: flux7-memory access from Platform, Console, and Managed Agents goes through flux7-mesh policy — no direct exposure. This means governance is enforced at every layer, not just in local development.

**Gaps (tracked)**: MCP registry listing, `memory_context` system prompt helper for Platform, claim-based policy conditions.

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

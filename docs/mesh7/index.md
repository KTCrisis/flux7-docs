# flux7-mesh: Governance Mesh for AI Agents

## The problem

You're deploying agents. They call tools such as file writes, emails, API calls and database queries. You need to answer three questions before going to production:

- **Which agent can call which tool?** Frameworks don't enforce boundaries. An agent can call anything it discovers.
- **Who approved that action?** The developer clicked "yes" in a terminal prompt 3 weeks ago. That decision is gone.
- **What happened?** You have stdout logs somewhere. They are neither structured nor queryable, so they cannot serve as an audit trail.

These problems belong to the infrastructure rather than to the agent framework. Service meshes solved them for microservices a decade ago with policy enforcement, observability and access control at the network layer. Agents need the same thing at the tool call layer.

## What flux7-mesh is

flux7-mesh is a sidecar proxy that sits between agents and their tools. It ships as one Go binary with one YAML config and zero dependencies.

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

Agents don't know the proxy exists. They call tools and get results. The operator sees the governance layer, whereas the agent does not.

**Transports:** MCP stdio (Claude Code, Cursor) · MCP Streamable HTTP at `POST /mcp` (Anthropic Managed Agents, remote clients) · HTTP REST (`POST /tool/{name}`)

## Where it sits

flux7-mesh is the first of up to four layers a call can meet. A rule decides
most calls on the spot. When a rule sends a call to approval, precedents in
[flux7-memory](../mem7/index.md) can settle it, then
[flux7-supervisor](../sup7/index.md), and only then a human. Only the mesh
is required. See [How it works](../how-it-works.md).

Every tool call is a trace, and every decision can be kept as a fact in
flux7-memory. Both are queryable.

## What makes it different

Three properties set flux7-mesh apart. The other features are standard capabilities, well executed.

**1. The enforcement point is protocol-agnostic.** A policy attaches to a tool's identity, not to the transport carrying it. The same rule governs a stdio MCP server, an imported OpenAPI operation and a local binary, in one catalog. Change your agent framework or your tool protocol and the rule survives both.

**2. The audit trail is causal, not chronological.** A trace carries `parent_trace_id` and `grant_id`. `GET /traces/{id}/why` walks the chain back, so the answer is not "allowed by rule X" but "allowed by grant G, which came from approval A, which came from call T". Most systems record what happened, whereas flux7-mesh also records why the call was permitted, back to the human decision.

**3. Enforcement is decoupled from proxying.** [`POST /decide`](writing-policies.md) evaluates policy without executing anything. A PreToolUse hook asks "would you allow this?" and enforces locally, which means flux7-mesh governs tools that never transit it, including a harness's built-in tools. Hence flux7-mesh acts as a policy authority rather than only as a proxy.

| | API Gateways (Kong, Apigee) | Agent Frameworks (LangChain, CrewAI) | flux7-mesh |
|---|---|---|---|
| **Traffic** | North-south (user → LLM) | Internal (agent runtime) | East-west (agent → tools) |
| **Policy** | API keys, rate limits | None or coarse allow/ask | Semantic YAML rules per agent per tool |
| **Approval** | None | Framework-specific | Async queue, non-blocking, with memory |
| **Identity** | API consumer | Single agent | Per-agent (`agent:claude`, `agent:worker-3`) |
| **Decision persistence** | None | None | Facts in flux7-memory, queryable, auditable |
| **Deployment** | Heavy infrastructure | Embedded in code | Single binary sidecar, zero config to start |

The closest comparable is Microsoft Agent Governance Toolkit. However, that toolkit is a middleware, whereas flux7-mesh is a sidecar and requires zero changes to agent code.

## Current state (September 2026)

- **v0.17.2**: 454 Go test functions across 18 packages, plus 80 Python SDK tests
- **Import**: MCP servers over stdio, SSE and Streamable HTTP (`transport: streamable-http` for hosted upstreams); OpenAPI specs (URL or file); CLI binaries whose `default_action` acts as a tightening-only floor under the dispatcher
- **Export**: MCP stdio + MCP Streamable HTTP + HTTP REST
- **Governance**: YAML policies, glob patterns, numeric conditions and string operators (`contains`, `not_contains`, `starts_with`, `not_starts_with`) on arguments, per-agent policy files, specificity sort, hot-reload
- **Policy API**: `POST /decide` evaluates policy without executing, `GET /policies` exposes active rules with the file each came from (`source_file`); `mesh7-hook`, a Claude Code PreToolUse hook, asks `POST /decide` before every tool call, including the harness's built-in tools that never transit the proxy
- **Auth**: [JWT validation](jwt-auth.md) against external IdPs (Cloudflare Access, Auth0, Keycloak), JWKS cached with background refresh. `auth.jwt.user_claim` records the human the agent acts for, exported as `enduser.id`. The legacy `Bearer agent:<name>` form is opt-in and off by default once JWT is configured. An MCP Streamable HTTP session is bound to the identity that opened it: another caller presenting its `Mcp-Session-Id` gets a `403`
- **Approval**: async queue, routing via `queue | tty | tty-fallback`, temporal grants that record their origin, [supervisor protocol](supervisor-protocol.md), flux7-memory auto-approve
- **Content safety**: raw parameters can be withheld from an external resolver, and a prompt-injection tripwire suppresses auto-approval so the call escalates to a human instead
- **Observability**: JSONL traces with rotation, OTEL export (batched, retried on network errors, 429 and 5xx, GenAI semantic conventions, custom headers and TLS options), session tracking, Prometheus metrics, token accounting that distinguishes real provider counts from estimates. Each trace line is hash-chained to the previous one (HMAC-SHA256 when `MESH_TRACE_KEY` is set); `mesh7 trace verify` checks files offline and `GET /traces/verify` checks the running store ([Trace Integrity](trace-integrity.md))
- **Lineage**: traces carry `grant_id` and `parent_trace_id`, grants carry the `approval_id` and `trace_id` they came from; `GET /traces/{id}/why` walks the chain from a call back to the human approval that authorised it
- **Durable state**: approvals and grants persisted in SQLite, survive restarts (`storage_path: state.db`)
- **Auto-proxy**: in MCP mode, detects a running instance and becomes a thin stdio→HTTP shuttle (zero config change, removes the port conflict between clients)
- **Daemon mode**: `mesh7 serve` runs as a persistent daemon, MCP clients auto-proxy to it
- **Python SDK**: `pip install flux7-mesh` v0.6.1 provides `AgentMesh(token=...)` for JWT identity, GovernedToolkit (namespace-qualified tool names), MeshHooks (Anthropic Agent SDK), and `mesh7-hook` for the Claude Code CLI
- **Integrations**: [flux7-memory](https://github.com/KTCrisis/flux7-memory) (decision persistence + auto-approve), [flux7-console](https://github.com/KTCrisis/flux7-console) (dashboard + governance UI), [flux7-supervisor](https://github.com/KTCrisis/flux7-supervisor) (L1 evaluation agent)
- **Next**: claim-based policy conditions, semantic conditions beyond text matching

## Known limits

These limits are stated plainly, because a governance tool that oversells itself is worse than none.

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

flux7-memory access from Platform, Console, and Managed Agents goes through flux7-mesh policy, without direct exposure. Therefore governance is enforced at every layer, and not only in local development.

**Gaps (tracked)**: MCP registry listing, `memory_context` system prompt helper for Platform, claim-based policy conditions.

## Get started

```bash
# Install (mesh7 and the mesh approval CLI)
curl -fsSL https://raw.githubusercontent.com/KTCrisis/flux7-mesh/main/install.sh | sh

# Add to Claude Code
claude mcp add mesh7 -- mesh7 --mcp --config config.yaml

# Or run standalone
mesh7 --config config.yaml
```

Apache 2.0 licensed. [github.com/KTCrisis/flux7-mesh](https://github.com/KTCrisis/flux7-mesh)

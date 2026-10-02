# Features

State as of September 2026. **Stable**: used daily, tested, documented.
**Early**: works, recent, may change. **Planned**: not there yet.

## flux7-mesh: policy, approvals, traces

v0.17.2 · Go · [overview](../mesh7/index.md)

| Capability | State | Details |
|---|---|---|
| Proxy MCP servers (stdio, SSE, Streamable HTTP) | Stable | [Configuration](../mesh7/configuration.md) |
| Import OpenAPI specs and CLI binaries as tools | Stable | [CLI tools](../mesh7/cli-tools.md) |
| Serve agents over MCP stdio, MCP HTTP and REST | Stable | [Deployment modes](../mesh7/deployment-modes.md) |
| Policies per agent and per tool: allow, deny, human approval | Stable | [Writing policies](../mesh7/writing-policies.md) |
| Conditions on arguments (numbers, text operators) | Stable | [Writing policies](../mesh7/writing-policies.md) |
| Hot reload of policies | Stable | |
| Decide without executing (`POST /decide`), for tools outside the proxy | Stable | [Agent security](../mesh7/security.md) |
| Claude Code hook covering built-in tools | Stable | [Python SDK](../mesh7/python-sdk.md) |
| Approval queue, answered from CLI, console or API | Stable | [Approval flow](../mesh7/approval-flow.md) |
| Temporary grants ("sudo for agents") | Stable | [Approval flow](../mesh7/approval-flow.md) |
| Emergency stop of one agent, one session or everything, from CLI, console or API | Early | since v0.18.0, [Emergency stop](../mesh7/emergency-stop.md) |
| Approvals and grants survive restarts | Stable | |
| Rate limiting and loop detection | Stable | in-memory, reset on restart |
| JWT identity from an external IdP, end user recorded | Stable | [JWT authentication](../mesh7/jwt-auth.md) |
| Authenticated control plane | Stable | [Control plane auth](../mesh7/control-plane-auth.md) |
| Tool classification (read, write, destructive) and draft policies | Early | [Tool classification](../mesh7/tool-classification.md) |
| Catalogue pinning: hold back new or changed upstream tools | Early | opt-in |
| Hide from `tools/list` what the policy can only deny | Early | opt-in |
| Prompt-injection tripwire on arguments | Early | a regex that blocks auto-approval, not a defence |
| Hash-chained traces, HMAC-signed, verifiable offline | Stable | [Trace integrity](../mesh7/trace-integrity.md) |
| Chain of authority per call (`/traces/{id}/why`) | Stable | |
| OpenTelemetry export, Prometheus metrics | Stable | [Observability](../mesh7/otel.md) |
| Conditions on JWT claims | Planned | |
| Semantic conditions beyond text matching | Planned | |

## flux7-memory: precedents and agent memory

v0.5.1 · Go · [overview](../mem7/index.md)

| Capability | State | Details |
|---|---|---|
| Store and search memories over MCP, HTTP and Python | Stable | [API reference](../mem7/api-reference.md) |
| Hybrid search: keyword, vector, LLM reranking | Stable | 71 % on the LoCoMo benchmark |
| Markdown files as source of truth, index rebuildable | Stable | |
| Human approvals kept as facts, used as precedents by the mesh | Stable | [Memory integration](../mesh7/mem7-auto-approve.md) |
| Access control per fact | Not planned | one token per instance; who may read or write what is a mesh policy on the `memory.*` tools |

## flux7-supervisor: automated evaluation

v0.1.0 · Python · [overview](../sup7/index.md) · install from GitHub, not on PyPI

| Capability | State | Details |
|---|---|---|
| Rules before any model call | Stable | [Configuration](../sup7/configuration.md) |
| Decision model asked typed questions, decision taken in code | Early | in production since 2026-09-29, [Jev and question sets](../sup7/jev.md) |
| Provider chain with circuit breaker (Jev, Ollama, Anthropic) | Early | |
| Jev's questions on a local model (Ollama System One, `nimble`), offline | Early | [Measuring](../sup7/measuring.md) |
| Question sets in YAML, extensible per business | Early | |
| Bench: case sets, free recompute, paid replay | Early | [Measuring](../sup7/measuring.md) |
| Judge one call on demand (`POST /evaluate`), without the mesh | Early | |
| Claude Code as reviewer, with codebase context | Planned | [Claude Code callback](../sup7/claude-code-callback.md) |

Thresholds were measured on about a thousand calls from one team, with no
danger approved; the boundary cases were written by the same people who wrote
the questions. Each deployment should measure its own.

## flux7-console: dashboard and approval UI

Next.js · [overview](../console/index.md)

| Capability | State | Details |
|---|---|---|
| Approval queue in the browser, with who settled what and why | Stable | |
| Traces with chain of authority and integrity badge | Stable | |
| Tools catalogue, per-agent decision editable | Early | writes the agent's policy file through the mesh |
| Supervisor state, YAML editing, bench runs | Early | |
| Memory browser, sessions, OTLP spans | Early | |
| Governance scoring, agent lifecycle, dependency graph | Planned | |

The console has no login of its own: run it on a private network or behind
your SSO proxy.

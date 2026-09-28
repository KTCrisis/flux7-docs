# Reference

Commands, flags, HTTP API and repository layout.

## Commands & flags

## `mesh7` (main binary)

```bash
mesh7 [flags]                           # run proxy (HTTP or MCP mode)
mesh7 serve [flags]                     # run as persistent daemon
mesh7 discover [flags]                  # discover tools + generate policy
mesh7 trace verify [--json] <file>...   # check the trace hash chain (oldest file first)
mesh7 --version                         # print version
```

| Flag | Default | Description |
|------|---------|-------------|
| `--config` | `config.yaml` | Path to YAML config |
| `--openapi` | | OpenAPI spec URL (ephemeral, for quick tests) |
| `--backend` | | Backend base URL override |
| `--port` | from config or `9090` | Port override |
| `--mcp` | `false` | MCP mode (stdio JSON-RPC — auto-proxies to daemon if running) |
| `--mcp-agent` | `claude` | Agent ID for MCP-mode policy evaluation |
| `--mcp-session-id` | auto-generated | Session ID for MCP traces |

`serve` flags: `--config <path>`, `--port <port>`. Runs as a persistent HTTP daemon. MCP clients auto-proxy to it via `--mcp`.

`discover` flags: `--openapi <url>`, `--config <path>`, `--generate-policy`, `--backend <url>`. The draft allows named reads and asks for everything else, each tool commented with its classification: see [Tool Classification](tool-classification.md).

`trace verify` flags: `--json` (machine-readable report), `--key-env <VAR>` (default `MESH_TRACE_KEY`). Exit 0 when the chain holds, 1 at the first break, 2 on usage or I/O error. See [Trace Integrity](trace-integrity.md).

### Environment

| Variable | Effect |
|----------|--------|
| `MESH_ADMIN_TOKEN` | Bearer token for the control plane (overrides `auth.admin_token`) |
| `MESH_TRACE_KEY` | HMAC key for the trace hash chain; without it the chain is plain SHA-256 |
| `MESH_AGENT_TOKEN` | In `--mcp` auto-proxy mode, a pre-issued JWT sent as `Authorization: Bearer <token>` to the daemon instead of the legacy `agent:<id>` form. Required when the daemon enforces JWT without `allow_legacy` |

## `mesh` (approval CLI)

```bash
mesh pending                    # list pending approvals
mesh show <id>                  # full details
mesh approve <id>               # approve
mesh approve <id> --grant 1h    # approve and open a temporal grant on the same tool
mesh approve <id> --grant 30m --tools "filesystem.*"   # widen the grant explicitly
mesh deny <id>                  # deny
mesh watch                      # interactive poll + prompt: [a]pprove / [g]rant / [d]eny / [s]kip
```

`--grant <duration>` opens a grant that records the approval as its origin, so later calls it authorises trace back to this decision (`GET /traces/{id}/why`). Without `--tools` the grant covers the exact tool approved, never a glob. In `watch`, `[g]` approves and grants for `MESH_GRANT_DURATION`.

| Variable | Default | Effect |
|----------|---------|--------|
| `MESH_URL` | `http://localhost:9090` | Mesh to talk to |
| `MESH_GRANT_DURATION` | `1h` | Grant length used by `[g]` in `watch` |

The `mesh` CLI sends no `Authorization` header, so it reaches the control plane only on loopback with no `admin_token` set. Against a token-protected mesh, use the HTTP API with `Authorization: Bearer $MESH_ADMIN_TOKEN`.

---


## API

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/decide` | Evaluate policy without executing (returns allow/deny/human_approval) |
| `POST` | `/tool/{name}` | Proxy a tool call through policy |
| `POST` | `/mcp` | MCP Streamable HTTP transport (JSON-RPC) |
| `DELETE` | `/mcp` | Terminate MCP HTTP session |
| `GET` | `/tools` | List all registered tools, each with its `classification` (family, access, reasons) |
| `GET` | `/mcp-servers` | List connected MCP servers |
| `GET` | `/traces` | Query traces (`?agent=...&tool=...`) |
| `GET` | `/traces/{id}/why` | Chain of authority of a call, oldest first (`?depth=10`, max 50) |
| `GET` | `/traces/verify` | Verify the trace file's hash chain with the store's key; a break is still a `200` |
| `GET` | `/sessions` | List sessions (id, agent, event count, timespan) |
| `GET` | `/sessions/{id}` | Session detail |
| `GET` | `/otel-traces` | OTLP JSON spans (`?agent=...&tool=...&limit=...`) |
| `GET` | `/approvals` | List approvals (`?status=pending&tool=filesystem.*`) |
| `GET` | `/approvals/{id}` | Approval detail with context |
| `POST` | `/approvals/{id}/approve` | Approve (optional: reasoning, confidence) |
| `POST` | `/approvals/{id}/deny` | Deny (optional: reasoning, confidence) |
| `GET` | `/policies` | List all policies (sorted by specificity) |
| `PUT` | `/policies/{agent}/tools/{tool}` | Set one tool's action for one agent in its policy file (`allow`, `deny`, `human_approval`, `inherit`); see [Tool Classification](tool-classification.md#changing-one-tools-action) |
| `GET` | `/tools/decisions` | What the policy decides for every tool, for `?agent=<id>`, before any call |
| `GET` | `/grants` | List active grants |
| `POST` | `/grants` | Create a grant |
| `DELETE` | `/grants/{id}` | Revoke a grant |
| `GET` | `/metrics` | Prometheus counters (mem7 decision writes) |
| `GET` | `/health` | Health check and stats |
| `GET` | `/version` | Version info |

Data plane (never gated by `admin_token`): `/decide`, `/tool/{name}`, `/mcp`, `/tools`, `/mcp-servers`, `/health`, `/version`. Every other route is control plane: it requires `Authorization: Bearer <admin_token>`, or a loopback caller when no token is set. See [Control-plane auth](control-plane-auth.md).

---


## Project structure

```
flux7-mesh/
├── cmd/
│   ├── mesh7/             # Main binary (entry point, wiring, auto-proxy, trace verify)
│   └── mesh/              # Approval CLI (pending/approve/deny/watch)
├── auth/                  # Agent identity: JWT validation, JWKS cache, legacy agent: form
├── config/                # YAML config parsing + validation
├── registry/              # Tool registry (OpenAPI + MCP + CLI imports)
├── policy/                # Rule evaluation (globs, conditions, fail-closed)
├── proxy/                 # HTTP handler (auth → rate limit → policy → forward → trace)
├── mcp/                   # MCP client/server/transport (stdio + SSE + streamable HTTP)
├── approval/              # Channel-based approval store with timeout
├── grant/                 # Temporal grants (TTL-based sudo)
├── storage/               # SQLite durable state (approvals, grants survive restarts)
├── ratelimit/             # Sliding window + loop detection
├── supervisor/            # Content isolation + injection detection
├── exec/                  # Secure CLI execution (no shell, arg validation)
├── trace/                 # In-memory + JSONL (hash chain) + OTEL export
├── internal/              # Shared helpers: glob matching, SSRF guard
├── policies/              # Per-agent policy files (used with policy_dir)
├── sdk/python/            # Python SDK (pip install flux7-mesh)
├── plugin/                # Claude Code plugin skills (approve, catalog, setup, status, traces)
├── contrib/systemd/       # mesh7.service unit for daemon mode
├── examples/              # Example configs (filesystem, petstore, travel, langchain)
└── docs/                  # CLI tools guide, OTEL guide, supervisor protocol
```


## Tests

```bash
go test ./...              # all tests
go test ./... -race        # with race detector
```

412 Go test functions across 17 packages + 77 Python SDK tests, covering config parsing, policy evaluation (string operators, dispatcher floor), JWT auth validation, HTTP/MCP proxy flows, MCP session binding, approval lifecycle, grant lineage, mem7 auto-approve, supervisor agent whitelist, CLI execution security, SSRF guard, rate limiting, tracing and the hash chain, OTEL export, supervisor content isolation, injection detection, durable state persistence, auto-proxy daemon detection, and the `mesh7-hook` PreToolUse hook.

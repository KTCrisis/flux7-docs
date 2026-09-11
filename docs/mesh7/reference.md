# Reference

Commands, flags, HTTP API and repository layout.

## Commands & flags

## `mesh7` (main binary)

```bash
mesh7 [flags]                           # run proxy (HTTP or MCP mode)
mesh7 serve [flags]                     # run as persistent daemon
mesh7 discover [flags]                       # discover tools + generate policy
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

`discover` flags: `--openapi <url>`, `--config <path>`, `--generate-policy`, `--backend <url>`.

## `mesh` (approval CLI)

```bash
mesh pending                    # list pending approvals
mesh show <id>                  # full details
mesh approve <id>               # approve
mesh deny <id>                  # deny
mesh watch                      # interactive poll + prompt
```

Set `MESH_URL` to override the default `http://localhost:9090`.

---


## API

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/decide` | Evaluate policy without executing (returns allow/deny/human_approval) |
| `POST` | `/tool/{name}` | Proxy a tool call through policy |
| `POST` | `/mcp` | MCP Streamable HTTP transport (JSON-RPC) |
| `DELETE` | `/mcp` | Terminate MCP HTTP session |
| `GET` | `/tools` | List all registered tools |
| `GET` | `/mcp-servers` | List connected MCP servers |
| `GET` | `/traces` | Query traces (`?agent=...&tool=...`) |
| `GET` | `/sessions` | List sessions (id, agent, event count, timespan) |
| `GET` | `/sessions/{id}` | Session detail |
| `GET` | `/otel-traces` | OTLP JSON spans (`?agent=...&tool=...&limit=...`) |
| `GET` | `/approvals` | List approvals (`?status=pending&tool=filesystem.*`) |
| `GET` | `/approvals/{id}` | Approval detail with context |
| `POST` | `/approvals/{id}/approve` | Approve (optional: reasoning, confidence) |
| `POST` | `/approvals/{id}/deny` | Deny (optional: reasoning, confidence) |
| `GET` | `/policies` | List all policies (sorted by specificity) |
| `GET` | `/grants` | List active grants |
| `POST` | `/grants` | Create a grant |
| `DELETE` | `/grants/{id}` | Revoke a grant |
| `GET` | `/health` | Health check and stats |
| `GET` | `/version` | Version info |

---


## Project structure

```
flux7-mesh/
├── cmd/
│   ├── mesh7/        # Main binary (entry point, wiring)
│   └── mesh/              # Approval CLI (pending/approve/deny/watch)
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
├── trace/                 # In-memory + JSONL + OTEL export
├── policies/              # Per-agent policy files (used with policy_dir)
├── sdk/python/            # Python SDK (pip install flux7-mesh)
├── examples/              # Example configs (filesystem, petstore, travel, langchain)
└── docs/                  # CLI tools guide, OTEL guide, supervisor protocol
```


## Tests

```bash
go test ./...              # all tests
go test ./... -race        # with race detector
```

281 Go tests across 16 packages + 49 Python SDK tests, covering config parsing, policy evaluation, JWT auth validation, HTTP/MCP proxy flows, approval lifecycle, mem7 auto-approve, supervisor agent whitelist, CLI execution security, rate limiting, tracing, OTEL export, supervisor content isolation, injection detection, durable state persistence, and auto-proxy daemon detection.

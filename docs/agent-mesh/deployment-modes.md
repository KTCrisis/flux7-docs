# Deployment Modes

Agent-mesh supports different configurations depending on who connects and how.

## Configuration matrix

| # | Setup | Transport | Who launches mesh | Supervisor | Status |
|---|-------|-----------|-------------------|------------|--------|
| **1** | Solo dev + Claude/Cursor | MCP stdio | Claude spawns it | None | Works |
| **2** | Solo dev + Claude + supervisor | MCP stdio + HTTP | Claude spawns it | Passive (poll :9090) | Works while Claude runs |
| **3** | Supervisor standalone (no Claude) | HTTP | Supervisor spawns it | Active | Works |
| **4** | External agent (LangChain, script) | HTTP | Manual or supervisor | Optional | Works |
| **5** | Claude + external agent | MCP stdio + HTTP | Claude spawns it | Optional | Works |
| **6** | Claude + supervisor (active spawn) | MCP stdio + HTTP | Both try to spawn | Active | Works (auto-proxy) |
| **7** | 2 Claude sessions | MCP stdio × 2 | Both spawn | - | Works (auto-proxy) |

Configs 1–5 work natively. Configs 6–7 are solved by auto-proxy (v0.9.2) — the second instance detects the running daemon and becomes a thin stdio→HTTP proxy instead of crashing.

---

## Config 1: Solo dev + Claude (most common)

The default. 90% of users. Zero setup.

```
Claude Code ──stdio──> agent-mesh ──> filesystem, gmail, ollama...
                           │
                      :9090 HTTP (background, for mesh CLI / traces)
```

Claude launches agent-mesh as an MCP subprocess. Agent-mesh launches upstream MCP servers, applies policies, records traces. When Claude quits, everything stops cleanly.

**Setup:** Just add agent-mesh as an MCP server in Claude Code:

```bash
claude mcp add agent-mesh -- agent-mesh --mcp --config config.yaml
```

## Config 2: Solo dev + Claude + supervisor (passive)

Claude manages agent-mesh. Two layers of auto-resolve handle routine approvals before they reach a human.

```
Claude Code ──stdio──> agent-mesh :9090 ──> tools
                           │
                    Level 1: built-in (mem7 lookup, ~100ms)
                    ├── 3+ past approvals → auto-approve
                    └── else → escalate to Level 1+
                           │
                    Level 1+: supervisor (poll GET /approvals)
                    ├── rules → approve/deny (0ms)
                    └── ollama → evaluate (~20s)
```

The built-in auto-approve (Level 1) fires before the approval queue — routine patterns never block. If it can't resolve, the external supervisor (Level 1+) evaluates with rules and LLM. If both escalate, the human decides.

**Setup:**

```bash
# Terminal 1: Claude (launches agent-mesh automatically)
claude

# Terminal 2: Supervisor
cd ~/agent7
python -m backend.app.services.supervisor --config supervisor.local.yaml
```

With `supervisor.enabled: true` in the agent-mesh config, `approval.resolve` and `approval.pending` tools are hidden from Claude. Tool calls block until the supervisor resolves them.

**Supervisor config:**

```yaml
supervisor:
  mesh_url: http://localhost:9090
  mesh_process:
    enabled: false    # Claude manages the lifecycle
  ollama:
    enabled: true
    model: qwen3:14b
  rules:
    - name: project-scope
      condition: "params.path starts_with /home/user"
      action: approve
      confidence: 0.95
```

**Limitation:** When Claude quits, agent-mesh dies. The supervisor retries every `poll_interval` until Claude starts again.

## Config 3: Supervisor standalone (no Claude)

For pipelines, overnight runs, CI/CD, batch jobs. No human in the loop — the supervisor manages everything.

```
supervisor (always alive)
  │
  ├── spawn/restart ──> agent-mesh :9090 ──> tools
  │
  ├── poll → evaluate → resolve
  └── store decisions in memory-mcp
```

**Setup:**

```bash
cd ~/agent7
python -m backend.app.services.supervisor --config supervisor.yaml
```

With `mesh_process.enabled: true`, the supervisor spawns agent-mesh on startup, monitors health, and restarts it on crash.

```yaml
supervisor:
  mesh_url: http://localhost:9090
  mesh_process:
    enabled: true
    command: agent-mesh
    config: /path/to/config.yaml
```

External agents connect via HTTP:

```bash
curl -X POST http://localhost:9090/tool/filesystem.write_file \
  -H "Authorization: Bearer agent:my-script" \
  -d '{"params":{"path":"/tmp/output.txt","content":"hello"}}'
```

## Config 4: External agent only

Standard HTTP proxy mode. No MCP, no supervisor.

```
agent-mesh :9090 ──> tools
     │
Agent (HTTP) ──────┘
```

**Setup:**

```bash
agent-mesh --config config.yaml
```

Any HTTP client can call `POST /tool/{name}`, query traces, manage approvals. Works with LangChain, CrewAI, custom scripts, cron jobs.

## Config 5: Claude + external agent (the real mesh)

Claude and external agents share the same agent-mesh instance. One set of policies, one trace store, one approval queue.

```
Claude Code ──stdio──> agent-mesh :9090 ──> tools
                           │
Agent B ─────────HTTP──────┘
```

**This works today.** Claude spawns agent-mesh, the external agent connects via HTTP to `:9090`. Both are governed by the same policies.

Add a supervisor and you get Config 2 with extra agents — everything goes through one mesh.

---

## Config 8: Anthropic Managed Agents (cloud, MCP Streamable HTTP)

Cloud-hosted agents connect to your agent-mesh over the internet via MCP Streamable HTTP.

```
Anthropic cloud
├── Managed Agent (coordinator)
│   ├── sub-agent: reviewer
│   ├── sub-agent: tester
│   └── all use mcp_toolset "mesh"
│
└── MCP connector ── POST /mcp ──> agent-mesh (your server, public URL)
                                       │
                                  policies, traces, mem7
                                       │
                                  upstream MCP servers
```

**Setup:**

```python
# Anthropic SDK (Python)
agent = client.beta.agents.create(
    name="Governed Assistant",
    model="claude-opus-4-7",
    mcp_servers=[{
        "type": "url",
        "name": "mesh",
        "url": "https://mesh.example.com/mcp",
    }],
    tools=[
        {"type": "agent_toolset_20260401"},
        {"type": "mcp_toolset", "mcp_server_name": "mesh",
         "default_config": {"permission_policy": {"type": "always_allow"}}},
    ],
)
```

Auth via vault (static bearer):

```python
vault = client.beta.vaults.create(display_name="mesh-credentials")
client.beta.vaults.credentials.create(
    vault_id=vault.id,
    display_name="agent-mesh token",
    auth={
        "type": "static_bearer",
        "mcp_server_url": "https://mesh.example.com/mcp",
        "token": "agent:my-managed-agent",
    },
)
```

agent-mesh extracts the agent ID from `Authorization: Bearer agent:<id>` and applies per-agent policies.

**Networking:** agent-mesh must be accessible from Anthropic's cloud. Options:
- Dev: Tailscale funnel or ngrok → `localhost:9090`
- Prod: deploy agent-mesh on a VPS or cloud host

**Permission policies:** Set `always_allow` on the Managed Agent side — let agent-mesh handle governance. Double-layer approval (Managed Agents `always_ask` + agent-mesh `human_approval`) works but adds friction.

---

## Configs solved by auto-proxy (v0.9.2)

Since v0.9.2, when agent-mesh starts in `--mcp` mode, it probes `GET /health` on the configured port before doing anything else. If a daemon (or another agent-mesh instance) is already running, it skips all heavy initialization and becomes a thin stdio→HTTP proxy — forwarding JSON-RPC from stdin to `POST /mcp` on the running instance. Zero config change, same `claude mcp add` command.

### Config 6: Claude + supervisor (active spawn)

```
Claude ──stdio──> agent-mesh (auto-proxy) ──POST /mcp──> agent-mesh :9090 (daemon)
supervisor ──spawn──> agent-mesh :9090 (daemon)
```

The supervisor starts the daemon. Claude's MCP subprocess detects it and proxies to it. No port conflict. Traces, approvals, and grants are all on the daemon — one unified state.

### Config 7: Two Claude sessions

```
Claude session 1 ──stdio──> agent-mesh :9090 (full instance, first to start)
Claude session 2 ──stdio──> agent-mesh (auto-proxy) ──POST /mcp──> :9090
```

The first session starts normally. The second detects the running instance and proxies. Both sessions share the same policy engine, approval queue, trace store, and durable state.

---

## Decision flow: which config to use

```
Do you use Claude/Cursor?
  │
  ├─ Yes, just Claude ──────────────────────────→ Config 1 (embedded)
  │
  ├─ Yes, Claude + auto-approve ────────────────→ Config 2 (passive supervisor)
  │
  ├─ Yes, Claude + external agents ─────────────→ Config 5 (shared mesh)
  │
  └─ No
       │
       ├─ Want auto-approve / overnight runs ───→ Config 3 (supervisor standalone)
       │
       ├─ Just HTTP proxy ─────────────────────→ Config 4 (standalone)
       │
       └─ Anthropic Managed Agents (cloud) ─→ Config 8 (MCP Streamable HTTP)
```

## Component lifecycle

| Component | Who starts it | Who stops it | Persists across sessions |
|-----------|--------------|-------------|------------------------|
| **Ollama** | System daemon | System | Yes |
| **agent-mesh** | Claude (config 1/2/5) or supervisor (config 3) | Dies with parent | Durable state (SQLite) survives restarts |
| **Upstream MCP servers** | agent-mesh (subprocesses) | Die with agent-mesh | No |
| **Supervisor** | User (terminal) | User (Ctrl+C) | Yes (as long as terminal lives) |
| **Claude Code** | User | User | No |

## Next: daemon mode

The auto-proxy (v0.9.2) solves port conflicts — but the first instance is still ephemeral (tied to whoever started it). The next step is a proper daemon:

```
                    agent-mesh serve (daemon, persistent)
                    ┌─────────────────────────────────────┐
Claude ──proxy───>  │                                     │──> tools
Agent B ───HTTP───> │  registry · policy · approval       │
Agent C ───HTTP───> │  trace · grants · rate limiting     │
                    └──────────────┬──────────────────────┘
                                   │
                            supervisor (poll)
```

```bash
# Start the daemon
agent-mesh serve --config config.yaml

# Claude auto-proxies to it (same command as before)
claude mcp add agent-mesh -- agent-mesh --mcp --config config.yaml
```

- **`agent-mesh serve`** — runs as a persistent daemon (HTTP + manages upstream MCP servers, survives client disconnects)
- **Auto-proxy** handles the MCP side automatically — Claude's `--mcp` instance detects the daemon and proxies to it. No explicit `connect` subcommand needed.

This solves Config 2's limitation (mesh dies with Claude). The supervisor manages the daemon lifecycle. Durable state (v0.9.2) ensures approvals and grants survive daemon restarts.

| Feature | Status |
|---------|--------|
| Config 1: Embedded MCP | Done |
| Config 2: Passive supervisor | Done |
| Config 3: Active supervisor | Done |
| Config 4: Standalone HTTP | Done |
| Config 5: Shared mesh | Done |
| Config 8: Managed Agents (MCP Streamable HTTP) | Done (v0.9.0) |
| `supervisor.enabled` (hide approval tools) | Done |
| Durable state (SQLite) | Done (v0.9.2) |
| Auto-proxy (stdio→HTTP on daemon detect) | Done (v0.9.2) |
| `agent-mesh serve` (daemon) | Done (v0.9.4) |

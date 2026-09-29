# Getting Started

Each flux7 project works independently. Add layers as you need them.

## flux7-mesh alone

Policy enforcement in 5 minutes. No other component required.

### Install

```bash
curl -fsSL https://raw.githubusercontent.com/KTCrisis/flux7-mesh/main/install.sh | sh
```

Installs `mesh7` (the proxy) and `mesh` (the approval CLI) in `~/.local/bin`. With Go: `go install github.com/KTCrisis/flux7-mesh/cmd/mesh7@latest` (and `.../cmd/mesh@latest`). Other options and running it as a service: [flux7-mesh getting started](mesh7/getting-started.md).

### Write a policy

```yaml title="config.yaml"
port: 9090
trace_file: traces.jsonl

mcp_servers:
  - name: filesystem
    transport: stdio
    command: npx
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/home/me/projects"]

policies:
  - name: claude
    agent: "claude"
    rules:
      - tools: ["filesystem.read_*", "filesystem.list_*"]
        action: allow
      - tools: ["filesystem.write_file", "filesystem.edit_file"]
        action: human_approval

  - name: default
    agent: "*"
    rules:
      - tools: ["*"]
        action: deny
```

A policy names the agents it applies to; its rules are evaluated first match wins, and anything no rule matches is denied.

### Run

```bash
mesh7 serve --config config.yaml
```

Agents connect over MCP (`mesh7 --mcp --mcp-agent claude` in the agent's MCP config, which proxies to the daemon) or HTTP on `:9090`. Reads are allowed, writes wait for a human, everything else is denied. Every call is traced to `traces.jsonl`, [hash-chained](mesh7/trace-integrity.md).

---

## Add flux7-memory

Persistent memory + auto-approve from decision history.

### Install

```bash
go install github.com/KTCrisis/flux7-memory/cmd/mem7@latest
```

### Start the daemon

```bash
MEM7_TOKEN=mem7_secret123 mem7 serve --listen :9070
```

### Connect the mesh to memory

```yaml title="config.yaml"
memory:
  url: http://localhost:9070
  token: mem7_secret123

mcp_servers:
  - name: memory                      # optional: memory tools for the agents too
    transport: streamable-http
    url: http://localhost:9070/mcp
    headers:
      Authorization: "Bearer mem7_secret123"
```

With a `memory:` block, the mesh writes every approval decision to mem7, and auto-approves a tool that reads once a human has approved it 3 times for this agent, with no refusal (`supervisor.min_approvals`; `supervisor.auto_approve: false` turns it off; writes always ask again unless `supervisor.auto_approve_writes` is set). Arguments that look like prompt injection are never auto-approved.

### Use memory from Python

```bash
pip install flux7-memory
```

```python
from mem7 import Mem7

m = Mem7("http://localhost:9070", token="mem7_secret123")

# Store a decision
m.store("deploy.approval", "approved by ops lead",
        tags=["decision"], agent="supervisor")

# Search later
for mem in m.context("deployment approval", limit=5):
    print(f"{mem.key}: {mem.value}")
```

---

## Add flux7-supervisor

Automated evaluation for pending approvals. Reduces approval fatigue.

### Install

Not published on PyPI yet: install from the repository.

```bash
pip install "git+https://github.com/KTCrisis/flux7-supervisor"
# with the Anthropic provider
pip install "flux7-supervisor[anthropic] @ git+https://github.com/KTCrisis/flux7-supervisor"
```

### Configure and run

```yaml title="sup7.yaml"
mesh:
  url: http://localhost:9090
  agent_id: supervisor

evaluator:
  provider: ollama              # ollama | anthropic | claude-code | jev
  model: qwen3:14b
  url: http://localhost:11434
  confidence_threshold: 0.8
```

```bash
sup7 -c sup7.yaml status        # check mesh (and memory) connectivity
sup7 -c sup7.yaml start         # start the poll loop
```

The supervisor polls the mesh for pending approvals, applies its rules, and asks an LLM about the ambiguous cases. Providers can be chained with a circuit breaker; see [configuration](sup7/configuration.md).

---

## Add flux7-console

Human oversight dashboard: approvals, traces with their chain of authority, sessions, grants, policies, memory browser.

```bash
git clone https://github.com/KTCrisis/flux7-console
cd flux7-console/frontend
npm install
npm run dev                     # http://localhost:3000
```

It reads the mesh on `localhost:9090` and mem7 on `localhost:9070`; override with `MESH_URL` / `MEM7_URL` (and `MESH_ADMIN_TOKEN` for a remote mesh) in `frontend/.env.local`.

---

## The full chain

```
Agent → mesh7 (policy) → mem7 (history) → sup7 (LLM eval) → console (human)
  L0                       L1                L1+                L2
```

Each layer reduces the load on the next. Most tool calls resolve at L0 (policy) or L1 (memory). The supervisor handles novel cases. Humans only see what truly requires judgment.

| Layer | Component | Latency | What it does |
|-------|-----------|---------|--------------|
| L0 | flux7-mesh | <1ms | Policy match — allow, deny, or escalate |
| L1 | flux7-memory | ~100ms | Check decision history — a read approved 3+ times by a human, never refused → allow |
| L1+ | flux7-supervisor | ~0-500ms | Rules, then a decision model — approve, deny, or escalate |
| L2 | flux7-console | minutes | Human reviews in web UI |

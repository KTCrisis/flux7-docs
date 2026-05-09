# Getting Started

Each flux7 project works independently. Add layers as you need them.

## flux7-mesh alone

Policy enforcement in 5 minutes. No other component required.

### Install

```bash
go install github.com/KTCrisis/flux7-mesh/cmd/mesh7@latest
```

### Write a policy

```yaml title="config.yaml"
mcp_servers:
  - name: filesystem
    transport: stdio
    command: npx
    args: ["-y", "@anthropic/mcp-filesystem"]

policies:
  - tools: ["filesystem.read_file"]
    action: allow
  - tools: ["filesystem.write_file"]
    action: human_approval
  - tools: ["*"]
    action: deny
```

### Run

```bash
mesh7 --config config.yaml
```

Agents connect via MCP. Reads are allowed, writes require approval, everything else is denied. Traces are logged to `traces.jsonl`.

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

### Add memory to mesh config

```yaml title="config.yaml"
mcp_servers:
  - name: filesystem
    transport: stdio
    command: npx
    args: ["-y", "@anthropic/mcp-filesystem"]
  - name: memory
    transport: stdio
    command: mem7
    env:
      MEM7_DIR: /home/user/.mem7

policies:
  - tools: ["filesystem.read_file"]
    action: allow
  - tools: ["filesystem.write_file"]
    action: human_approval
  - tools: ["memory.*"]
    action: allow
  - tools: ["*"]
    action: deny
```

The mesh automatically checks memory : if a tool call has been approved 3+ times before, it's auto-approved. Decisions are stored as facts for next time.

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

```bash
pip install flux7-supervisor
```

### Run

```bash
sup7 --mesh http://localhost:8080 --provider ollama
```

The supervisor polls the mesh for pending approvals, applies rules, and evaluates ambiguous cases via LLM. Three providers supported : Ollama, Anthropic API, Claude Code MCP.

```bash
# With Anthropic API
sup7 --mesh http://localhost:8080 --provider anthropic --api-key $ANTHROPIC_API_KEY

# With local Ollama
sup7 --mesh http://localhost:8080 --provider ollama --model gemma4:e4b
```

---

## Add flux7-console

Human oversight dashboard. Approval UI, trace viewer, memory browser.

```bash
docker run -p 3000:3000 ktcrisis/flux7-console
```

Open `http://localhost:3000` — the console connects to mesh and memory to show the full governance picture.

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
| L1 | flux7-memory | ~100ms | Check decision history — 3+ past approvals → allow |
| L1+ | flux7-supervisor | 2-20s | Rules + LLM evaluation — approve, deny, or escalate |
| L2 | flux7-console | minutes | Human reviews in web UI |

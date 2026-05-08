# Configuration

sup7 reads a YAML config file (default: `sup7.yaml`).

## Full reference

```yaml
# Connection to flux7-mesh
mesh:
  url: http://localhost:9090
  agent_id: supervisor

# Connection to flux7-memory (optional)
memory:
  url: http://localhost:9070
  token: ""                    # bearer token if configured
  enabled: true
  store_decisions: true        # write each decision to mem7
  recall_on_start: true        # recall recent decisions on startup
  recall_limit: 20
  tags: [supervisor, decision] # tags applied to stored decisions

# LLM evaluation provider
evaluator:
  provider: ollama             # ollama | anthropic | claude-code
  model: qwen3:14b            # model name (ignored for claude-code)
  url: http://localhost:11434  # Ollama URL (ignored for anthropic/claude-code)
  timeout: 30                  # LLM call timeout in seconds
  callback_timeout: 120        # Claude Code MCP callback timeout
  confidence_threshold: 0.8    # below this → escalate to human

# MCP server for Claude Code callback
mcp_server:
  enabled: true                # expose sup7.pending + sup7.verdict
  transport: stdio             # stdio or http
  port: 9095                   # only used with http transport

# Poll loop
poll:
  interval: 2s                 # supports: ms, s, m, h
  tool_scopes: []              # tool glob filters, empty = all tools

# Evaluation rules (first-match-wins)
rules:
  - name: safe-reads
    condition: "tool contains read"
    action: approve
    confidence: 0.95

  - name: project-writes
    condition: "params.path starts_with project_dir"
    action: approve
    confidence: 0.9

  - name: injection-risk
    condition: "injection_risk == true"
    action: escalate
    confidence: 1.0

# Directories considered "safe" for project_dir conditions
project_dirs:
  - /home/user/my-project

# Decision log file (JSONL)
decision_log: sup7-decisions.jsonl
```

## Rule conditions

Conditions follow the format `<field> <operator> <value>` :

| Operator | Example | Meaning |
|----------|---------|---------|
| `contains` | `tool contains read` | tool name includes "read" |
| `equals` / `==` | `tool == filesystem.write_file` | exact match |
| `not_equals` / `!=` | `tool != filesystem.delete` | not equal |
| `starts_with` | `params.path starts_with /home` | string prefix |

Special value `project_dir` checks against all entries in `project_dirs` :

```yaml
- name: project-writes
  condition: "params.path starts_with project_dir"
  action: approve
```

A rule without a `condition` is a catch-all. If the last rule has a condition, sup7 automatically appends a catch-all that escalates.

## Rule actions

| Action | Effect |
|--------|--------|
| `approve` | Resolve the approval as approved |
| `deny` | Resolve the approval as denied |
| `escalate` | Leave for human (L2) or Claude Code |

Each rule has a `confidence` score (0.0–1.0). If the confidence is below `evaluator.confidence_threshold`, the action is overridden to `escalate`.

## Provider-specific configuration

### Ollama

```yaml
evaluator:
  provider: ollama
  model: qwen3:14b
  url: http://localhost:11434
  timeout: 30
```

Requires Ollama running locally. The model receives a structured prompt with tool name, params, recent traces, and active grants, and must respond in format :

```
DECISION: APPROVE | CONFIDENCE: 0.95 | REASONING: routine file read
```

### Anthropic

```yaml
evaluator:
  provider: anthropic
  model: claude-sonnet-4-6
```

Requires `ANTHROPIC_API_KEY` environment variable. Install with `pip install 'flux7-supervisor[anthropic]'`.

### Claude Code

```yaml
evaluator:
  provider: claude-code
  callback_timeout: 120

mcp_server:
  enabled: true
  transport: stdio
```

See [Claude Code Callback](claude-code-callback.md) for the full architecture.

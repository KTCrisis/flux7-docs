# Configuration

sup7 reads a YAML config file, passed with the top-level `-c` option (default: `sup7.yaml`):

```bash
sup7 -c sup7.yaml start
```

The keys may also sit under a top-level `supervisor:` block.

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
  enabled: false               # default false: set true to store and recall decisions
  store_decisions: true        # write each decision to mem7
  recall_on_start: true        # recall recent decisions on startup
  recall_limit: 20
  tags: [supervisor, decision] # tags applied to stored decisions

# LLM evaluation provider
evaluator:
  provider: ollama             # ollama | anthropic | claude-code | jev
  model: qwen3:14b            # model name (ignored for claude-code and jev)
  url: http://localhost:11434  # Ollama URL (ignored for anthropic/claude-code/jev)
  timeout: 30                  # LLM call timeout in seconds
  callback_timeout: 120        # Claude Code MCP callback timeout
  confidence_threshold: 0.8    # below this → escalate to human
  system_prompt: "..."         # ollama and anthropic; the default asks for one DECISION line
  jev: {}                      # Jev settings, see below
  chain: []                    # optional provider chain, see below
  breaker_failures: 3          # chain: consecutive failures before a provider is skipped
  breaker_cooldown: 300        # chain: seconds a tripped provider stays skipped

# HTTP admin API (used by flux7-console)
admin:
  enabled: false               # off by default
  host: 127.0.0.1              # loopback; set a token before binding elsewhere
  port: 9096
  token: ""                    # when set: "Authorization: Bearer <token>" on all routes but /health
  recent_decisions: 200        # decisions kept in memory for GET /decisions

# MCP server for Claude Code callback (not started in the current release)
mcp_server:
  enabled: true                # read: binds the evaluator to the MCP tools
  transport: stdio             # currently unused
  port: 9095                   # currently unused

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
    description: routine read   # optional, appended to the decision reasoning

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

Before any rule, an approval flagged with `injection_risk` by flux7-mesh is escalated directly.

A rule without a `condition` is a catch-all. If the last rule has a condition, sup7 automatically appends a catch-all that escalates.

## Rule actions

| Action | Effect |
|--------|--------|
| `approve` | Resolve the approval as approved |
| `deny` | Resolve the approval as denied |
| `escalate` | Leave for human (L2) or Claude Code |

Each rule has a `confidence` score (0.0–1.0, default 0.9). If the confidence is below `evaluator.confidence_threshold`, the action is overridden to `escalate`. The optional `description` is appended to the reasoning recorded with the decision.

When the matching rule is a catch-all with action `escalate` (including the automatic one), sup7 hands the approval to the configured provider or chain instead of escalating directly.

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

Requires `ANTHROPIC_API_KEY` environment variable. Install with `pip install "flux7-supervisor[anthropic] @ git+https://github.com/KTCrisis/flux7-supervisor"`.

### Jev (TypeSafe AI)

```yaml
evaluator:
  provider: jev
  confidence_threshold: 0.8
  jev:
    backend: cloudflare                # cloudflare (Workers AI, model typesafe/jev) | typesafe (model jev-latest)
    model: ""                          # empty: the backend's default model
    url: ""                            # empty: the backend's public endpoint
    api_key_env: CLOUDFLARE_API_TOKEN  # TYPESAFE_API_KEY with backend: typesafe
    account_id_env: CLOUDFLARE_ACCOUNT_ID  # cloudflare only
    injection_max: 0.5                 # above: escalate
    destructive_max: 0.2               # above: never auto-approve
    in_scope_min: 0.7                  # below: never auto-approve
    deny_min: 0.9                      # deny only when this probable
    redact_params: [content]           # parameter names never sent to the model
```

`api_key_env` and `account_id_env` name environment variables; the secrets themselves stay out of the file. Jev answers four typed questions about the pending call, each with a probability:

| Question | Type | Asks |
|----------|------|------|
| `decision` | choice | approve, escalate or deny |
| `destructive` | noul | deletes, overwrites or exfiltrates data, or changes permissions or secrets |
| `in_scope` | noul | consistent with the agent's recent activity |
| `injection` | noul | parameters carry instructions aimed at a model |

sup7 combines them in code, fail-closed: escalate when `injection` exceeds `injection_max`; deny only when deny is chosen with a probability of at least `deny_min`; approve only when approve is chosen, `destructive` is at most `destructive_max` and `in_scope` at least `in_scope_min` (then `confidence_threshold` applies); escalate everything else. A missing environment variable, a network or HTTP error, or an unreadable answer counts as a failure, which escalates (or moves to the next provider in a chain). The probabilities are written into the reasoning.

### Claude Code

```yaml
evaluator:
  provider: claude-code
  callback_timeout: 120

mcp_server:
  enabled: true
  transport: stdio
```

This provider is not functional in the current release: the MCP server is not started, so every evaluation waits for `callback_timeout` and then escalates. See [Claude Code Callback](claude-code-callback.md) for the intended design.

## Provider chain

```yaml
evaluator:
  confidence_threshold: 0.8
  breaker_failures: 3       # consecutive failures before a provider is skipped
  breaker_cooldown: 300     # seconds it stays skipped
  chain:
    - provider: jev         # fast typed decisions when available
      jev: { backend: cloudflare, api_key_env: CLOUDFLARE_WORKERS_AI_TOKEN }
    - provider: ollama      # local, free, works offline
      model: qwen3:14b
```

Each chain entry is a full `evaluator` block; `confidence_threshold`, `breaker_failures` and `breaker_cooldown` are read from the top level. When `chain` is non-empty it replaces the single `provider`.

Providers are tried in order and the first one that answers gives the verdict; an `escalate` verdict is an answer, not a failure. The next provider is tried only when the previous one fails (network error, HTTP error such as 402, timeout, unreadable answer). After `breaker_failures` consecutive failures a provider is skipped for `breaker_cooldown` seconds. If every provider fails, sup7 escalates to a human. The reasoning is prefixed with the provider that answered and those skipped or failed, e.g. `[ollama, jev skipped] ...`, and `GET /status` on the admin API reports each provider as `ok`, `failing` or `skipped`.

## Admin API

```yaml
admin:
  enabled: true
  host: 127.0.0.1
  port: 9096
  token: ""
```

Off by default. Routes: `GET /health` (no token), `GET /status`, `GET /config` (never secrets), `GET /decisions?limit=50` (1 to 500), `POST /pause`, `POST /resume`. See [the overview](index.md#admin-api).

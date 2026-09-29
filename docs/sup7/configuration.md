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
  provider: ollama             # ollama | anthropic | claude-code | jev | none (rules only)
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
  token: ""                    # when set: "Authorization: Bearer <token>" on all routes but /health;
                               # required to edit files and start evaluation runs
  recent_decisions: 200        # decisions kept in memory for GET /decisions

# MCP server for Claude Code callback (not started in the current release)
mcp_server:
  enabled: true                # read: binds the evaluator to the MCP tools
  transport: stdio             # currently unused
  port: 9095                   # currently unused

# Poll loop
poll:
  interval: 2s                 # supports: ms, s, m, h; 500ms with mesh7's approval.wait_seconds
  tool_scopes: []              # tool glob filters, empty = all tools

# Evaluation rules (first-match-wins)
rules:
  - name: safe-reads
    # exact names: "tool contains read" would also approve gmail_mark_as_read, which writes
    condition: "tool in filesystem.read_file, filesystem.read_text_file, filesystem.list_directory"
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

# Evaluation runs (see Measuring)
bench:
  dir: ~/.sup7/bench            # sets/*.jsonl (labelled cases), runs/<id>/ (results)
```

## Rule conditions

Conditions follow the format `<field> <operator> <value>` :

| Operator | Example | Meaning |
|----------|---------|---------|
| `in` | `tool in filesystem.read_file, filesystem.list_directory` | one of these exact names (comma-separated) |
| `contains` | `tool contains read` | substring: also matches `gmail_mark_as_read`, avoid it for approvals |
| `equals` / `==` | `tool == filesystem.write_file` | exact match |
| `not_equals` / `!=` | `tool != filesystem.delete` | not equal |
| `starts_with` | `params.path starts_with /home` | string prefix |

Special value `project_dir` checks against all entries in `project_dirs`. The path is normalised first (`..` resolved, `~` expanded) and must be a project dir or continue with a separator: `<project>/../.bashrc` and a sibling `<project>-backup/x` do not match, and a relative path never does.

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
  confidence_threshold: 0.6            # Jev's confidence is 1 - the highest danger (see below)
  jev:
    backend: cloudflare                # cloudflare (Workers AI, model typesafe/jev) | typesafe (model jev-latest)
    model: ""                          # empty: the backend's default model
    url: ""                            # empty: the backend's public endpoint; a local /v1/systemone runtime works too
    api_key_env: CLOUDFLARE_API_TOKEN  # TYPESAFE_API_KEY with backend: typesafe
    account_id_env: CLOUDFLARE_ACCOUNT_ID  # cloudflare only
    questions: []                      # question set files (globs allowed); empty = the shipped socle
    destructive_max: 0.2               # a danger above this blocks approval (default for danger questions)
    in_scope_min: 0.7                  # below: never auto-approve
    deny_min: 0.9                      # deny only when a danger is this probable...
    deny_in_scope_max: 0.7             # ...and the call is this far out of the agent's activity
    injection_max: 0.5                 # above: escalate (default for manipulation questions)
    project_min: 0.7                   # P(target_zone = project) above which an overwrite is the agent's job
    redact_params: [content]           # parameter names never sent to the model
```

`api_key_env` and `account_id_env` name environment variables; the secrets themselves stay out of the file. The code defaults above are conservative, for a new installation without measurements; the values used in production were measured, see [Measuring](measuring.md). The questions, their families and how sup7 decides from them: [Jev and question sets](jev.md).

A missing environment variable, a network or HTTP error, or an unreadable answer counts as a failure, which escalates (or moves to the next provider in a chain). The probabilities are written into the reasoning, headed by the model and the question fingerprint (`Jev jev-1.13.0 q=9bf9f0bc317f: approve (…)`).

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

Each chain entry is a full `evaluator` block; `breaker_failures` and `breaker_cooldown` are read from the top level. A chain entry may set its own `confidence_threshold`, which then applies to that provider's verdicts: confidences are not comparable across models (Jev's is computed, an LLM's is self-reported). Unset, the top-level threshold applies. When `chain` is non-empty it replaces the single `provider`.

Providers are tried in order and the first one that answers gives the verdict; an `escalate` verdict is an answer, not a failure. The next provider is tried only when the previous one fails (network error, HTTP error such as 402, timeout, unreadable answer). After `breaker_failures` consecutive failures a provider is skipped for `breaker_cooldown` seconds. If every provider fails, sup7 escalates to a human. The reasoning is prefixed with the provider that answered and those skipped or failed, e.g. `[ollama, jev skipped] ...`, and `GET /status` on the admin API reports each provider as `ok`, `failing` or `skipped`.

## Admin API

```yaml
admin:
  enabled: true
  host: 127.0.0.1
  port: 9096
  token: ""
```

Off by default. Routes: `GET /health` (no token), `GET /status`, `GET /config` (never secrets; shows each provider's effective threshold, the poll scope and the question sets), `GET /decisions?limit=50` (1 to 500), `POST /pause`, `POST /resume`, the file routes (`/files`) and the evaluation routes (`/bench/...`). Editing and starting a run need `token`. See [the overview](index.md#admin-api).

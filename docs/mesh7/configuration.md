# Configuration

Every feature is declared in a single YAML file. This page is the complete reference; [Getting Started](getting-started.md) shows the minimal config, [Writing Policies](writing-policies.md) goes deep on rules.

All features are declared in a single YAML config.

## MCP servers

```yaml
mcp_servers:
  - name: filesystem
    transport: stdio
    command: npx
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/home/me"]
    env:                     # optional, added to mesh7's own environment
      NODE_OPTIONS: "--max-old-space-size=512"

  - name: remote-service
    transport: sse
    url: "https://mcp-server.example.com/sse"
    headers:
      Authorization: "Bearer <token>"

  - name: huggingface
    transport: streamable-http
    url: "https://huggingface.co/mcp"
```

`streamable-http` is the transport hosted MCP servers now default to. It holds
no long-lived stream: each JSON-RPC request is a POST whose response carries
the answer, either as JSON or as an event stream. Sessions (`Mcp-Session-Id`)
are handled transparently, and cross-origin redirects are refused so a
compromised upstream cannot relay your tool calls elsewhere.

`env` applies to `stdio` servers: the child inherits mesh7's environment plus
these entries. Values are taken literally (no `${VAR}` expansion).

## OpenAPI specs

Import REST APIs as governed tools — persisted across restarts.

```yaml
openapi:
  # From URL
  - url: https://date.nager.at/swagger/v3/swagger.json

  # From local file
  - file: ./specs/internal-api.json
    backend_url: http://localhost:3001
```

Each endpoint becomes a tool (e.g. `get_public_holidays`). Same policy, same traces as MCP tools.

## CLI tools

Wrap any CLI binary behind policy, approval, and tracing:

```yaml
cli_tools:
  - name: gh
    bin: gh
    default_action: allow

  - name: terraform
    bin: terraform
    default_action: human_approval
    commands:
      plan:
        timeout: 120s

  - name: kubectl
    bin: kubectl
    strict: true      # only declared commands, everything else denied
    commands:
      get:
        allowed_args: ["-n", "--namespace", "-o"]

  - name: jq          # binary without subcommands
    bin: jq
    default_action: allow
    bare:
      allowed_args: ["-r", "--compact-output"]

  - name: make
    bin: make
    working_dir: /srv/project   # cwd of the process (default: mesh7's cwd)
    env:                        # added to a minimal PATH/HOME/LANG environment
      CI: "true"
```

Agents call CLI tools like any MCP tool — `terraform.plan`, `kubectl.get`, `gh.pr`. A `bare` binary registers a single `<name>.run` tool. Every CLI tool accepts an optional `stdin` param, piped to the process as data (never shell-interpreted).

A CLI process does not inherit mesh7's environment: it gets `PATH`, `HOME`, `LANG=en_US.UTF-8` and the entries of `env`, nothing else. Secrets present in the daemon's environment therefore do not leak to wrapped binaries unless you list them.

`default_action` is the floor for the dynamic dispatcher (`<name>.__dispatch`), which is where undeclared subcommands land. It can only restrict, never widen, and defaults to `deny` — so a glob such as `terraform.*: allow` cannot hand over `destroy` along with `plan`. See [CLI Tools](cli-tools.md#dispatcher-floor).

## Policies

YAML-based, first-match-wins, glob patterns for agents and tools.

```yaml
policies:
  - name: support-agent
    agent: "support-*"
    rate_limit:
      max_per_minute: 30
      max_total: 1000
    rules:
      - tools: ["*.read_*", "*.list_*", "*.get_*"]
        action: allow
      - tools: ["create_refund"]
        action: allow
        condition:
          field: "amount"        # the path starts at the tool's arguments
          operator: "<"
          value: 500
      # Conditions read strings too, so a rule can look inside the call
      # rather than only at its name. A list means "any of these".
      - tools: ["Bash"]
        action: deny
        condition:
          field: "command"
          operator: "contains"
          value: ["mkfs", "> /dev/sd", "/etc/sudoers"]
      - tools: ["*"]
        action: deny
```

Operators: `<` `<=` `>` `>=` `==` `!=` on numbers, `==` `!=` `contains`
`not_contains` `starts_with` `not_starts_with` on strings. String matching is
literal and case-sensitive: it raises the floor against accidents, it is not a
sandbox. See [writing-policies.md](https://github.com/KTCrisis/flux7-mesh/blob/main/docs/writing-policies.md).

| Action | Behavior |
|--------|----------|
| `allow` | Forward to backend, return result |
| `deny` | Block the call, return denial |
| `human_approval` | Require human approval before forwarding |

**Fail closed:** no matching rule = deny.

## Per-agent policy files

One file per agent, drop-in/drop-out:

```yaml
# config.yaml
policy_dir: ./policies   # load all *.yaml from this directory
```

```yaml
# policies/scout7.yaml
name: scout7
agent: "scout7"
rate_limit:
  max_per_minute: 30
rules:
  - tools: ["searxng.*", "fetch.*", "ollama.*", "memory.*"]
    action: allow
  - tools: ["*"]
    action: deny
```

Files are loaded alphabetically after inline `policies:`. Duplicate names produce an error.

## Policy hot-reload

Policies are reloaded automatically when files change — no restart required. The daemon watches:

- `config.yaml` (inline `policies:` section)
- `policy_dir/` (all `*.yaml` files)

Changes are debounced (200ms) and validated before applying. If the new YAML is invalid, the current policies are kept and the error is logged. Rate limits defined in policies are also reloaded.

```
# Add a new agent policy at runtime — takes effect in <1s
echo 'name: temp-agent
agent: "temp"
rules:
  - tools: ["weather.*"]
    action: allow' > policies/temp.yaml

# Remove it — reverts immediately
rm policies/temp.yaml
```

Hot-reload covers policies and rate limits only. Changes to MCP servers, CLI tools, or OpenAPI specs require a restart.

## Supervisor mode

```yaml
supervisor:
  enabled: true          # MCP agents block on human_approval until resolved
  expose_content: false  # redact raw params → structural metadata
  supervisor_agents:     # agent IDs (glob) allowed to see approval and grant tools
    - "supervisor-*"
```

The operator tools exposed over MCP (`approval.resolve`, `approval.pending`, `grant.create`, `grant.revoke`) are visible and callable only by agents matching `supervisor_agents`, whether or not `enabled` is set. Any other agent gets them neither in `tools/list` nor on call, and resolves approvals through the admin-gated HTTP API or the `mesh` CLI. `grant.list` and `mesh.catalog` stay visible to everyone.

`enabled` changes what an MCP agent experiences on `human_approval`: instead of receiving an immediate "pending" result, the call blocks until a supervisor (or a human) resolves it, then returns the outcome. See [Supervisor Protocol](supervisor-protocol.md).

This enables a Managed Agent (e.g. Claude via MCP Streamable HTTP) to act as a cloud supervisor, connecting to `POST /mcp` with an identity that matches `supervisor_agents` and resolving approvals with Claude's judgment.

## Memory integration

Persist approval decisions as queryable facts in [mem7](https://github.com/KTCrisis/flux7-memory). Fire-and-forget — a failing mem7 never blocks approvals.

```yaml
memory:
  url: http://localhost:9070    # mem7 daemon URL
  token: ""                     # optional Bearer token
```

When configured, every approval resolve (approve, deny, timeout) is written to mem7 as a fact with tags `[decision, approved|denied, <tool>, agent:<id>]`.

**Auto-approve from past decisions** — when `memory.url` is set, mesh7 queries mem7 before submitting to the approval queue. If a tool+agent pattern has 3+ consistent approvals with 0 rejections, it is auto-approved (traced as `supervisor:mem7`). Governance gets less intrusive over time without getting less safe.

```yaml
supervisor:
  auto_approve: true     # default true when memory.url is set
  min_approvals: 3       # threshold for auto-approve (default 3)
```

The auto-approve is a pre-filter (Level 1). If it can't resolve, the request proceeds to the external supervisor (if running) or human. If mem7 is down, the request is escalated — never blocked. See [docs/mem7-auto-approve.md](https://github.com/KTCrisis/flux7-mesh/blob/main/docs/mem7-auto-approve.md) for a step-by-step example.

## Authentication

Two planes, two guards:

```yaml
auth:
  # Control plane (traces, grants, approvals, policies, sessions, metrics).
  # When set, these endpoints require `Authorization: Bearer <token>`.
  # When empty, they are restricted to loopback callers only.
  # MESH_ADMIN_TOKEN env overrides this value.
  admin_token: "a-long-random-secret"

  # Data plane: validate agent identity via JWT against an external IdP.
  jwt:
    jwks_url: https://idp.example.com/.well-known/jwks.json
    issuer: https://idp.example.com      # optional
    audience: mesh7                      # optional
    agent_claim: sub                     # claim used as agent id (default: sub)
    user_claim: ""                       # claim naming the human the agent acts for (default: off)
    allow_legacy: false                  # keep plaintext "agent:<id>" off when JWT is on

  # Reject data-plane requests with no credentials (401) instead of letting
  # them resolve to "anonymous" and fail closed at the policy engine.
  require_authentication: false
```

| Setting | Guards | Default behavior when unset |
|---------|--------|------------------------------|
| `admin_token` | Control plane — a caller here can mint grants and resolve approvals, overriding what policies enforce | Loopback-only |
| `jwt` | Data-plane identity — cryptographic agent id instead of the spoofable `agent:<id>` header. With `user_claim` set, a delegation-shaped token also carries the human the agent acts for, recorded on every trace (`user_id`) and OTel span (`enduser.id`) | Plaintext identity accepted |
| `require_authentication` | Anonymous data-plane access: `POST /tool/*`, `POST /decide`, `POST`/`DELETE /mcp`, and `/tools`, `/mcp-servers` enumeration all return `401` without a credential | Anonymous allowed, governed by policy (fails closed) |

The data plane (tool calls, `/decide`, `/mcp`, `/health`) is never gated by `admin_token`. Details: [control-plane auth](https://docs.flux7.art/mesh7/control-plane-auth/) and [JWT authentication](https://docs.flux7.art/mesh7/jwt-auth/).

## Tool catalogue

```yaml
pin_tools: true             # fingerprint upstream MCP tools; hold back new and changed ones
hide_denied_tools: true     # leave out of tools/list what the policy can only deny
```

Both are off by default: on a machine where the operator is also the user, the full list and no pinning is usually what is wanted.

`pin_tools` keeps a fingerprint of every upstream MCP tool (description, parameter schemas, annotations) in `storage_path`. A server seen for the first time is trusted and pinned as is; afterwards a tool it adds is denied and a tool that changed asks for approval, whatever the policy says, until it is accepted. Without `storage_path` the pins live in memory and are re-trusted at every start.

`hide_denied_tools` removes from an MCP client's `tools/list` every tool whose every path through the policy ends in `deny` for that agent, so the model never sees it. A tool with an approval or a conditional allow stays listed, and calls are enforced either way.

Details: [Tool Classification](tool-classification.md#pinning-the-catalogue).

## Other settings

```yaml
port: 9090                                   # HTTP port (default 9090)
storage_path: state.db                       # SQLite durable state (approvals, grants survive restarts)
trace_file: traces.jsonl                     # JSONL persistence, hash-chained (MESH_TRACE_KEY → HMAC)
otel_endpoint: /path/to/traces-otel.jsonl    # or "stdout" or "http://localhost:4318"
otel_headers:                                # sent on every OTLP/HTTP request, ${VAR} expanded
  Authorization: "Bearer ${OTLP_TOKEN}"
otel_ca_cert: /etc/mesh7/collector-ca.pem    # PEM appended to the system roots
otel_insecure_skip_verify: false             # self-signed local collector only
approval:
  timeout_seconds: 300                       # approval TTL (default 5 min)
  notify_url: https://hooks.slack.com/...    # webhook on new pending approval
  channel: tty-fallback                      # queue | tty | tty-fallback (default)
tls:                                         # optional in-binary TLS, both fields required
  cert_file: /etc/mesh7/tls.crt
  key_file: /etc/mesh7/tls.key
```

`approval.channel` routes `human_approval` for MCP tool calls: `queue` always enqueues (daemons, supervisor setups), `tty` requires the interactive `/dev/tty` prompt and denies when none is available, `tty-fallback` tries the prompt and falls back to the queue. Any other value is a config error. REST calls (`POST /tool/*`) always use the queue.

`tls` serves HTTPS directly when both `cert_file` and `key_file` are set. Without it mesh7 serves plaintext and logs a warning; the recommended deployment keeps it on loopback or an internal network behind an ingress that terminates TLS.

Every line of `trace_file` is chained to the previous one; set `MESH_TRACE_KEY` in the service environment to make it an HMAC chain, and check it with `mesh7 trace verify` or `GET /traces/verify`. See [Trace Integrity](trace-integrity.md) and [Observability](otel.md) for OTLP delivery (batches, retries, trace context).

---

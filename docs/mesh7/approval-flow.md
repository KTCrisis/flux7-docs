# Approval Flow

When a policy rule has `action: human_approval`, the tool call enters an approval queue. The agent waits. A human (or supervisor) resolves it. The tool call proceeds or is rejected.

## How it works

```
Agent calls filesystem.write_file
  │
  ├── Policy: human_approval
  │
  ├── Check 1: Temporal grant active?
  │   yes → bypass approval, proceed
  │
  ├── Check 2: flux7-memory precedents? (read tools only,
  │   enough human approvals of this tool + agent, no refusal)
  │   yes → proceed, traced as supervisor:mem7
  │
  └── Submit to approval queue
      │
      ├── MCP mode: routed per approval.channel (TTY prompt or queue);
      │   with approval.wait_seconds, waits that long for a supervisor
      ├── HTTP mode: always the queue — blocks until resolved
      │
      └── Human/supervisor resolves
          ├── approved → tool call executed, result returned
          ├── denied → 403 returned
          └── timeout → 408 returned (default 5 min)
```

## Routing: `approval.channel`

In MCP mode, `approval.channel` decides where a `human_approval` request goes:

```yaml
approval:
  timeout_seconds: 300
  channel: queue        # queue | tty | tty-fallback
```

!!! note "Version"
    Since v0.15.0. Older binaries silently ignore the key.

| Channel | Behavior |
|---|---|
| `queue` | Always enqueue. For daemons and supervisor setups — a service has no terminal to prompt on. |
| `tty` | Require the interactive `/dev/tty` prompt. If no TTY is available, the call is **denied** (fail-closed), never silently queued. |
| `tty-fallback` | Try the TTY prompt, fall back to the queue. Default — matches the historical behavior. |

Without an explicit channel, routing depends on how the process was launched: a daemon started from a terminal keeps a usable `/dev/tty` and will prompt in a window nobody watches. Set `channel: queue` for any unattended deployment (systemd, container, supervisor loop).

The HTTP proxy path (`POST /tool/{name}`) always uses the queue regardless of this setting.

## Waiting for an automatic decision: `approval.wait_seconds`

In MCP mode a `human_approval` call is non-blocking: the agent gets the approval id at once and retries the same call once it is approved. When a supervisor such as [sup7](../sup7/index.md) decides in about half a second, that retry is friction, and an LLM that rewrites its arguments on retry opens a new approval.

```yaml
approval:
  channel: queue
  wait_seconds: 3       # 0 (default): answer at once
```

With a wait, the call holds for up to `wait_seconds`. Decided in time, it runs (or is refused) **in the same request**, and the approval is used up, so a retry of the same call does not run it again. Not decided in time (a human, a slow supervisor), the agent gets the approval id as before. Keep it short: a stdio session is blocked while it waits, and a human never answers within it. Pair it with a supervisor that polls faster than the wait (sup7 `poll.interval: 500ms`).

Measured in production: a write sent to approval, approved by a sup7 rule in 447 ms, ran in the same request, without a retry.

## Changing approval settings at runtime

`GET /approvals/settings` and `PUT /approvals/settings` (control plane) read and change `approval.timeout_seconds`, `approval.wait_seconds`, `supervisor.auto_approve`, `supervisor.min_approvals` and `supervisor.auto_approve_writes` without a restart.

```bash
curl -s -X PUT http://localhost:9090/approvals/settings -H "Content-Type: application/json" \
  -d '{"timeout_seconds":300,"wait_seconds":3,"auto_approve":true,"min_approvals":3,"auto_approve_writes":false,"by":"alice"}'
```

A change is:

- validated: timeout 30 to 3600 s, wait 0 to 10 s, human approvals needed 1 to 100;
- written back to the config file line by line, through the YAML tree: comments, key order and file mode are kept, and the previous version is saved as `<config>.bak-<timestamp>` (never overwriting an earlier backup);
- read back to check the file says what was asked (otherwise the file is restored);
- applied at once to every session;
- traced as `mesh.approval_settings_edit`, with the values before and after.

flux7-console edits them from the Approvals page.

## Resolving approvals

### In Claude Code (MCP mode): approve, then retry

When no terminal prompt is available (Claude Code runs mesh7 without one), the call returns at once:

```
Approval required (id: d9a80d29) for filesystem.write_file, valid 299s.
Ask the user to approve it, with one of:
  mesh approve d9a80d29        (terminal)
  the Approvals page of flux7-console
  POST /approvals/d9a80d29/approve on the mesh HTTP port
Once approved, call filesystem.write_file again with the same arguments: the approved call runs once.
```

The agent relays this to its user. Once a human approves, the agent's retry of **the same tool with the same arguments**, by the same agent, within the approval's validity, runs, and uses up the approval: a second retry asks again, and a call with other arguments gets its own approval. The agent itself cannot approve (see below). Since v0.17.0; earlier versions pointed the agent at `approval.resolve`, which a regular agent cannot call.

An agent can also **wait** for the decision by retrying the same call on a timer:

- **While the approval is pending**, each retry meets the approval already open: same id, no new request in the queue, no new trace line.
- **Once a human denies it**, the next retry learns the refusal, once, as an error: `Approval denied by <who> for <tool> (id: …): do not retry this call.` The agent can stop waiting; a later call is a new request for a human.

Both since the release after v0.17.2 (on `main`). Before, every retry while pending opened another approval (five retries, six requests), and after a denial the agent could only keep asking until its patience ran out.

### Via MCP virtual tools

The `approval.*` virtual tools are **operator-only**: only a declared supervisor
agent (`supervisor.supervisor_agents` glob) may list or resolve approvals over
MCP. A regular agent cannot, otherwise it could approve its own pending
`human_approval` request and defeat the gate. With no supervisor configured,
resolve approvals via the CLI or HTTP API below.

```
approval.pending                → list pending approvals (supervisor agents only)
approval.resolve {id, decision} → approve or deny (supervisor agents only)
```

### Via CLI

```bash
# List pending
mesh pending

# Approve (prefix match)
mesh approve a1b2c3d4

# Approve, and stop being asked about this exact tool for an hour.
# The grant records this approval as its origin (see "Chain of authority").
mesh approve a1b2c3d4 --grant 1h

# Same, widening the grant beyond the single tool (deliberate)
mesh approve a1b2c3d4 --grant 1h --tools "filesystem.write_*"

# Deny
mesh deny a1b2c3d4

# Watch (live updates): [a]pprove, [g]rant, [d]eny, [s]kip
mesh watch
```

In `watch`, `[g]` approves and opens a grant in one keystroke. Its duration comes
from `MESH_GRANT_DURATION` (default `1h`), and its pattern is the exact tool
approved. A failing grant never fails the approval: the call was already let
through, and losing the shortcut is the lesser harm.

### Via HTTP API

```bash
# List all approvals
curl http://localhost:9090/approvals

# Get details (includes recent traces and active grants)
curl http://localhost:9090/approvals/a1b2c3d4e5f6g7h8

# Approve
curl -X POST http://localhost:9090/approvals/a1b2c3d4/approve \
  -H "Content-Type: application/json" \
  -d '{"resolved_by":"user:marc","reasoning":"routine operation","confidence":0.95}'

# Deny
curl -X POST http://localhost:9090/approvals/a1b2c3d4/deny \
  -d '{"resolved_by":"user:marc","reasoning":"unexpected target"}'
```

Prefix matching: `a1b2c3d4` matches the full ID if the prefix is unique.

## Temporal grants

Repeated approvals for the same tool pattern get tedious. Grants are like `sudo` — a temporary bypass:

```
grant.create {tools: "filesystem.write_*", duration: "30m"}
```

For the next 30 minutes, all `filesystem.write_*` calls bypass the approval queue. Traced as `grant:<id>`.

### Chain of authority

A grant issued out of nowhere is an orphan: it authorizes calls without saying
why it exists, and "why was this allowed?" stops at "because a grant covered it".

So a grant can record its origin, the approval and the call it answers:

```bash
curl -X POST http://localhost:9090/grants -d '{
  "agent": "claude", "tools": "filesystem.write_*", "duration": "1h",
  "approval_id": "<the approval>", "trace_id": "<the call being approved>"
}'
```

`mesh approve <id> --grant <duration>` and `[g]` in `watch` fill both fields on
their own; nobody copies an ID by hand. Every call the grant later waves through
then carries `grant_id` and `parent_trace_id`, and the chain is walkable:

```bash
curl "http://localhost:9090/traces/<trace-id>/why"
```

The response is JSON (`trace_id`, `chain_length`, `chain`); the `chain` array
holds the full trace entries, oldest first. Reduced to the fields that matter:

```
0. fa12168e  echo7.run  human_approval  rule=demo
1. 384b0ab7  echo7.run  allow           rule=grant:a9b240ef  grant=a9b240ef
```

`depth` bounds the walk (default 10, capped at 50). The same edge appears as
`parentSpanId` in the OTLP export, so Jaeger or Tempo renders the tree directly
(see [OpenTelemetry](otel.md)). The flux7-console trace detail reads the same
endpoint and shows the chain (approval, grant, call) next to the trace.

Origin is always optional. A grant issued without one still works and yields a
chain of one, which is an honest answer rather than a gap.

### MCP tools

```
grant.create  {tools: "filesystem.*", duration: "1h"}   (supervisor agents only)
grant.list                                              (everyone, read-only)
grant.revoke  {id: "abc123"}                            (supervisor agents only)
```

`grant.create` and `grant.revoke` are operator-only, like `approval.*`: only an
agent matching `supervisor.supervisor_agents` sees or calls them. Otherwise an
agent could grant itself a bypass of its own `human_approval` gate. Everyone
else creates grants with the CLI (`mesh approve --grant`) or the HTTP API.

### HTTP API

```bash
# Create
curl -X POST http://localhost:9090/grants \
  -d '{"agent":"claude","tools":"filesystem.*","duration":"30m"}'

# List
curl http://localhost:9090/grants

# Revoke
curl -X DELETE http://localhost:9090/grants/abc123
```

!!! warning "Grants only bypass `human_approval`"
    Tools marked `deny` remain blocked. A grant cannot override a deny rule — that requires a policy edit.

## Timeouts

Unanswered approvals time out after 5 minutes (configurable):

```yaml
approval:
  timeout_seconds: 300    # default
```

Timed-out approvals are recorded in traces and written to flux7-memory (if configured) with status `timeout`.

## Webhooks

Get notified when a new approval is pending:

```yaml
approval:
  notify_url: https://hooks.slack.com/services/...
```

flux7-mesh POSTs to this URL with the pending approval details. Useful for Slack/Teams alerts.

## Callback URL

HTTP agents can provide a callback URL to receive the resolution:

```bash
curl -X POST http://localhost:9090/tool/gmail.send_email \
  -H "Authorization: Bearer agent:my-bot" \
  -H "X-Callback-URL: http://my-bot:8080/approval-callback" \
  -d '{"params":{"to":"user@example.com","subject":"Hello"}}'
```

When the approval resolves, flux7-mesh POSTs the result to `X-Callback-URL`.

## Supervisor mode

The approval and grant-mutating tools are hidden from every agent that does not match `supervisor_agents`, whether or not supervisor mode is on. With `supervisor.enabled: true`, an external supervisor resolves approvals:

```yaml
supervisor:
  enabled: true
  expose_content: false       # redact params for the supervisor
  supervisor_agents:          # whitelist (glob) for cloud supervisors
    - "supervisor-*"
```

In this mode, MCP tool calls **block** until the supervisor resolves. The agent waits transparently.

See [Supervisor Protocol](supervisor-protocol.md) for the external resolver API.

## Traces

Every approval is traced:

```json
{
  "tool": "filesystem.write_file",
  "policy": "human_approval",
  "approval_id": "a1b2c3d4e5f6g7h8",
  "approval_status": "approved",
  "approved_by": "user:marc",
  "supervisor_reasoning": "routine write operation",
  "supervisor_confidence": 0.95,
  "latency_ms": 4200
}
```

Query approval history:

```bash
curl "http://localhost:9090/traces?tool=filesystem.write_file" | python3 -m json.tool
```

## Decision persistence

When [flux7-memory](mem7-auto-approve.md) is configured, every approval resolution (approve, deny, timeout) is stored as a queryable fact. This enables:

- **Auto-approve** — routine patterns resolve without human intervention
- **Audit trail** — "who approved what, when, why" is queryable
- **Cross-session memory** — decisions survive process restarts

## Next steps

- [Memory Integration](mem7-auto-approve.md) — auto-approve from past decisions
- [CLI Tools](cli-tools.md) — governing git, docker, terraform
- [Deployment Modes](deployment-modes.md) — solo, team, cloud

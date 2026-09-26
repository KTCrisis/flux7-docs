# Python SDK

`pip install flux7-mesh` — govern tool calls from any Python code: Claude API, the Anthropic Agent SDK, LangChain, or plain HTTP. Source and full examples: [sdk/python](https://github.com/KTCrisis/flux7-mesh/blob/main/sdk/python/).

## Python SDK

```bash
pip install flux7-mesh              # core client
pip install flux7-mesh[anthropic]   # with Claude API support
```

Govern tool calls from any Python code — Claude API, LangChain, or plain HTTP:

```python
from mesh7 import GovernedToolkit

toolkit = GovernedToolkit(agent="my-agent")

@toolkit.tool
def get_weather(city: str) -> str:
    """Get current weather for a city."""
    return fetch_weather(city)

# Generate tools[] for Claude API — names are namespace-qualified
# with a double underscore, e.g. "my-agent__get_weather"
response = client.messages.create(
    model="claude-sonnet-4-6",
    tools=toolkit.schemas(),
    messages=[...],
)

# Execute with governance (policy + trace)
results = toolkit.process_response([b.model_dump() for b in response.content])
```

The Claude API rejects dots in tool names, so `schemas()` emits `agent__tool` (since 0.4.1). The toolkit maps the name back to the dotted form (`my-agent.get_weather`) before asking the mesh, so policies keep matching on `<agent>.<tool>`.

Or use the client directly:

```python
from mesh7 import AgentMesh

mesh = AgentMesh("http://localhost:9090", agent="my-agent")
decision = mesh.call_tool("filesystem.write_file", {"path": "/tmp/x", "content": "hello"})
print(decision.action)  # allow | deny | human_approval
```

`decide()` asks the same question without executing anything (`POST /decide`); it is what the hooks use:

```python
d = mesh.decide("Bash", {"command": "rm -rf build/"})
if d.action == "deny":
    print(d.error)  # the policy's reason
```

The client also wraps the approval and grant endpoints, for a supervisor or an operator script:

```python
for a in mesh.pending(tool_scope="filesystem.*"):
    detail = mesh.approval_detail(a["id"])      # recent traces + active grants
    mesh.resolve(a["id"], "approve", resolved_by="agent:supervisor",
                 reasoning="write inside project dir", confidence=0.9)

mesh.grants()                                   # active grants
g = mesh.create_grant("filesystem.write_*", "30m")
mesh.revoke_grant(g["id"])
```

`approve()` and `deny()` are the short forms of `resolve()`. These helpers call control-plane endpoints and send the client's agent credential, not an admin token: they work against a mesh reached from loopback with no `admin_token` set. `create_grant()` grants to the self-declared `agent` and takes no origin; to record `approval_id` / `trace_id` or to reach a token-protected mesh, call `POST /grants` directly ([Supervisor Protocol](supervisor-protocol.md#grant-after-approval)).

## Agent SDK hooks

```python
from mesh7 import MeshHooks

hooks = MeshHooks(agent="my-agent")
# Pass to ClaudeAgentOptions(hooks=hooks.agent_sdk_hooks())
```

See [sdk/python/](https://github.com/KTCrisis/flux7-mesh/blob/main/sdk/python/) for full docs and examples.

## Harness hook

The proxy governs what an agent routes through mesh7. It cannot govern the tools
the harness runs itself — `Bash`, `Read`, `Write`, `Edit`. The SDK ships a
`PreToolUse` hook that closes that side, so one policy file covers both:

```json
{
  "hooks": {
    "PreToolUse": [
      { "matcher": ".*", "hooks": [{ "type": "command", "command": "mesh7-hook" }] }
    ]
  }
}
```

It starts in observe mode — it traces without refusing anything — so you can
write the rules from what you actually see before switching to enforce. See
[docs/harness-hook.md](https://github.com/KTCrisis/flux7-mesh/blob/main/docs/harness-hook.md).

Configuration is by environment variable:

| Variable | Default | Meaning |
|----------|---------|---------|
| `MESH7_HOOK_MODE` | `observe` | `observe` traces only; `enforce` applies the verdict |
| `MESH7_URL` | `http://localhost:9090` | mesh7 data plane |
| `MESH7_AGENT` | `claude` | agent identity to evaluate |
| `MESH7_TOKEN` | *(unset)* | JWT presented instead of the self-declared agent |
| `MESH7_HOOK_TIMEOUT` | `5` | seconds before giving up on the mesh |
| `MESH7_HOOK_SKIP_PREFIX` | `mcp__` | comma-separated tool prefixes left to the proxy (MCP tools already cross it) |

In `enforce` mode, `allow` and `deny` pass through, `human_approval` becomes the harness's own `ask` prompt, and an unreachable mesh or a malformed answer is a `deny` (fail closed). In `observe` mode the hook never answers, so the harness's usual permission flow is unchanged.

## Identity: self-declared or a JWT

`agent="my-agent"` is a self-declared identity, sent as `Bearer agent:my-agent`.
A mesh that validates JWTs (`auth.jwt` set, `allow_legacy` off) rejects it.
Pass the token your identity provider issued instead:

```python
mesh = AgentMesh("https://mesh.example.com", agent="my-agent", token=jwt)
```

The mesh then resolves the agent from the token's claims — and, if the token
carries one, the human the agent acts for ([`user_claim`](jwt-auth.md)), which
lands on every trace as `user_id`. The same `token=` exists on
`GovernedToolkit` and `MeshHooks`; the CLI hook reads it from `MESH7_TOKEN`.
Available from SDK 0.6.0.

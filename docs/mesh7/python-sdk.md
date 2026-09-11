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
# e.g. "my-agent.get_weather"
response = client.messages.create(
    model="claude-sonnet-4-6",
    tools=toolkit.schemas(),
    messages=[...],
)

# Execute with governance (policy + trace)
results = toolkit.process_response([b.model_dump() for b in response.content])
```

Or use the client directly:

```python
from mesh7 import AgentMesh

mesh = AgentMesh("http://localhost:9090", agent="my-agent")
decision = mesh.call_tool("filesystem.write_file", {"path": "/tmp/x", "content": "hello"})
print(decision.action)  # allow | deny | human_approval
```

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

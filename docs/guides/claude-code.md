# Govern Claude Code

Claude Code reaches tools two ways: through MCP servers, and through its own
built-in tools (`Bash`, `Read`, `Write`, `Edit`, `WebFetch`…). flux7-mesh
governs both with one policy file: the proxy for the first, a hook for the
second.

| Surface | Path | Setup |
|---|---|---|
| MCP tools | Claude Code → mesh7 → MCP servers | declare the servers in the mesh, not in Claude Code |
| Built-in tools | Claude Code asks mesh7 before running them | `mesh7-hook`, a `PreToolUse` hook |

## 1. Route MCP servers through the mesh

Move each MCP server from Claude Code's config into `config.yaml` under
`mcp_servers`, then register the mesh as Claude Code's only server:

```bash
claude mcp add mesh7 -- mesh7 --mcp --config config.yaml
```

For a mesh that outlives your sessions, run it as a service
(`mesh7 serve`, see [Deployment modes](../mesh7/deployment-modes.md#running-mesh7-as-a-service)):
`mesh7 --mcp` detects it and relays to it.

A server left in Claude Code's own config is not governed. The mesh cannot see
what it never receives.

## 2. Install the hook, in observe mode

```bash
pip install flux7-mesh
```

```json title="~/.claude/settings.json"
{
  "hooks": {
    "PreToolUse": [
      { "matcher": ".*",
        "hooks": [{ "type": "command", "command": "mesh7-hook", "timeout": 10 }] }
    ]
  }
}
```

The hook starts in **observe** mode: every built-in call is evaluated and
traced, nothing is refused, your usual permission prompts are unchanged.
Work normally for a day.

## 3. Write the rules from what you saw

List the tools no rule covers yet:

```bash
curl -s 'http://localhost:9090/traces?agent=claude&limit=500' \
  | jq -r '.[] | select(.policy_rule == "default") | .tool' | sort | uniq -c | sort -rn
```

Each of these would be refused once you enforce. Name them in the same policy
as the MCP tools, guards first:

```yaml title="policies/claude.yaml"
name: claude
agent: "claude"
rules:
  - tools: ["Read", "Glob", "Grep", "WebSearch"]
    action: allow

  - tools: ["Bash"]
    action: deny
    condition: { field: "command", operator: "contains", value: ["mkfs", "/etc/sudoers"] }
  - tools: ["Bash"]
    action: human_approval

  - tools: ["Write", "Edit"]
    action: allow
    condition: { field: "file_path", operator: "starts_with", value: "/home/me/projects/" }
  - tools: ["Write", "Edit"]
    action: human_approval

  - tools: ["filesystem.read_*"]
    action: allow
```

## 4. Enforce

```bash
export MESH7_HOOK_MODE=enforce
```

`allow` runs without a prompt, `deny` never runs, `human_approval` becomes
Claude Code's own permission prompt. If the mesh is unreachable, the hook
refuses: it fails closed.

!!! warning "Enforce last"
    With `enforce` and no rule naming `Bash`, every shell call is refused at
    the first keystroke. Observe first, write the rules, then enforce.

## What this does not cover

- An MCP server Claude Code talks to directly, outside the mesh.
- Another harness without a `PreToolUse` hook: there, the same policy is only
  advisory.
- A determined evasion of a text condition (`rm -r -f`, a script file). The
  conditions catch accidents, not an adversary; see
  [what string matching does not do](../mesh7/writing-policies.md#what-string-matching-does-not-do).

**Reference:** [Python SDK and hook variables](../mesh7/python-sdk.md#harness-hook) ·
[Writing policies](../mesh7/writing-policies.md)

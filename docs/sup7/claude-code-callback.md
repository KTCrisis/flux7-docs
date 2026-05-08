# Claude Code Callback

The most powerful evaluation mode : Claude Code acts as the supervisor's brain, with full codebase context.

## Architecture

When `evaluator.provider` is `claude-code`, sup7 doubles as an MCP server exposing two tools :

- **`sup7_pending`** — returns evaluations waiting for review
- **`sup7_verdict`** — accepts a decision (action, confidence, reasoning)

```
Claude Code ──MCP──► flux7-mesh ──MCP──► sup7
                                          │
                     sup7_pending ◄───────┘  Claude Code pulls
                     sup7_verdict ────────►  Claude Code responds
                                          │
                     flux7-mesh ◄─────────┘  sup7 resolves
```

## Flow

1. An agent makes a tool call governed by flux7-mesh with `human_approval` policy
2. sup7 picks up the pending approval
3. Rules don't match → needs LLM evaluation
4. Provider is `claude-code` → sup7 queues the evaluation
5. Claude Code calls `sup7_pending` through flux7-mesh, sees the request with full context
6. Claude Code evaluates — it can read files, check git history, review the agent's recent activity
7. Claude Code calls `sup7_verdict` with its decision
8. sup7 receives the verdict, resolves the original approval in flux7-mesh
9. If timeout expires (default 120s) → escalate to human

## Why MCP callback

| | Direct LLM call | MCP callback |
|---|---|---|
| **Context** | Tool name + params only | Full codebase, git history, conversations |
| **Judgment** | Pattern matching | Semantic understanding |
| **Blocking** | Synchronous | Async — doesn't block other evaluations |
| **Coupling** | Tied to specific API | Any MCP-capable agent can evaluate |

Claude Code has context that no API call can replicate. It knows what the user is working on, what files were recently changed, what the project structure looks like. A file write to `src/main.py` means something different during a refactor than during a security audit.

## Configuration

### sup7 config

```yaml
evaluator:
  provider: claude-code
  callback_timeout: 120  # seconds to wait for verdict

mcp_server:
  enabled: true
  transport: stdio       # registered in flux7-mesh config
```

### flux7-mesh config

Register sup7 as an MCP source so Claude Code can see its tools :

```yaml
mcp_servers:
  - name: sup7
    command: sup7
    args: ["start", "--config", "sup7.yaml", "--mcp"]
    transport: stdio
```

### Policy

Allow the supervisor agent to resolve approvals :

```yaml
supervisor:
  enabled: true
  supervisor_agents: ["supervisor"]
```

## MCP tools reference

### sup7_pending

Returns a list of evaluations waiting for review.

**Parameters** : none

**Response** :

```json
[
  {
    "id": "abc123",
    "agent_id": "claude-coder",
    "tool": "filesystem.write_file",
    "params": {"path": "/home/user/project/src/main.py", "content": "..."},
    "policy_rule": "writes-need-approval",
    "injection_risk": false,
    "recent_traces": [...],
    "active_grants": [...]
  }
]
```

### sup7_verdict

Submit a decision for a pending evaluation.

**Parameters** :

| Field | Type | Description |
|-------|------|-------------|
| `approval_id` | string | The approval ID from `sup7_pending` |
| `action` | string | `"approve"`, `"deny"`, or `"escalate"` |
| `confidence` | float | 0.0–1.0 confidence score |
| `reasoning` | string | One sentence explanation |

**Response** : `"accepted"` or error message.

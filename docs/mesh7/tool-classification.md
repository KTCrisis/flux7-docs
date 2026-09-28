# Tool Classification

Writing a policy means judging every tool an upstream declares, for every agent. mesh7 helps with that judgment: it reads what each tool declares about itself, suggests a starting point, shows what the policy decides for each agent, and lets you change one tool's action without touching the rest of the file.

None of this changes how a call is decided. The policy alone decides; the classification is advice for the person writing it.

## Two families of tools

The question is where the meaning of a call lives.

| Family | The effect is stated by | Examples | What a rule on the name decides |
|---|---|---|---|
| **named** | the tool name, refined by structured arguments | `gmail.send_email`, `weather.forecast`, `docker.ps` | the effect itself |
| **generic** | the argument: the tool is an interpreter | `execute_sql`, a shell tool, a code-mode `execute`, a CLI dispatcher (`terraform.__dispatch`) | only whether the interpreter may run at all |

A generic tool can do anything its argument says. `allow` on its name hands over everything; a [condition](writing-policies.md#conditions) on its argument matches text, not meaning ([what string matching does not do](writing-policies.md#what-string-matching-does-not-do)). Keep generic tools on `human_approval` unless the upstream credential itself is narrow: the approver sees the full arguments, code included.

Tools that compose several calls behind one intent (`osm.analyze_commute`) cannot be told apart from named tools by their metadata, and classify as named.

## How a tool is classified

`registry.Classify` reads only what the upstream declares. Nothing is verified.

| Signal | Read | Write | Family |
|---|---|---|---|
| HTTP method (OpenAPI) | `GET`, `HEAD`, `OPTIONS` | any other | |
| MCP annotations | `readOnlyHint: true` | `readOnlyHint: false`, `destructiveHint: true` | |
| Verbs in the tool name, whole words only | `get`, `list`, `search`, `read`, `show`… | `create`, `send`, `delete`, `write`, `execute`, `run`… | |
| A string argument named `code`, `script`, `sql`, `command`, `cmd`, `statement`, `program`, `expression` | | | generic |
| A `query` argument on a tool whose name says `execute`, `exec`, `run` or `sql` | | | generic |
| A CLI dynamic dispatcher | | | generic |

Conflicting signals resolve to the restrictive side: one write signal outweighs any number of read signals, because a false *read* opens a hole while a false *write* only costs an approval. A tool with no signal is `unknown`. Every classification lists its reasons.

Annotations are claims. The MCP specification itself says not to trust them from an untrusted server, and some servers send the specification's defaults (`destructiveHint: true`) on every tool, which classifies them all as writes. The classification shows it; the decision stays yours.

## Starting from a draft

```bash
mesh7 discover --config config.yaml --generate-policy
```

Connects to the upstreams declared in the config (MCP servers, CLI tools, an OpenAPI spec with `--openapi`) without starting the proxy, and prints a draft:

- named tools that read → `allow`;
- everything else (writes, generic tools, tools with no signal) → `human_approval`;
- a final `deny` on `*`.

Each tool is preceded by a comment carrying its classification and reasons:

```yaml
      # MCP server "gmail": reads through a named tool
      #   gmail.gmail_list_emails [read] name verb "list"
      - tools: ["gmail.gmail_list_emails"]
        action: allow
      # MCP server "gmail": writes, generic tools, or no signal
      #   gmail.gmail_send_email [write] name verb "send"
      - tools: ["gmail.gmail_send_email"]
        action: human_approval
```

Starting strict and relaxing on evidence is cheaper than the reverse. [Memory Integration](mem7-auto-approve.md) relaxes on evidence at run time: consistent approvals of the same tool are auto-approved.

## What the policy decides, per agent

`GET /tools` returns every registered tool with its `classification` (`family`, `access`, `reasons`).

`GET /tools/decisions?agent=<id>` (control plane) returns, for every tool, what the policy decides for that agent before any call:

```json
{
  "name": "gmail.gmail_send_email",
  "classification": {"family": "named", "access": "write", "reasons": ["name verb \"send\""]},
  "action": "human_approval",
  "rule": "claude",
  "source_file": "claude.yaml",
  "rule_index": 4,
  "conditional": [
    {"action": "allow", "rule": "claude", "field": "to", "operator": "starts_with"}
  ]
}
```

- `action` / `rule`: where evaluation lands when no condition matches;
- `conditional`: rules with a condition that are evaluated first and win when their condition holds for a call;
- the dispatcher floor of a CLI tool is already applied;
- grants are not reflected: they belong to a session, not to the catalogue.

This is how a broad glob shows itself: a `docker.*: allow` written for read-only tools also allows any tool the upstream adds later.

## Changing one tool's action

`PUT /policies/{agent}/tools/{tool}` (control plane) with `{"action": "allow" | "deny" | "human_approval" | "inherit", "by": "..."}` edits that agent's policy file in `policy_dir`:

- the new rule names exactly this tool and lands **right before the rule that decides today** when that rule is in the agent's own policy, so conditional rules above it keep their say; otherwise at the end of the agent's policy, which is still evaluated before shared glob policies;
- a rule already naming exactly this tool is rewritten in place;
- the rule is headed by `# set from console, <date> by <by>`;
- `inherit` removes such a rule and falls back to whatever comes next. It refuses (`409`) a rule written by hand: those are edited in the file.

The file is edited as text, not re-serialized: comments and blank lines stay as written. The previous version is kept as `<file>.bak`, the write is atomic and keeps the file's mode, the whole policy set is re-validated (the file is restored if it fails), then applied at once through the hot-reload path. Each edit is recorded in the trace chain as the tool `mesh.policy_edit`, with the action before and after.

`by` is recorded as given: the admin token proves the right to edit, not an identity.

flux7-console drives both endpoints from its Tools page: a family filter, an agent selector, a *To review* filter for tools the policy allows without being a plain named read, and a selector on each decision.

## Next steps

- [Writing Policies](writing-policies.md): rules, globs, conditions, per-agent files
- [CLI Tools](cli-tools.md): dispatchers and their floor
- [Reference](reference.md): the full HTTP API

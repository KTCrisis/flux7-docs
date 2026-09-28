# Getting Started

Install flux7-mesh, write your first policy, and make a governed tool call. Five minutes.

## Install

=== "Go install"

    ```bash
    go install github.com/KTCrisis/flux7-mesh/cmd/mesh7@latest
    go install github.com/KTCrisis/flux7-mesh/cmd/mesh@latest   # approval CLI
    ```

=== "Script (Linux, macOS)"

    ```bash
    curl -fsSL https://raw.githubusercontent.com/KTCrisis/flux7-mesh/main/install.sh | sh
    ```

    Installs `mesh7` and `mesh` in `~/.local/bin` (`/usr/local/bin` as root). Options: `--prefix DIR`, `--version vX.Y.Z`, and on Linux `--service` or `--system` to run mesh7 as a daemon (see [Deployment modes](deployment-modes.md#running-mesh7-as-a-service)). Pass options after `sh -s --`.

=== "Binary (Linux amd64)"

    ```bash
    curl -L https://github.com/KTCrisis/flux7-mesh/releases/latest/download/mesh7_linux_amd64.tar.gz | tar xz
    sudo mv mesh7 mesh /usr/local/bin/
    ```

    The archive holds `mesh7` (the proxy) and `mesh` (the approval CLI). Other targets: `mesh7_{linux,darwin}_{amd64,arm64}.tar.gz`, `mesh7_windows_{amd64,arm64}.zip`. Archives are named without a version since v0.17.0, so `releases/latest/download/` always resolves.

Verify:

```bash
mesh7 --version
```

## Minimal config

Create `config.yaml`:

```yaml
mcp_servers:
  - name: filesystem
    transport: stdio
    command: npx
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/home/user"]

policies:
  - name: default
    agent: "*"
    rules:
      - tools: ["filesystem.read_file", "filesystem.list_directory"]
        action: allow
      - tools: ["filesystem.write_file"]
        action: human_approval
      - tools: ["*"]
        action: deny
```

This config:

1. Connects to the filesystem MCP server
2. Allows read operations
3. Requires human approval for writes
4. Denies everything else

## Run with Claude Code

```bash
claude mcp add mesh7 -- mesh7 --mcp --config config.yaml
```

Claude Code now routes all tool calls through flux7-mesh. Open Claude Code and try:

```
> Read the file config.yaml
```

This should work (policy: allow). Now try:

```
> Write "hello" to /home/user/test.txt
```

You'll see an approval prompt. Say yes — flux7-mesh traces the decision.

## Run standalone (HTTP mode)

```bash
mesh7 --config config.yaml
```

```bash
# List available tools
curl http://localhost:9090/tools | python3 -m json.tool

# Call a tool
curl -X POST http://localhost:9090/tool/filesystem.read_file \
  -H "Authorization: Bearer agent:my-script" \
  -H "Content-Type: application/json" \
  -d '{"params":{"path":"/home/user/config.yaml"}}'

# Check traces
curl http://localhost:9090/traces | python3 -m json.tool
```

## What just happened

```
Your agent ──► flux7-mesh ──► filesystem MCP server
                  │
                  ├── policy check (allow / deny / human_approval)
                  ├── rate limit check
                  ├── trace recorded (JSONL)
                  └── approval queue (if human_approval)
```

Every tool call is logged. Every policy decision is traceable. The agent doesn't know the proxy exists.

## Next steps

- [Writing Policies](writing-policies.md) — per-agent rules, globs, conditions
- [Approval Flow](approval-flow.md) — approval queue, grants, CLI
- [Deployment Modes](deployment-modes.md) — solo dev, team, Managed Agents

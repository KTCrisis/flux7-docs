# Getting Started

Install flux7-mesh, put it in front of Claude Code, and watch one call go
through, one call wait for you, and the trail they leave. About five minutes.

The other products come after, each on its own. None is needed to start.

## 1. Install

=== "Script (Linux, macOS)"

    ```bash
    curl -fsSL https://raw.githubusercontent.com/KTCrisis/flux7-mesh/main/install.sh | sh
    ```

    Installs `mesh7` (the proxy) and `mesh` (the approval CLI) in `~/.local/bin`. Options such as `--version` or `--service`: [Deployment modes](mesh7/deployment-modes.md#running-mesh7-as-a-service).

=== "Go install"

    ```bash
    go install github.com/KTCrisis/flux7-mesh/cmd/mesh7@latest
    go install github.com/KTCrisis/flux7-mesh/cmd/mesh@latest
    ```

=== "Binary"

    ```bash
    curl -L https://github.com/KTCrisis/flux7-mesh/releases/latest/download/mesh7_linux_amd64.tar.gz | tar xz
    sudo mv mesh7 mesh /usr/local/bin/
    ```

    Other targets: `mesh7_{linux,darwin}_{amd64,arm64}.tar.gz`, `mesh7_windows_{amd64,arm64}.zip`.

```bash
mesh7 --version
```

## 2. Write a policy

```yaml title="config.yaml"
trace_file: traces.jsonl

mcp_servers:
  - name: filesystem
    transport: stdio
    command: npx
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/home/me/projects"]

policies:
  - name: claude
    agent: "claude"
    rules:
      - tools: ["filesystem.read_*", "filesystem.list_*"]
        action: allow
      - tools: ["filesystem.write_file", "filesystem.edit_file"]
        action: human_approval

  - name: default
    agent: "*"
    rules:
      - tools: ["*"]
        action: deny
```

Reads are allowed, writes wait for a human, everything else is refused. Rules
are read top to bottom and the first match wins.

To start from your own servers instead, `mesh7 discover --config config.yaml
--generate-policy` writes a commented draft.

## 3. Put it in front of Claude Code

```bash
claude mcp add mesh7 -- mesh7 --mcp --config config.yaml
```

Restart Claude Code. The filesystem tools are there, now behind the policy.

- Ask it to **read a file**: it goes through.
- Ask it to **write a file**: the call is held. Claude relays an approval id.
  In another terminal:

    ```bash
    mesh pending
    mesh approve <id>
    ```

    Claude retries the same call, and it goes through.

## 4. Look at the trail

```bash
tail -n 3 traces.jsonl                 # one line per call: agent, tool, decision, rule
mesh7 trace verify traces.jsonl        # the lines are hash-chained: an edit breaks the chain
```

The held write's line records the approval: its id, its outcome and who gave
it. The trail says not only what happened, but who allowed it.

## Without Claude Code

Any agent or script can go through the mesh over HTTP:

```bash
mesh7 serve --config config.yaml

curl -X POST http://localhost:9090/tool/filesystem.read_file \
  -H "Authorization: Bearer agent:my-script" \
  -H "Content-Type: application/json" \
  -d '{"params":{"path":"/home/me/projects/README.md"}}'
```

`my-script` matches no named policy, so the default one applies and the call
is refused. Add a policy for it to let it through. For real identities, use
[JWTs from your IdP](mesh7/jwt-auth.md).

---

## Add the other products

Each is a separate install, and each works without the others.

### flux7-memory: precedents and agent memory

```bash
go install github.com/KTCrisis/flux7-memory/cmd/mem7@latest
MEM7_TOKEN=change-me mem7 serve --listen :9070
```

On its own, it is a memory server for any MCP client. Declared in the mesh
config, it also remembers human approvals: a read you approved three times,
never refused, stops being asked. [flux7-memory](mem7/index.md) ·
[how precedents work](mesh7/mem7-auto-approve.md)

### flux7-supervisor: automated evaluation

```bash
pip install "git+https://github.com/KTCrisis/flux7-supervisor"
sup7 -c sup7.yaml start
```

Pointed at the mesh, it settles the approval queue: rules first, then a
decision model, and it escalates to you when it is not confident. On its own,
it answers `POST /evaluate` for any hook or gateway. [flux7-supervisor](sup7/index.md) ·
[configuration](sup7/configuration.md)

### flux7-console: dashboard and approval UI

```bash
git clone https://github.com/KTCrisis/flux7-console
cd flux7-console/frontend && npm install && npm run dev    # http://localhost:3000
```

Approvals from a browser, traces with their chain of authority, policies,
grants, memory and supervisor state. It reads the mesh on `localhost:9090`.
[flux7-console](console/index.md)

## Next

- [How it works](how-it-works.md): the layers a call goes through
- [Writing policies](mesh7/writing-policies.md): conditions on arguments, per-agent files
- [Approval flow](mesh7/approval-flow.md): grants, the approval CLI, waiting for the supervisor
- [Deployment modes](mesh7/deployment-modes.md): running the mesh as a service, for a team

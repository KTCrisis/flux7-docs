# Provenance and agent scopes

Since 03/10/2026, mem7 records which governed call wrote each memory, and can
restrict what each agent reads, from identities that flux7-mesh vouches for.

## What the mesh sends

When mem7 sits behind flux7-mesh as a `streamable-http` upstream, the mesh adds
a `_meta` to each `tools/call`:

| Key | Content | Sent |
|---|---|---|
| `traceparent` | W3C trace context of the governed call | always |
| `art.flux7/agent` | the agent the mesh authenticated | with `forward_identity: true` on the upstream |

## Provenance

mem7 records the trace id on every write and every deletion: in the markdown
workspace (`trace:` in the entry's envelope), which stays the source of truth,
and in the index, rebuilt with it by `mem7 rescan`. `memory_recall` and
`memory_search` print it, `memory_context` returns it as `trace_id`. A memory
points back to the decision that produced it in the mesh's traces.

## Identity

Anyone can write a `_meta`. mem7 therefore honours `art.flux7/agent` **only on
a request that carried its bearer token** (`MEM7_TOKEN`), which only the mesh
and trusted infrastructure hold. The vouched identity replaces the `agent`
argument the caller declared. Without a token, the trace id is still recorded
but no request can speak for an agent.

## Scopes

A scopes file (`MEM7_SCOPES` or `--scopes`, JSON) restricts identified agents:

```json
{
  "read":  { "claude": ["*"], "sup7": ["scout7"] },
  "admin": ["claude"]
}
```

| Caller | Reads | Writes and forgets |
|---|---|---|
| identified agent | its own memories, plus the owners listed under `read` (glob patterns) | its own keys only; no forget by tags; no `memory_get` (raw workspace) |
| administrator | everything | everything |
| token, no agent (supervisor, console, the mesh's decision writer) | everything | everything |

Without a scopes file, identities still sign the writes and reads stay open.

## Wiring

```yaml
# flux7-mesh
memory:
  url: http://localhost:9070
  token: "${MEM7_TOKEN}"
mcp_servers:
  - name: memory
    transport: streamable-http
    url: http://localhost:9070/mcp
    forward_identity: true
    headers:
      Authorization: "Bearer ${MEM7_TOKEN}"
```

Every direct client then needs the token: the supervisor reads `MEM7_TOKEN`
from its environment when `memory.token` is empty, the console reads it from
its environment file. `contrib/systemd/enable-token.sh` in flux7-memory wires
the token and the scopes into the systemd units of a single machine.

!!! note "Tool pinning"
    The change adds an `agent` property to `memory_forget`. A mesh with tool
    pinning on holds the tool until the new definition is accepted on the
    console's Tools page: the protection against a silently changed tool works
    as intended.

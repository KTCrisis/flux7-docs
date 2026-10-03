# Provenance, tamper evidence and agent scopes

Since 03/10/2026, mem7 records which governed call wrote each memory, seals
every entry in a hash chain, and can restrict what each agent reads, from
identities that flux7-mesh vouches for.

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

## Hash chain (tamper evidence)

Every entry mem7 writes to the workspace (store, deletion, deletion by tags) carries the seal of the entry before it (`prev:`) and its own (`hash:`), computed over its parsed fields. Edit an entry, drop one or reorder them, and `mem7 verify` names the first place the chain no longer holds:

```
$ MEM7_CHAIN_KEY=... mem7 verify
211 entries: 78 sealed, 133 written before the chain
seals: HMAC-SHA256 with MEM7_CHAIN_KEY
chain holds
```

With `MEM7_CHAIN_KEY` the seal is an HMAC-SHA256: editing the workspace without the key leaves a break no one can reseal. Without it, a plain SHA-256 catches accidents and careless edits, not a forger. Keep the key stable: entries sealed with one key do not verify with another. Entries written before the chain existed are counted, not checked; the chain starts at the first sealed entry. The workspace stays plain markdown you can read and edit by hand; an edit now shows.

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

# Python SDK

Provider-agnostic Python client wrapping all MCP tools via JSON-RPC over HTTP.

## Install

```bash
pip install flux7-memory
```

!!! warning "PyPI package name"
    The package is `flux7-memory`, not `mem7`. `pip install mem7` installs a different, unrelated package.

Or from source :

```bash
pip install ./sdk/python
```

## Quick start

```python
from mem7 import Mem7

m = Mem7("http://localhost:9070", token="my-token")

# Store a memory
m.store("deploy.decision", "approved by ops lead, prod deploy greenlit",
        tags=["decision", "deploy"], agent="supervisor")

# Search (returns formatted text)
print(m.search("deployment approval", limit=5))

# Context (returns structured Memory objects)
for mem in m.context("deployment approval", limit=5):
    print(f"{mem.key}: {mem.value}")
```

## API

Every argument after the first positional one is keyword-only.

### `Mem7(url, token=None, timeout=30)`

Create a client connected to a running `mem7 serve` instance. `token=None` reads `MEM7_TOKEN` from the environment; `""` sends none. `timeout` is in seconds. Errors raise `Mem7Error`.

Moments (`valid_from`, `valid_to`, `valid_at`, `as_of`) accept a `date` (midnight UTC), a timezone-aware `datetime`, or a string mem7 parses (`"2026-03-20"`, RFC3339). They need mem7 0.8; see [Time](time.md).

### `health()`

`True` when `/healthz` answers 200, `False` otherwise (never raises).

### `store(key, value, *, tags=None, agent="", ttl=0, valid_from=None, valid_to=None)`

Record a new version of a memory. `valid_from` / `valid_to` say when it holds in the world: by default from now, open-ended; a past `valid_from` corrects history without erasing what was believed.

```python
m.store("user.prefs", "prefers dark mode", tags=["user", "ui"])
m.store("event7.host", "Cloudflare", valid_from="2026-03-20")
```

### `search(query, *, mode="natural", tags=None, agent="", limit=10, include_neighbors=False, neighbor_radius=1, since="", until="", valid_at=None, as_of=None)`

Full-text search (BM25), returns formatted markdown text. The SDK defaults to `mode="natural"` (stop words stripped, OR-joined: suited to agent questions); `mode="raw"` keeps FTS5 operators (`foo*`, `AND`, `OR`, `NOT`), and is the default of the MCP tool itself. `include_neighbors` expands hits on sequential keys (`conv.session.t005` brings `t004` and `t006`, `neighbor_radius` on each side). `since` / `until` bound `updated_at` (RFC 3339).

```python
results = m.search("dark mode", limit=5)
results = m.search("deploy*", mode="raw", since="2026-09-01T00:00:00Z")
```

### `context(query, *, mode="natural", tags=None, agent="", limit=10, ...)`

Same as `search` but returns a list of `Memory` objects with structured fields :

```python
for mem in m.context("deploy*", limit=10):
    print(mem.key)       # "deploy.decision"
    print(mem.value)     # "approved by ops lead"
    print(mem.tags)      # ["decision", "deploy"]
    print(mem.agent)     # "supervisor"
    print(mem.updated)   # "2026-05-09T10:30:00Z"
    print(mem.trace_id)  # trace of the governed call that wrote it (mem7 0.6+)
    print(mem.valid_from, mem.valid_to, mem.tx_from, mem.tx_to)  # mem7 0.8+, None = open
```

Like every read, `context` takes `valid_at` (what held then) and `as_of` (what mem7 believed then):

```python
m.context("event7 host", valid_at="2026-03-01")            # where was it on March 1st
m.context("event7 host", as_of="2026-04-11", valid_at="2026-04-08")  # as believed on the 11th
```

### `context_block(query, limit=10, **kwargs)`

Returns a pre-formatted text block for LLM prompt injection :

```python
block = m.context_block("user preferences", limit=10)
# Inject into system prompt or context window
```

### `recall(*, key="", tags=None, agent="", limit=10, valid_at=None, as_of=None)`

Recall by key, tags, or agent. Bumps access tracking.

```python
m.recall(key="deploy.decision")
m.recall(tags=["decision"], limit=5)
```

### `list(*, tags=None, agent="", valid_at=None, as_of=None)`

List keys with metadata (without values).

```python
entries = m.list(tags=["decision"])
```

### `get(path, *, from_line=0, to_line=0)`

Read a workspace file.

```python
content = m.get("memory/2026-05-09.md")
```

### `forget(*, key="", tags=None, agent="")`

Stop believing a key and/or every key with these tags. Appends a signed tombstone to the markdown workspace; a read `as_of` an earlier moment still sees what was forgotten.

```python
m.forget(key="user.prefs", agent="claude")
m.forget(tags=["temp"])
```

### `history(key)`

The life of one key, oldest first, as a list of `HistoryEvent` (`when`, `what`, `agent`, `trace`, `seal`, `valid`). mem7 0.7+.

```python
for ev in m.history("event7.host"):
    print(ev.when, ev.what, ev.agent, ev.valid, ev.trace, ev.seal)
```

### `chain()`

The workspace's hash chain report, as `mem7 verify` prints it: `{"holds": bool, "report": {"entries", "sealed", "legacy", "keyed", "break"}}`. mem7 0.7+.

## Usage in multi-agent systems

```python
from mem7 import Mem7

m = Mem7("http://localhost:9070", token="shared-token")

# Research agent stores findings
m.store("research.competitors", "Found 3 competitors with similar features",
        tags=["research"], agent="researcher")

# Execution agent checks decisions before acting
decisions = m.context("deployment approved", tags=["decision"], limit=5)
if any("approved" in d.value for d in decisions):
    proceed_with_deploy()

# Supervisor stores policy decisions
m.store("policy.deploy", "requires 2 approvals before prod deploy",
        tags=["policy", "decision"], agent="supervisor")
```

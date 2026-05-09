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

### `Mem7(url, token=None)`

Create a client connected to a running `mem7 serve` instance.

### `store(key, value, tags=None, agent=None, ttl=0)`

Upsert a memory entry.

```python
m.store("user.prefs", "prefers dark mode", tags=["user", "ui"])
```

### `search(query, limit=10, mode="raw", tags=None, agent=None)`

Full-text search, returns formatted markdown text.

```python
results = m.search("dark mode", limit=5, mode="natural")
```

### `context(query, limit=10, **kwargs)`

Same as `search` but returns a list of `Memory` objects with structured fields :

```python
for mem in m.context("deploy*", limit=10):
    print(mem.key)       # "deploy.decision"
    print(mem.value)     # "approved by ops lead"
    print(mem.tags)      # ["decision", "deploy"]
    print(mem.agent)     # "supervisor"
    print(mem.updated)   # "2026-05-09T10:30:00Z"
```

### `context_block(query, limit=10, **kwargs)`

Returns a pre-formatted text block for LLM prompt injection :

```python
block = m.context_block("user preferences", limit=10)
# Inject into system prompt or context window
```

### `recall(key=None, tags=None, agent=None, limit=10)`

Recall by key, tags, or agent. Bumps access tracking.

```python
m.recall(key="deploy.decision")
m.recall(tags=["decision"], limit=5)
```

### `list(tags=None, agent=None)`

List keys with metadata (without values).

```python
entries = m.list(tags=["decision"])
```

### `get(path, from_line=None, to_line=None)`

Read a workspace file.

```python
content = m.get("memory/2026-05-09.md")
```

### `forget(key=None, tags=None, agent=None)`

Delete by key and/or tags. Appends a tombstone to the markdown workspace.

```python
m.forget(key="user.prefs")
m.forget(tags=["temp"])
```

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

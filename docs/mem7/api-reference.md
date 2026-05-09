# API Reference

## MCP tools

flux7-memory exposes 7 MCP tools, available over stdio, HTTP JSON-RPC, and MCP SSE.

### memory_store

Upsert a memory entry by key. The markdown workspace receives an append-only section ; the SQLite index is updated in place.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `key` | string | yes | Unique key for this memory |
| `value` | string | yes | Content to remember (free-form markdown) |
| `tags` | string[] | no | Tags for filtering and grouping |
| `agent` | string | no | Identifier of the storing agent |
| `ttl` | number | no | Time-to-live in seconds (0 = permanent) |

### memory_recall

Recall memories by key, tags, or agent. Most recently updated first. Bumps `access_count` and `last_accessed` on returned entries.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `key` | string | no | Exact key to recall |
| `tags` | string[] | no | Filter by tags (AND logic) |
| `agent` | string | no | Filter by agent |
| `limit` | number | no | Max results (default 10) |

### memory_search

Full-text search using SQLite FTS5 with field-weighted BM25. When hybrid search is enabled, results are merged with dense cosine similarity via RRF.

Supports FTS5 operators in raw mode : `foo*` prefix, `AND` / `OR` / `NOT`, quoted phrases.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | yes | Search query |
| `mode` | string | no | `raw` (default, FTS5 syntax) or `natural` (plain language, auto-stemmed) |
| `tags` | string[] | no | Post-filter by tags |
| `agent` | string | no | Post-filter by agent |
| `since` | string | no | Lower bound on `updated_at` (RFC3339) |
| `until` | string | no | Upper bound on `updated_at` (RFC3339) |
| `limit` | number | no | Max results (default 10) |
| `include_neighbors` | boolean | no | Fetch sequential neighbors (default false) |
| `neighbor_radius` | number | no | Neighbors on each side (default 1) |

### memory_context

Same search as `memory_search` but returns a JSON array of structured objects instead of formatted markdown. Designed for programmatic use by agent SDKs.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | yes | Search query |
| `mode` | string | no | `raw` (default) or `natural` |
| `tags` | string[] | no | Post-filter by tags |
| `agent` | string | no | Post-filter by agent |
| `since` | string | no | Lower bound on `updated_at` (RFC3339) |
| `until` | string | no | Upper bound on `updated_at` (RFC3339) |
| `limit` | number | no | Max results (default 10) |
| `include_neighbors` | boolean | no | Fetch sequential neighbors (default false) |
| `neighbor_radius` | number | no | Neighbors on each side (default 1) |

Returns :

```json
[
  {
    "key": "deploy.decision",
    "value": "approved by ops lead",
    "tags": ["decision", "deploy"],
    "agent": "supervisor",
    "updated": "2026-05-09T10:30:00Z"
  }
]
```

### memory_get

Read a file from the markdown workspace. Paths are resolved relative to the workspace root and refused if they escape it.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | yes | Workspace-relative path (e.g. `memory/2026-04-11.md`) |
| `from_line` | number | no | First line to read (1-indexed) |
| `to_line` | number | no | Last line to read (1-indexed) |

### memory_list

List memory keys with metadata (without values).

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tags` | string[] | no | Filter by tags |
| `agent` | string | no | Filter by agent |

### memory_forget

Delete memories by key and/or tags. A tombstone section is appended to the markdown workspace, and the SQLite index soft-deletes the matching rows.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `key` | string | no | Exact key to delete |
| `tags` | string[] | no | Delete all entries matching these tags (AND logic) |
| `agent` | string | no | Recorded on the tombstone |

## HTTP endpoints

`mem7 serve` exposes these routes :

| Method | Path | Description |
|--------|------|-------------|
| `GET`  | `/healthz` | Liveness probe (always public, no auth) |
| `POST` | `/rpc` | JSON-RPC 2.0 — same tool surface as stdio |
| `GET`  | `/sse` | MCP SSE transport (for flux7-mesh daemon mode) |
| `POST` | `/messages` | MCP SSE message endpoint |
| `POST` | `/memory/snapshot_reminder` | Instructional payload for pre-compaction context injection |

Bearer auth is applied to `/rpc` and `/memory/*` when `MEM7_TOKEN` is set.

### Example : JSON-RPC call

```bash
curl -s -X POST http://localhost:9070/rpc \
  -H "Authorization: Bearer $MEM7_TOKEN" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call",
       "params":{"name":"memory_search","arguments":{"query":"roadmap*"}}}'
```

## Architecture

```
      Claude Code / flux7-mesh / Python SDK
                    │
          MCP stdio ┴ HTTP JSON-RPC ┴ MCP SSE
                    │
              ┌─────▼─────┐
              │ Dispatcher │   ← MCP protocol layer
              └─────┬─────┘
                    │
              ┌─────▼─────┐
              │   Store    │   ← orchestrator
              └──┬──┬──┬──┬┘
                 │  │  │  │
          ┌──────▼┐ │ ┌▼──────────┐ ┌▼─────────┐
          │markdown│ │ │ sqlite    │ │ reranker  │
          │workspace│ │ │ (facts +  │ │ (Ollama)  │
          │(truth) │ │ │ FTS5 +    │ │ opt-in    │
          └────────┘ │ │ embeds)   │ └───────────┘
                     │ └───────────┘
              ┌──────▼──────┐
              │  embedder   │  ← opt-in, external
              │ (Ollama /   │
              │  OpenAI)    │
              └─────────────┘
```

Every write goes through the markdown writer first, then updates the SQLite index. Reads consult the index only. Embeddings are cached in memory for sub-ms cosine search. If the index is corrupted, `mem7 rescan` drops it and replays the markdown to reconstruct a consistent state.

# Configuration

## Environment variables

| Variable | Default | Description |
|----------|---------|-------------|
| `MEM7_DIR` | `~/.mem7` | Data directory (hosts `workspace/` and `index.db`) |
| `MEM7_LISTEN` | `:9070` | HTTP bind address in `serve` mode |
| `MEM7_TOKEN` | *(empty)* | Bearer token for `/rpc` and `/memory/*` |
| `MEM7_MAX_ENTRIES` | `10000` | Soft ceiling on live entries |
| `MEM7_EMBED_URL` | *(empty)* | Embedding provider base URL. Enables hybrid search |
| `MEM7_EMBED_MODEL` | `nomic-embed-text` | Model name for the embedding API |
| `MEM7_EMBED_PROVIDER` | `ollama` | `ollama` (POST `/api/embed`) or `openai` (POST `/v1/embeddings`) |
| `MEM7_EMBED_KEY` | *(empty)* | Bearer token for the embedding API |
| `MEM7_RERANK_URL` | *(empty)* | Reranking LLM base URL. Enables LLM reranking after RRF merge |
| `MEM7_RERANK_MODEL` | `gemma4:e4b` | Model name for the Ollama generate API |

Flags on `mem7 serve` mirror `MEM7_LISTEN` and `MEM7_TOKEN` :

```bash
mem7 serve --listen :9070 --token mem7_secret123
```

## Hybrid search

Entirely opt-in. Without `MEM7_EMBED_URL`, mem7 uses pure BM25.

When enabled, `memory_store` computes and persists an embedding alongside each entry. `memory_search` retrieves BM25 top-2N and cosine top-2N candidates, then merges them via Reciprocal Rank Fusion (RRF, k=60) into the final top-N. Embeddings are stored as BLOBs in SQLite and cached in memory for sub-ms cosine search.

### With local Ollama

```bash
MEM7_EMBED_URL=http://localhost:11434 \
MEM7_EMBED_MODEL=nomic-embed-text \
  mem7 serve --listen :9070
```

### With OpenAI API

```bash
MEM7_EMBED_URL=https://api.openai.com \
MEM7_EMBED_MODEL=text-embedding-3-small \
MEM7_EMBED_PROVIDER=openai \
MEM7_EMBED_KEY=sk-... \
  mem7 serve --listen :9070
```

### With any OpenAI-compatible endpoint

vLLM, LiteLLM, Azure OpenAI, etc. :

```bash
MEM7_EMBED_URL=http://localhost:8000 \
MEM7_EMBED_MODEL=BAAI/bge-small-en-v1.5 \
MEM7_EMBED_PROVIDER=openai \
  mem7 serve --listen :9070
```

## LLM reranking

Opt-in on top of hybrid search. Over-fetches 3x candidates, merges via RRF, then uses an LLM to score relevance before returning the final top-N. Falls back to non-reranked results if the LLM is unavailable.

```bash
MEM7_EMBED_URL=http://localhost:11434 \
MEM7_RERANK_URL=http://localhost:11434 \
MEM7_RERANK_MODEL=gemma4:e4b \
  mem7 serve --listen :9070
```

## Workspace layout

```
~/.mem7/
├── workspace/
│   ├── MEMORY.md                      # reserved for long-term notes
│   └── memory/
│       ├── 2026-04-11.md              # append-only daily logs
│       └── 2026-04-12.md
└── index.db                           # SQLite (facts + facts_fts + embeddings)
```

The markdown files are the source of truth. `index.db` is a derived cache that can be dropped and rebuilt at any time via `mem7 rescan`.

Each entry is written as a level-2 heading followed by a fenced `mem7` envelope and a free-form body :

````markdown
## example_key

```mem7
op: store
agent: claude
tags: demo, example
created: 2026-04-11T20:00:00Z
updated: 2026-04-11T20:00:00Z
```

Free-form markdown content lives here.

---
````

## Usage with flux7-mesh

In your mesh `config.yaml` :

```yaml
mcp_servers:
  - name: memory
    transport: stdio
    command: /home/user/go/bin/mem7
    env:
      MEM7_DIR: /home/user/.mem7
```

flux7-mesh discovers the tools via `tools/list`. Grants and policies apply as usual.

To share memory across machines, run `mem7 serve` on one host and connect via SSE or HTTP JSON-RPC.

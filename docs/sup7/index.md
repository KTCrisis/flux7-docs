# flux7-supervisor — L1 Evaluation Agent

## The problem

Your AI agents make hundreds of tool calls per session. [flux7-mesh](../agent-mesh/index.md) enforces policy on every call — but some actions land in a grey zone. The policy says `human_approval`, and a human stares at a terminal prompt:

> *Allow filesystem.write_file to /home/user/project/src/main.py? [y/n]*

For the 47th time today. Same agent, same directory, same pattern. The human approves mechanically, attention already elsewhere.

Meanwhile, an agent sends an email to an external address. Same `human_approval` policy. The human, deep in approval fatigue, hits `y` without reading.

The problem isn't the policy. The problem is that routine and risky look the same to a queue.

## What flux7-supervisor is

A standalone agent that sits between policy (L0) and human (L2). It evaluates pending approvals using rules and an LLM, auto-resolving the routine ones so humans only see what actually needs judgment.

```
flux7-mesh (L0)              sup7 (L1)                    Human (L2)
policy: human_approval  ──►  rules + LLM evaluate    ──►  only ambiguous cases
                              │                            │
                              ├─ approve (routine)         ├─ approve/deny
                              ├─ deny (dangerous)          │
                              └─ escalate (unsure)  ───────┘
```

Install and run :

```bash
pip install flux7-supervisor
sup7 start --config sup7.yaml
```

It polls flux7-mesh for pending approvals, evaluates each one, and resolves. Decisions are logged to JSONL and stored in [flux7-memory](../mem7/index.md) as queryable facts.

## Three-level approval flow

| Level | Component | Speed | Judgment | Example |
|-------|-----------|-------|----------|---------|
| **L0** | flux7-mesh policy | instant | none — static rules | `allow` reads, `deny` deletes |
| **L1** | flux7-supervisor | seconds | bounded — rules + LLM | project writes → approve, unknown tool → escalate |
| **L2** | Human (terminal or UI) | minutes | full | external email, ambiguous intent |

The supervisor reduces L2 load by handling the predictable cases. Over time, as decisions accumulate in flux7-memory, patterns emerge and the supervisor gets more confident.

## Pluggable LLM providers

The evaluation brain is configurable. Choose based on your constraints :

| Provider | Transport | Latency | Cost | Context |
|----------|-----------|---------|------|---------|
| **Ollama** | HTTP to local model | ~1s | free | tool name + params only |
| **Anthropic** | Claude Messages API | ~2s | per-token | tool name + params only |
| **Claude Code** | MCP callback | async | per-session | full codebase + conversation |

The Claude Code provider is the most interesting : instead of calling an API, the supervisor queues the evaluation and exposes it as an MCP tool. Claude Code — already connected to flux7-mesh — sees the pending evaluation, reviews it with full codebase context, and submits its verdict. The supervisor acts on it.

## How it works

```
                           sup7
                      ┌────────────┐
 flux7-mesh           │  poll loop │           flux7-memory
 GET /approvals ◄─────│            │──────►    store decision
     ?status=pending  │  rules     │           (decisions as facts)
                      │    ↓       │
 POST /approvals/     │  LLM eval  │
   {id}/approve  ◄────│    ↓       │
   {id}/deny     ◄────│  resolve   │
                      └────────────┘
```

1. **Poll** — `GET /approvals?status=pending` via mesh7 SDK, deduplicated across tool scopes
2. **Fetch context** — `GET /approvals/{id}` returns params, recent traces, active grants, injection risk
3. **Evaluate rules** — YAML conditions, first-match-wins, with confidence scores
4. **LLM fallback** — if no rule matches and an LLM provider is configured, delegate evaluation
5. **Confidence gate** — if LLM confidence is below threshold, escalate to human
6. **Resolve** — `POST /approvals/{id}/approve` or `/deny` with reasoning and confidence
7. **Log** — JSONL file + flux7-memory store (tagged `supervisor`, `decision`)

## Current state (May 2026)

- **v0.1.0** — 3 providers, rule engine, MCP server for Claude Code callback, 49 tests
- **SDKs** — consumes `mesh7` (AgentMesh) and `mem7` (Mem7) Python SDKs
- **Extracted** from flux7-console backend, now standalone

MIT licensed. [github.com/KTCrisis/flux7-supervisor](https://github.com/KTCrisis/flux7-supervisor)

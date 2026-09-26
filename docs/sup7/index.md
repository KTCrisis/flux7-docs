# flux7-supervisor — L1 Evaluation Agent

## The problem

Your AI agents make hundreds of tool calls per session. [flux7-mesh](../mesh7/index.md) enforces policy on every call — but some actions land in a grey zone. The policy says `human_approval`, and a human stares at a terminal prompt:

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
pip install "git+https://github.com/KTCrisis/flux7-supervisor"
# with the Anthropic provider
pip install "flux7-supervisor[anthropic] @ git+https://github.com/KTCrisis/flux7-supervisor"

sup7 -c sup7.yaml status   # check mesh and memory connectivity
sup7 -c sup7.yaml start    # start the poll loop
```

sup7 is not published on PyPI; install it from the repository. `-c` (default `sup7.yaml`) and `-v` are top-level options, placed before the subcommand.

It polls flux7-mesh for pending approvals, evaluates each one, and resolves. Decisions are logged to JSONL and, when `memory.enabled` is set (off by default), stored in [flux7-memory](../mem7/index.md) as queryable facts.

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
| **Ollama** | HTTP to local model | ~1s | free | tool, params, 5 recent traces, active grants |
| **Anthropic** | Claude Messages API | ~2s | per-token | tool, params, 5 recent traces, active grants |
| **Jev** (TypeSafe AI) | Cloudflare Workers AI or TypeSafe API | not measured | per-call | same context, `redact_params` withheld; typed answers with probabilities |
| **Claude Code** | MCP callback | async | per-session | full codebase + conversation (not functional yet) |

Jev does not generate text: sup7 asks it four typed questions (decision, destructive, in scope, injection) and combines the probabilities in code, fail-closed. The probabilities are written into the decision reasoning.

The Claude Code provider is designed so that, instead of calling an API, the supervisor queues the evaluation and exposes it as an MCP tool for Claude Code to review with full codebase context. In the current release the MCP server is not started, so this provider times out and escalates; see [Claude Code Callback](claude-code-callback.md).

### Provider chain

Providers can be chained under `evaluator.chain`. They are tried in order; the first one that answers gives the verdict (an `escalate` is an answer). The next one is tried only on failure: network or HTTP error, timeout, unreadable answer. A circuit breaker skips a provider after `breaker_failures` consecutive failures (default 3) for `breaker_cooldown` seconds (default 300). If every provider fails, sup7 escalates to a human. The reasoning records who decided, e.g. `[ollama, jev skipped] ...`. See [Configuration](configuration.md#provider-chain).

## Admin API

When `admin.enabled` is set (off by default), sup7 serves a small HTTP API in-process, on `127.0.0.1:9096` by default. [flux7-console](../console/index.md) uses it to show the supervisor state and to pause or resume it.

| Route | Does |
|-------|------|
| `GET /health` | liveness (no token) |
| `GET /status` | running or paused, mesh reachability, decision counters, state of each provider (ok, failing, skipped) |
| `GET /config` | rules, thresholds, provider chain; never secrets |
| `GET /decisions?limit=50` | most recent decisions with their reasoning (kept in memory, 200 by default) |
| `POST /pause` | stop evaluating: approvals stay pending in the mesh, for a human |
| `POST /resume` | evaluate again |

When `admin.token` is set, every route except `/health` requires `Authorization: Bearer <token>`. Set a token before binding to anything other than loopback.

## Run as a service

A systemd unit ships in `contrib/systemd/sup7.service` (`After=`/`Wants=mesh7.service`, `Restart=on-failure`):

```bash
sudo cp contrib/systemd/sup7.service /etc/systemd/system/
sudo systemctl daemon-reload && sudo systemctl enable --now sup7
```

Adapt `User=` and the paths in `ExecStart=` first. sup7 waits for the mesh on its own, so start order is not critical. Pair it with `approval.channel: queue` in the mesh config: a service has no TTY to prompt on.

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
7. **Log** — JSONL file, plus a flux7-memory store (tagged `supervisor`, `decision`) when memory is enabled

While paused through the admin API, the loop does not poll: approvals stay pending in the mesh for a human.

## Current state (September 2026)

- **v0.1.0** — 4 providers (Ollama, Anthropic, Jev, Claude Code) and a provider chain with circuit breaker, rule engine, HTTP admin API, systemd unit, 85 tests
- **Claude Code callback**: MCP tools defined but the MCP server is not started yet
- **Install**: from the GitHub repository, not on PyPI
- **SDKs** — consumes `mesh7` (AgentMesh) and `mem7` (Mem7) Python SDKs
- **Extracted** from flux7-console backend, now standalone

Apache 2.0 licensed. [github.com/KTCrisis/flux7-supervisor](https://github.com/KTCrisis/flux7-supervisor)

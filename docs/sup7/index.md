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
| **L1** | flux7-supervisor | 0 ms (rules) to ~500 ms (decision model) | bounded — rules, then typed questions | project writes → approve, unknown tool → escalate |
| **L2** | Human (terminal or UI) | minutes | full | external email, ambiguous intent |

The supervisor reduces L2 load by handling the predictable cases. Over time, as decisions accumulate in flux7-memory, patterns emerge and the supervisor gets more confident.

## Pluggable LLM providers

The evaluation brain is configurable. Choose based on your constraints :

| Provider | Transport | Latency | Cost | Context |
|----------|-----------|---------|------|---------|
| **Ollama** | HTTP to local model | ~1s | free | tool, params, 5 recent traces, active grants |
| **Anthropic** | Claude Messages API | ~2s | per-token | tool, params, 5 recent traces, active grants |
| **Jev** (TypeSafe AI) | Cloudflare Workers AI or TypeSafe API | ~330 ms (p95 ~440 ms) | ~$0.00004 per call | same context plus `project_dirs`, `redact_params` withheld; typed answers with probabilities |
| **Claude Code** | MCP callback | async | per-session | full codebase + conversation (not functional yet) |

Jev does not generate text and is not asked to decide: sup7 asks it narrow factual questions (does the call delete, overwrite outside the project, send local data out, touch secrets; where does it act; does it fit the agent's activity; does it carry an injection) and decides in code, with a threshold per question, fail-closed. The questions are YAML, extensible with business packs; each decision records the model, a fingerprint of the questions and the thresholds. See [Jev and question sets](jev.md), and [Measuring](measuring.md) for how the thresholds were chosen.

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
| `POST /evaluate` | judge one tool call on demand, outside the mesh queue (see below) |
| `GET /files`, `GET /files/{id}` | the editable files (sup7.yaml, question sets) as text, token values masked |
| `PUT /files/{id}` | replace one: validated whole, backed up, applied without restart where possible |
| `GET /bench/sets`, `/bench/runs`, `/bench/estimate` | labelled case sets, evaluation runs, cost of a replay |
| `POST /bench/runs` | measure the live configuration on a case set (free recompute or paid replay) |

When `admin.token` is set, every route except `/health` requires `Authorization: Bearer <token>`. Editing a file and starting a run always require it, even on loopback. Set a token before binding to anything other than loopback.

### Judging a call on demand: `POST /evaluate`

sup7 also works without the mesh queue, as a decision service for any enforcement point: a Claude Agent SDK `PreToolUse` hook, a gateway plugin, another agent framework's guardrail.

```bash
curl -s -X POST localhost:9096/evaluate -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"agent_id": "bot", "tool": "Bash", "params": {"command": "rm -rf ~/.ssh"}}'
```
```json
{"id": "eval-…", "decision": "deny", "confidence": 0.99, "rule_matched": "jev:cloudflare",
 "reasoning": "[jev] Jev jev-1.13.0 q=…: deny (destructive 0.99 (deletes 0.99 · … · secrets 0.95) …)",
 "evaluator": {"model": "jev-1.13.0", "questions": "…", "thresholds": {…}}, "evaluation_ms": 339}
```

The same rules, provider chain, questions and thresholds as a polled approval; `recent_traces` (up to 5), `active_grants`, `injection_risk` and `policy_rule` are optional context. Measured in production: a test run in the project approved, `rm -rf ~/.ssh` and a credentials upload denied, in about 330 ms; a read settled by a rule without calling the model.

sup7 **advises, the caller enforces**: it blocks a `deny` and sends an `escalate` to its own humans. Nothing is resolved in any mesh; the decision is logged with `"via": "evaluate"`, and a paused sup7 answers `escalate`. Used alone, sup7 brings the judgment; [flux7-mesh](../mesh7/index.md) brings what surrounds it: enforcement, signed traces, the human queue, grants, precedents and the approval wait.

### Editing from the console

`PUT /files/{id}` takes the text of `sup7.yaml` or of a question set, with `If-Match: <fingerprint>` (`new` to create a question set). A file changed on disk since it was read is not overwritten (409). The whole resulting configuration is validated first (schema, rule conditions, every question set with the edit in place): a bad edit answers 400 with the reason and nothing is written. The previous version is backed up next to the file (never overwriting an earlier backup, file mode kept), the write is atomic, and rules, evaluator, thresholds, questions, project dirs and poll interval apply at once; sections read only at start (`mesh`, `memory`, `admin`) are reported as `restart_required`. Each change is a `config_change` event in the decision log.

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
4. **Evaluator** — if no rule matches, the provider (or chain) evaluates; `provider: none` keeps sup7 to its rules
5. **Confidence gate** — below the threshold of the provider that answered (a chain entry may set its own), escalate to human
6. **Resolve** — `POST /approvals/{id}/approve` or `/deny` with reasoning and confidence
7. **Log** — JSONL file with the evaluator's provenance (model, question fingerprint, thresholds), plus a flux7-memory store (tagged `supervisor`, `decision`) when memory is enabled

Paired with mesh7's [`approval.wait_seconds`](../mesh7/approval-flow.md#waiting-for-an-automatic-decision-approvalwait_seconds) and a fast `poll.interval` (500 ms), a call sup7 decides runs in the same request: the agent never retries.

While paused through the admin API, the loop does not poll: approvals stay pending in the mesh for a human.

## Current state (September 2026)

- **v0.1.0** — 4 providers (Ollama, Anthropic, Jev, Claude Code) and a provider chain with circuit breaker, rule engine, HTTP admin API with file editing and evaluation runs, question sets in YAML, `sup7 bench replay`, systemd unit, 176 tests
- **Jev in production** since 2026-09-29, first in the chain, Ollama as fallback
- **Claude Code callback**: MCP tools defined but the MCP server is not started yet
- **Install**: from the GitHub repository, not on PyPI
- **SDKs** — consumes `mesh7` (AgentMesh) and `mem7` (Mem7) Python SDKs
- **Extracted** from flux7-console backend, now standalone

Apache 2.0 licensed. [github.com/KTCrisis/flux7-supervisor](https://github.com/KTCrisis/flux7-supervisor)

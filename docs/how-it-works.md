# How it works

An agent calls a tool. Before the call reaches the tool, it meets up to four
layers, each asking one question. Most calls stop at the first: a rule
decides. Only what no rule, no history and no evaluator can settle reaches a
human.

```{.text .f7-diagram}
                               ◆ agent calls a tool
                               │
  ┌────────────────────────────▼────────────────────────────┐
  │ L0   flux7-mesh     does a policy rule decide?          │
  │      first match wins · a grant lifts human_approval    │
  └─┬──────────────────────────┬──────────────────────────┬─┘
  allow                 human_approval                  deny
    │                                                     │
    │    ┌─────────────────────▼─────────────────────┐    │
    ◄────┤  L1   flux7-memory     approved before?   │ · · ·
    │ ok │       reads · 3 human yes, 0 no           │    │
    │    └─────────────────────┬─────────────────────┘    │
    │                      otherwise                      │
    │    ┌─────────────────────▼─────────────────────┐    │
    ◄────┤  L1+  flux7-supervisor rules, then judge  ├────►
    │ ok │       low confidence → escalate           │ no │
    │    └─────────────────────┬─────────────────────┘    │
    │                      escalate                       │
    │    ┌─────────────────────▼─────────────────────┐    │
    ◄────┤  L2   human   console · mesh CLI          ├────►
    │ ok │       may open a grant · timeout = no     │ no │
    │    └───────────────────────────────────────────┘    │
    ▼                                                     ▼
 reaches the tool                                    refused
═══════════════════════════════════════════════════════════════
  every step is traced · hash-chained file · OTel · decisions → mem7
```

## The four layers

| Layer | Component | Question it answers | Can conclude |
|-------|-----------|---------------------|--------------|
| L0 | [flux7-mesh](mesh7/index.md) | Does a policy rule decide? | allow, deny, or ask for approval |
| L1 | [flux7-memory](mem7/index.md) | Has this agent been approved for this tool before, and never refused? | approve, or pass |
| L1+ | [flux7-supervisor](sup7/index.md) | Do its rules, then a decision model, settle it with enough confidence? | approve, deny, or escalate |
| L2 | a human, in [flux7-console](console/index.md) or the `mesh` CLI | Everything else | approve (optionally with a grant), deny |

Only flux7-mesh is required. Each other layer is optional: without flux7-memory
nothing is approved from history, without flux7-supervisor pending approvals wait
for a human, and an approval nobody answers expires and the call is refused.

## Follow one call

1. The agent calls `filesystem.write_file` through flux7-mesh, over MCP or HTTP.
   The mesh knows who is calling: a JWT, or the agent name in legacy mode.
2. **L0.** The mesh evaluates the agent's policies, first match wins. `allow`
   forwards the call, `deny` refuses it. `human_approval` puts it in the approval
   queue, unless an active grant covers this agent and tool, in which case it is
   forwarded.
3. **L1.** If the mesh is connected to flux7-memory, it asks for past decisions on
   this agent and tool. For a tool that reads, three approvals by a human and no
   refusal: approved. Writes, a refusal in the history, or suspicious arguments:
   the call stays pending. Exact conditions and settings:
   [Precedents](mesh7/mem7-auto-approve.md).
4. **L1+.** flux7-supervisor polls the queue. Its rules come first; what no rule
   settles goes to a decision model that answers narrow questions with
   probabilities (Jev in production; the same questions on a local model, nimble through Ollama, if it is down), a few hundred
   milliseconds. Below the confidence threshold it escalates: the approval stays
   pending for a human.
5. **L2.** A human approves or denies in the console, the `mesh` CLI or a
   terminal prompt. Approving can open a grant, so the same call does not ask
   again for a while; the grant records this approval as its origin.
6. Whatever the outcome, the call is traced: one line in the hash-chained trace
   file, one OTel span. When the call went through an approval, its decision is
   also written to flux7-memory, where it becomes history for step 3 next time.

## Vocabulary

**Policy**
: A named set of rules for one agent or a glob of agents. Rules match tool names
  (globs) and optionally arguments; the first match gives the action.

**Approval**
: A call waiting for a decision, created by the `human_approval` action. It is
  resolved by flux7-memory, flux7-supervisor or a human, or it expires.

**Grant**
: A temporary permission for an agent on a set of tools. It only lifts
  `human_approval`, never a `deny`. It can record the approval it came from.

**Decision**
: The outcome of an approval, stored as a fact in flux7-memory. Past decisions
  are what L1 reads.

**Trace**
: One line per call: who, which tool, which rule, what outcome, who approved.
  Updates (an approval outcome, the backend status) are appended, never rewritten.

**Chain of authority**
: For a call let through by a grant, the path back to the approval that created
  the grant: `GET /traces/{id}/why`, shown in the console's trace detail.

**Trace chain**
: The hash chain over the trace file. An edited, deleted or inserted line breaks
  it; with a key, rewriting it requires the key. See
  [Trace integrity](mesh7/trace-integrity.md).

## Where to go next

- Run it: [Getting started](getting-started.md)
- Write rules: [Writing policies](mesh7/writing-policies.md)
- The approval queue in detail: [Approval flow](mesh7/approval-flow.md)
- Prove what happened: [Trace integrity](mesh7/trace-integrity.md), [Observability](mesh7/otel.md)

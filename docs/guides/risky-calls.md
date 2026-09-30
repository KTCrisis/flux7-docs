# Hold risky calls for a human

Most calls are routine; a few deserve a look. This guide sends the few to a
human, lets the rest through, and keeps the human from answering the same
question twice.

## 1. Decide on the arguments, not only on the tool

"May this agent use `send_email`?" is too coarse: the answer is yes for a
colleague, no for an unknown address. Conditions read the call's arguments:

```yaml title="policies/assistant.yaml"
name: assistant
agent: "assistant"
rules:
  # Internal mail goes out, anything else waits
  - tools: ["gmail.send_email"]
    action: allow
    condition: { field: "to", operator: "contains", value: "@example.com" }
  - tools: ["gmail.send_email"]
    action: human_approval

  # Small refunds pass, large ones wait
  - tools: ["billing.refund"]
    action: allow
    condition: { field: "amount", operator: "<", value: 100 }
  - tools: ["billing.refund"]
    action: human_approval

  # Deleting is never the assistant's call
  - tools: ["*.delete_*"]
    action: deny
```

First match wins, so each guarded rule sits above its fallback. The field path
starts at the arguments: `amount`, not `params.amount`. Check a new rule
before trusting it:

```bash
curl -s -X POST localhost:9090/decide -H "Content-Type: application/json" \
  -d '{"agent":"assistant","tool":"billing.refund","arguments":{"amount":250}}'
# → {"action": "human_approval", ...}
```

## 2. Answer the held calls

A held call waits in the queue. Answer it from wherever someone is on duty:

```bash
mesh watch                 # prompts for each pending call as it arrives
mesh approve <id>          # or one at a time
mesh deny <id>
```

or from the approval page of [flux7-console](../console/index.md). A call
nobody answers expires and is refused.

## 3. Don't ask twice

Three ways to spare the human a repeated question, from the most explicit to
the most automatic:

| | What it does | When |
|---|---|---|
| **Grant** | `mesh approve <id> --grant 1h` lets the same call through for an hour, and records your approval as its origin | a batch of similar calls, now |
| **Precedents** | with [flux7-memory](../mem7/index.md) connected, a read a human approved several times and never refused is approved on its own; writes always ask again | routine reads, over weeks |
| **Supervisor** | [flux7-supervisor](../sup7/index.md) judges the pending call and escalates only when unsure | novel calls, at volume |

Each leaves a trace that names what allowed the call: the grant and the
approval behind it, the precedents, or the supervisor's reasoning.

**Reference:** [Writing policies](../mesh7/writing-policies.md) ·
[Approval flow](../mesh7/approval-flow.md) ·
[Precedents](../mesh7/mem7-auto-approve.md)

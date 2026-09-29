# Measuring

A calibrated model tells you that a 0.7 is right about seven times in ten. It does not tell you where to cut: letting a 0.7 through alone, or waiting for 0.9, is a choice of risk, and the only way to make it is to measure on cases whose answer you know. sup7 carries the measuring bench with it.

## The method: observe, measure, delegate

1. **Observe.** On day one, delegate nothing: every call the policy sends to approval goes to a human, as before. The decisions humans take become the labels.
2. **Measure.** Replay labelled cases through the evaluator and look at two numbers first: **dangers approved** (must stay at 0) and **normal calls approved** (the load taken off humans).
3. **Delegate by steps.** Automatic approval first, for what is clearly safe. Automatic denial last, or never: a wrong denial costs little, but it saves the human nothing that an escalation would not.

Thresholds are set on this evidence, question by question, and measured again whenever a question, a threshold or the model changes.

## Case sets

A case set is a JSONL file in the mesh7 trace format, in `bench.dir/sets/` (default `~/.sup7/bench/sets/`). The label rides on `policy`: `allow` = approve, `human_approval` = escalate, `deny` = deny. Two kinds are useful together:

- **real calls**, frozen from mesh7 traces: what the agents actually do. Most are normal work, so they measure false alerts;
- **boundary cases**, written by hand: the dangers the traces do not contain yet (deleting outside the project, sending a secrets file out, a hidden instruction in a document), each next to a benign twin.

`sup7 bench replay` builds a set from mesh7 traces. Nothing leaves before it is filtered:

```bash
# allowlist: only calls naming these repositories; keywords as a second barrier;
# --dry-run counts what would be sent and sends nothing
sup7 bench replay --traces traces.jsonl \
  --allow-repo my-service --allow-repo my-lib \
  --exclude-file exclusions.txt --dry-run

# freeze the selection as a case set, recent activity included
sup7 bench replay --traces traces.jsonl --allow-repo my-service --allow-repo my-lib \
  --exclude-file exclusions.txt --export-set ~/.sup7/bench/sets/real.jsonl
```

The allowlist keeps a call only if it names an allowed repository and no other home or `/tmp` path; query tools (web search, fetch) may go without a path; free-text tools stay home. Secret-looking strings are masked, long strings truncated, and a call excluded never appears in the recent activity sent with the next ones. A denylist of keywords alone leaked on the first dry run: prefer the allowlist.

## Runs

A run measures the live configuration on one set, and is kept in `bench.dir/runs/<id>/` with its summary and results. Two modes:

| Mode | Cost | When |
|---|---|---|
| **recompute** | free, about a second | only thresholds changed: re-decides from the raw answers of an earlier run |
| **replay** | a few cents for a thousand cases | the questions or the project dirs changed: Jev's answers depend on them, so it is asked again |

A recompute is refused, with the reason, once the question fingerprint or `project_dirs` differ from the run it would reuse. A replay shows its cost and duration first (`GET /bench/estimate`), runs four calls at a time (the Cloudflare credits gateway answers HTTP 429 above that) and retries a failed case.

From the console (Supervisor, Evaluate tab) or the admin API:

```bash
curl -s -X POST localhost:9096/bench/runs -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" -d '{"set":"boundary","mode":"recompute"}'
```

The summary puts **dangers approved** first, then normal calls approved, correct denies, errors, cases to review (a hesitation between 0.3 and 0.7 on a danger or manipulation signal, or a verdict opposite to the label), latency, and the difference with the previous run of the same set.

## What the first measurements showed

On 2026-09-29, on about a thousand real development calls (shell, file edits, web searches) and 28 boundary cases:

| Step | Normal calls approved | Dangers approved |
|---|---|---|
| broad `destructive` question, thresholds 0.2 / 0.7 | 25 % | 0 |
| narrow questions, `target_zone`, `exfiltrates` = local data leaving | 69 % | 0 |
| thresholds chosen on the measurements: danger ≤ 0.4, `in_scope` floor 0.3, Jev confidence 0.6 | 89 % | 0 |

The dangers all came out at 0.70 or above on at least one signal, normal work mostly below 0.5: the threshold of 0.4 keeps a margin of 0.3 under the lowest danger measured. The two injections scored low on danger (0.06, 0.07) but 0.95 and 0.98 on `injection`.

Every improvement came from rewording questions, not from the model: a web search was read as "data sent to a URL" until `exfiltrates` said *local* data; writing in the repository was an "overwrite" until `target_zone` placed it in the project.

**Limits to keep in mind.** The boundary cases were written by the people who wrote the questions: a danger phrased another way may come out lower. The margin is a reasoned bet, not a proof; cases from independent sources (public agent-safety benchmarks, the operator's own incidents) are what turn it into evidence. And normal-work numbers measured on one team's usage do not transfer to another: each deployment measures its own.

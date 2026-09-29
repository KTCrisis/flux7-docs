# Jev and question sets

[Jev](https://docs.typesafe.ai) (TypeSafe AI) is a decision model, not a text model: it reads a state and answers typed questions (yes/no, one option among several, a level on a scale), each with a probability, in one pass of a few hundred milliseconds. sup7 uses it as its first evaluator for the calls no rule can settle.

## When a model like Jev is the right tool

**If you can write the rule, write the rule.** Jev is for a world where the rule cannot be written: every call is different (any tool, any argument), and what is needed is a quick common-sense judgement in a closed form.

| Situation | Tool |
|---|---|
| closed world, known rules (a path prefix, a tool list) | a sup7 rule, or a mesh7 policy |
| open world, rules that cannot be enumerated (any shell command, any tool) | Jev |
| stable task, thousands of labelled examples, numeric data | a trained model |
| text to produce, several reasoning steps | an LLM |

Jev learns nothing at runtime, but it is pre-trained: it already knows what a destructive command or a secret looks like. You write the criterion in a sentence, not the rule case by case.

## Jev describes, the code decides

The first integration asked Jev one broad question, "approve, escalate or deny?". On real calls it stayed soft (0.61 to 0.79) where narrow questions on the same calls answered 0.99. Deciding means combining facts with a risk tolerance that belongs to the operator, not to the model. So sup7 asks only for facts, and decides in code.

The base set, the **socle**, shipped with sup7 in `data/socle.yaml`:

| Family | Question | Type | True when |
|---|---|---|---|
| danger | `deletes` | yes/no | the call removes or truncates data |
| danger | `overwrites` | yes/no | it replaces a file or record the state shows to exist (not counted inside the project) |
| danger | `exfiltrates` | yes/no | local data (files, code, secrets, personal data) leaves the machine; a web search query does not, a `git push` does |
| danger | `secrets` | yes/no | it targets credentials, keys, tokens or permissions; code that talks about authentication does not |
| context | `target_zone` | choice | where it acts: `project`, `home`, `system`, `remote`, `none`, against the `project_dirs` given in the state |
| context | `in_scope` | yes/no | it fits the agent's recent activity |
| manipulation | `injection` | yes/no | the parameters carry instructions aimed at a model |

Every yes/no question carries `criteria` with its true and false boundary: Jev reads instructions literally, and a boundary case written in the criterion is what keeps "create a new test file" from reading as "overwrite".

The decision, fail-closed:

1. a manipulation answer above its threshold (`injection_max`) → **escalate**;
2. a danger at or above `deny_min` **and** the call out of the agent's activity (`in_scope` below `deny_in_scope_max`) → **deny**;
3. every counted danger at or below its threshold (`destructive_max` by default) **and** `in_scope` at or above `in_scope_min` → **approve**, with confidence `1 - max(danger, manipulation)`, then checked against the provider's `confidence_threshold`;
4. anything else → **escalate**.

`overwrites` does not count when `target_zone = project` is probable enough (`project_min`): overwriting a file in the project is the agent's job, not a harm. `in_scope` gates approval and denial but stays out of the confidence: on the bench it separated nothing and mostly produced false alerts.

## What a decision records

The reasoning keeps every probability and says which model and which wording decided:

```
[jev] Jev jev-1.13.0 q=9bf9f0bc317f: escalate (destructive 0.50 (deletes 0.14 · overwrites 0.36 · exfiltrates 0.50 · secrets 0.10) · project 0.00 · in_scope 0.85 · injection 0.08)
```

`q=` is a fingerprint of the questions sent to Jev (type, instructions, criteria): it changes as soon as one word of one criterion changes. The decision log, flux7-memory and `GET /decisions` also carry, under `evaluator`, the model version, the fingerprint, the packs asked and the thresholds applied. A blocked call says which fact blocked it, which a human can check.

## Business packs

The socle fits any agent. A business adds its own questions in a pack, a YAML file listed under `evaluator.jev.questions` (a glob such as `~/.sup7/questions/*.yaml` lets a new file be picked up without touching the config):

```yaml
pack: finance
applies_to: ["agent:accounting-*", "tool:bank.*"]   # empty = every call
questions:
  payment:
    type: noul
    group: danger
    threshold: 0.1            # stricter than the socle's default
    instructions: The call initiates or approves a payment, a transfer or a refund.
    criteria:
      true: Creates, validates or releases a payment order, a transfer, a refund
      false: Reads balances, lists invoices, prepares a draft not yet submitted
```

For each pending call, sup7 asks the socle plus every pack that applies to its agent or tool, in the same request. One sup7 serves every domain; several sup7 instances make sense for organisational reasons only (data that must not cross, different owners, jurisdictions, volume), split with `poll.tool_scopes`.

Only `type`, `instructions` and `criteria` are sent to Jev; `group`, `role` (`scope` on the question that gates approval and denial), `threshold` and `ignore_when` stay in sup7. Every set is validated when sup7 starts, and when it is edited through the admin API: types, families, choice options, thresholds, `applies_to` patterns, a question defined twice, an `ignore_when` pointing at a missing option. A set that loads is a set sup7 can decide with.

## Known limits of the model

From TypeSafe's own notes on jev-1.13, those that matter here:

- **literal reading**: it answers the question written, not the one meant; write the boundary in the criteria;
- **hostile content** is not treated as hostile by default; `injection` caught crude injections at 0.95 to 0.98, a subtle one is not measured;
- **large irrelevant state** lowers accuracy: sup7 sends the call, five recent calls and the active grants, nothing more;
- **no invariant** between question types: a yes/no probability and a choice probability are not on the same scale, thresholds do not transfer between them, nor between model versions.

Open alternatives with the same `/v1/systemone` contract (Laya, Kev) can run locally, which matters when call parameters must not leave the infrastructure: set `backend: typesafe` and point `url` at the local runtime. Their calibration is not provided; measure it.

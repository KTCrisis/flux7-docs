# Time: what held, and what mem7 believed

Since 03/10/2026 (mem7 0.8), every memory is a series of **versions** with two
time axes:

| Axis | Columns | Question it answers |
|---|---|---|
| **Valid time** | `valid_from`, `valid_to` | When did this hold in the world? |
| **Transaction time** | `tx_from`, `tx_to` | When did mem7 believe it? |

A plain memory store/read works as before: a fact holds from the moment it is
written, and reads return what mem7 believes now of what holds now.

## Writing

`memory_store` takes two optional arguments, a date (`2026-03-20`, midnight UTC)
or an RFC3339 time:

- `valid_from`: when the fact starts to hold. In the past, it corrects history.
- `valid_to`: when it stops holding.

A new version ends the versions it overlaps, and keeps what they said outside
its own period:

```
2026-02-15  store event7:host "Railway"                      Railway    [02-15, …)
2026-03-20  store event7:host "Cloudflare"                   Railway    [02-15, 03-20)  Cloudflare [03-20, …)
2026-04-12  store event7:host "local only" valid_from=04-05  … Cloudflare [03-20, 04-05)  local only [04-05, …)
```

The last line is a correction of the past: what mem7 believed on 04-11
(Cloudflare until today) is no longer believed, but is still readable `as_of`
that day.

`memory_forget` stops believing a key; it does not erase the past: a read
`as_of` an earlier moment still sees it.

## Reading

`memory_recall`, `memory_search`, `memory_context` and `memory_list` take:

- `valid_at`: what held at that moment ("where was event7 on March 1st?")
- `as_of`: what mem7 believed at that moment ("what did we know on April 11th?")

Both together: what mem7 believed on one day about another. A version with an
explicit validity shows a `Valid: from → to` line (recall, search) or
`valid_from` / `valid_to` / `tx_from` / `tx_to` fields (context);
`memory_history` lists each write with the validity it declared.

## Storage

The markdown workspace stays the source of truth: a store entry carries
`valid_from` / `valid_to` when the caller gave them, under the hash chain. The
index keeps every version; an index built before versions (one row per key) is
rebuilt from the markdown when mem7 starts, keeping embeddings.

Out of scope for now: detecting that two different keys contradict each other.

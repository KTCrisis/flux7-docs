# Agent Security

Giving an AI agent tools is giving it hands. The moment an agent can write a
file, send an email, call an API, or run a command, the question stops being
"is the model good?" and becomes "what can this thing actually *do*, and who
said it could?". flux7-mesh exists to answer that question at the tool-call
layer — and, just as importantly, to keep the answer honest under attack.

This page maps the agent-security threat model to the mechanisms mesh7 provides,
and records how those mechanisms were hardened by auditing the mesh against
itself.

## The threat model

An agent runtime is an attacker-influenced system. The model's input includes
web pages, file contents, tool results, and other agents' output — any of which
can carry instructions. So the threats are not hypothetical:

| Threat | What it looks like | OWASP |
|---|---|---|
| **Excessive agency** | An agent calls a tool it should never have been allowed to | LLM06 |
| **Privilege escalation** | A constrained agent grants itself, or assumes, more rights | ASI03 |
| **Indirect prompt injection** | A poisoned web page / file makes the agent run a dangerous tool call | LLM01 |
| **Human-in-the-loop bypass** | The "a human must approve" gate is circumvented | ASI09 |
| **Identity spoofing** | An agent claims to be a different, more privileged agent | ASI03 |
| **Supply-chain / tool poisoning** | A compromised upstream tool server redirects or exfiltrates traffic | ASI04 |
| **Cascading failure / DoS** | One agent exhausts shared resources and takes down governance for all | ASI08 |

mesh7 does not try to make the model trustworthy. It assumes the model can be
manipulated and constrains what its tool calls can achieve.

## The mechanisms

Every tool call — over MCP stdio, MCP Streamable HTTP, or the REST API —
travels the same pipeline: **identity → rate limit → policy → (grant) →
(auto-approve) → (human approval) → forward → trace**. Each stage answers one
question.

| Mechanism | Defends against | Page |
|---|---|---|
| **Per-agent policy**, first-match, fail-closed | excessive agency, unauthorized tools | [Writing Policies](writing-policies.md) |
| **`human_approval` action** (blocking) | high-stakes actions without oversight | [Approval Flow](approval-flow.md) |
| **JWT identity** (strict) | identity spoofing | [JWT Authentication](jwt-auth.md) |
| **Control-plane auth** (admin token / loopback) | tampering with traces, grants, approvals | [Control Plane Auth](control-plane-auth.md) |
| **Injection detection** before auto-approve | indirect prompt injection | [Memory Integration](mem7-auto-approve.md) |
| **Rate limiting + loop detection** | runaway agents, resource exhaustion | [Writing Policies](writing-policies.md) |
| **SSRF-guarded egress** | tool/spec poisoning, metadata theft | this page |
| **Full JSONL trace** | "who approved that?", audit, forensics | [Observability](otel.md) |

Two principles run through all of them:

- **Fail closed.** No matching policy means deny. Missing credentials mean
  anonymous, and anonymous has no rights unless you grant them. The default is
  always the safe one.
- **The mesh governs the mesh.** Actions that *modify* governance — minting a
  grant, resolving an approval, reading the trace history — are themselves
  privileged operations, not free side-doors.

## Hardening: auditing the mesh against itself

A governance tool that can be bypassed is theatre. To check that mesh7 is not
theatre, we pointed [audit7](https://github.com/KTCrisis/audit7) — a
RAG-powered audit agent that runs *through the mesh* and stores findings in
[flux7-memory](https://github.com/KTCrisis/flux7-memory) — at flux7-mesh's own
source, with the OWASP LLM and Agentic corpora as its knowledge base.

It found real issues. The interesting ones were structural:

**Self-escalation via grants.** The virtual `grant.create` tool was dispatched
*before* policy evaluation, so any agent could mint itself a `*` grant and
bypass its own `human_approval` rules. A grant is sudo; minting one had no
guard. Fixed: grant mutation is now reserved to declared supervisor agents.

**Its twin: self-approval.** A second audit pass found the sibling we had
missed — `approval.resolve` was callable by any agent in the default mode, so an
agent could approve *its own* pending request. Same class of bug, the other door
to the same room. The lesson: when you guard one privileged primitive, look for
its twin. Both are now gated identically.

**Identity spoofing past JWT.** The legacy `Bearer agent:<name>` form was checked
*before* JWT validation, so it worked even with JWT configured — making the
cryptographic check decorative. Fixed: with JWT on, plaintext identity claims
are rejected (strict by default).

**Injection slipping through one transport.** The injection check that gates
auto-approval lived only on the HTTP path; the MCP path — the busiest one, used
by Claude Code and Cursor — had none. A poisoned tool call could be silently
auto-approved. Fixed: the guard now lives in one shared function both paths call.

**SSRF via callbacks and spec URLs.** Agent-supplied callback URLs and OpenAPI
spec URLs were fetched without protection, vulnerable to DNS rebinding and cloud
metadata theft (`169.254.169.254`). Fixed: outbound fetches go through an
SSRF-guarded client that checks every resolved IP at dial time.

Every fix followed the same shape, which is the single most useful pattern for
agent security work:

> **A safe, explicit default plus an opt-in escape hatch.**
> Control plane → loopback-only by default, `admin_token` to open it.
> Identity → JWT-strict by default, `allow_legacy` to relax it.
> Transport → plaintext with a warning by default, `tls.cert_file` to encrypt.
> Data plane → anonymous-allowed (policy-governed) by default,
> `require_authentication` to lock it.

The default never silently exposes anything; the capability is there when the
deployment genuinely needs it.

## Production hardening checklist

For any deployment where the mesh is reachable beyond loopback:

- [ ] **Identity**: configure `auth.jwt` and leave `allow_legacy` off, so agent
      identity is cryptographically proven.
- [ ] **Control plane**: set `auth.admin_token` (or `MESH_ADMIN_TOKEN`) — never
      rely on loopback when the port is exposed.
- [ ] **Transport**: terminate TLS at your ingress, or set `tls.cert_file` /
      `tls.key_file` for standalone hosts. Don't expose the port in plaintext.
- [ ] **Data plane**: set `auth.require_authentication: true` to reject
      anonymous callers and stop registry enumeration.
- [ ] **Supervisors**: declare `supervisor.supervisor_agents` explicitly — only
      those agents can mint grants or resolve approvals over MCP.
- [ ] **Policies**: prefer `strict: true` for CLI tools, write glob policies
      (`terraform.*`) so undeclared commands are covered, and keep a final
      `"*" → deny` so nothing slips through.
- [ ] **Limits**: set per-agent `rate_limit` so a runaway or hostile agent can't
      exhaust shared resources.

## See also

- [Writing Policies](writing-policies.md) · [Approval Flow](approval-flow.md) ·
  [JWT Authentication](jwt-auth.md) · [Control Plane Auth](control-plane-auth.md)
- OWASP [LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
  and [Agentic Security Initiative](https://genai.owasp.org/)

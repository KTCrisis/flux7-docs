# What flux7 does

flux7 sits between AI agents and the tools they call. Every call is checked
against a policy before it reaches the tool, recorded in a verifiable trail,
and, when the policy asks for it, held until someone or something with the
authority to decide has decided.

No code change in the agent: it sees the same tools, through a proxy.

## The question it answers

An agent with access to files, mail, a database or a cloud API can do
something irreversible. Today there are two answers, and neither holds:

- **Allow everything**, and read the logs afterwards.
- **Ask a human every time**, until the human approves without reading.

flux7 fills the space between them. Rules settle what is clear-cut. What a
human has already approved, repeatedly, stops being asked. What remains goes
to an automated evaluator, then to a human if the evaluator is not confident.

## What is governed

| Governed | How |
|---|---|
| **MCP tools**, local (stdio) or hosted (HTTP) | the agent connects to flux7-mesh instead of the server |
| **REST APIs** described by an OpenAPI spec | each operation becomes a governed tool |
| **Command-line binaries** | each subcommand becomes a governed tool, flags allowlisted |
| **Tools that never transit the proxy**, such as an agent harness's built-in shell or file tools | a hook asks the mesh "would you allow this?" before each call and enforces locally |

Any MCP client works: Claude Code, Cursor, Claude Managed Agents, the Agent
SDK, LangChain, a plain script over HTTP.

## What you get

- **A decision per call, per agent, on the arguments.** Not only "may this
  agent use `send_email`", but "may it send to this domain", "may it write
  outside this directory", "may it set `credit_limit` above 10 000".
- **Human approval outside the agent.** Pending calls wait in a queue,
  answered from a terminal or a browser, by whoever is on duty.
- **Precedents.** A read a human approved three times, never refused, is
  approved on its own the next time. Writes are always asked again.
- **An automated evaluator** for the calls no rule and no precedent settles.
  It asks a small model narrow factual questions (does this delete, does it
  send local data out, does it touch secrets) and decides in code from the
  answers, with thresholds measured on real traffic.
- **A trail that says why.** Each call records the rule, the grant or the
  human approval that allowed it. The trace file is hash-chained, so an
  edited or deleted line is detected.
- **Standard observability.** OpenTelemetry export and Prometheus metrics,
  into the tools you already run.

## What it does not do

- **It does not govern model output.** It governs tool calls: what an agent
  *does*, not what it *says*. Pair it with a guardrail on the model traffic
  if you need both.
- **It does not replace an identity provider or an API gateway.** It reads
  identity from JWTs issued by yours (Keycloak, Auth0, Cloudflare Access), and
  it runs behind a gateway such as Kong: the gateway checks who calls which
  tool, flux7 judges the call itself.
- **Policy conditions match text.** Denying `rm -rf` does not stop `RM -RF`
  or a base64 payload. The automated evaluator exists for what text matching
  cannot see, and it is measured, not proven.
- **It is young.** One maintainer, versions below 1.0. The
  [features page](features.md) states the maturity of each part.

## How it is deployed

Four components, each optional except the first:

| Component | Role | Form |
|---|---|---|
| [flux7-mesh](../mesh7/index.md) | policy, approvals, traces | one Go binary, one YAML file |
| [flux7-memory](../mem7/index.md) | precedents and agent memory | one Go binary, Markdown files on disk |
| [flux7-supervisor](../sup7/index.md) | automated evaluation | Python service |
| [flux7-console](../console/index.md) | dashboard and approval UI | Next.js web app |

Everything is self-hosted and Apache 2.0. Nothing leaves your network unless
you configure it to: the supervisor's model can be local (Ollama) or remote.

**Next:** the [feature list](features.md), [how a call flows through the
layers](../how-it-works.md), or [install it](../getting-started.md).

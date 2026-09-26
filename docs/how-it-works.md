# How it works

An agent calls a tool. Before the call reaches the tool, it meets up to four
layers, each asking one question. Most calls stop at the first: a rule
decides. Only what no rule, no history and no evaluator can settle reaches a
human.

<div class="f7-deco">
<svg viewBox="0 0 760 880" role="img" aria-labelledby="hiw-title hiw-desc" xmlns="http://www.w3.org/2000/svg">
  <title id="hiw-title">The path of a tool call through flux7</title>
  <desc id="hiw-desc">A call descends through four stepped tiers. L0 flux7-mesh applies the policy. L1 flux7-memory approves from history. L1+ flux7-supervisor applies rules then an LLM. L2 a human decides in the console or the CLI. At each tier the call can leave to the left, allowed to reach the tool, or to the right, refused. Every step is traced.</desc>

  <!-- sunburst over the agent -->
  <g class="f7d-rays">
    <line x1="380" y1="112" x2="380" y2="24"/>
    <line x1="380" y1="112" x2="335" y2="30"/><line x1="380" y1="112" x2="425" y2="30"/>
    <line x1="380" y1="112" x2="296" y2="48"/><line x1="380" y1="112" x2="464" y2="48"/>
    <line x1="380" y1="112" x2="266" y2="76"/><line x1="380" y1="112" x2="494" y2="76"/>
    <line x1="380" y1="112" x2="250" y2="108"/><line x1="380" y1="112" x2="510" y2="108"/>
  </g>
  <path class="f7d-arc" d="M 290 112 A 90 90 0 0 1 470 112"/>
  <path class="f7d-arc f7d-thin" d="M 316 112 A 64 64 0 0 1 444 112"/>

  <!-- agent -->
  <polygon class="f7d-agent" points="380,84 404,112 380,140 356,112"/>
  <text class="f7d-cap" x="380" y="116">AGENT</text>
  <text class="f7d-small" x="380" y="160">calls a tool</text>

  <!-- central axis -->
  <line class="f7d-axis" x1="380" y1="140" x2="380" y2="744"/>

  <!-- pillars -->
  <g class="f7d-pillar">
    <rect x="34" y="196" width="64" height="552"/>
    <rect class="f7d-inset" x="40" y="202" width="52" height="540"/>
    <polygon points="26,196 106,196 98,184 34,184"/>
    <polygon points="26,748 106,748 98,760 34,760"/>
  </g>
  <g class="f7d-pillar f7d-refuse">
    <rect x="662" y="196" width="64" height="552"/>
    <rect class="f7d-inset" x="668" y="202" width="52" height="540"/>
    <polygon points="654,196 734,196 726,184 662,184"/>
    <polygon points="654,748 734,748 726,760 662,760"/>
  </g>
  <text class="f7d-vert" transform="translate(70 472) rotate(-90)">REACHES THE TOOL</text>
  <text class="f7d-vert f7d-vert-refuse" transform="translate(698 472) rotate(90)">REFUSED</text>

  <!-- L0 -->
  <polygon class="f7d-tier" points="162,196 598,196 610,208 610,284 598,296 162,296 150,284 150,208"/>
  <polygon class="f7d-inset" points="166,202 594,202 604,212 604,280 594,290 166,290 156,280 156,212"/>
  <text class="f7d-level" x="380" y="228">L0 · FLUX7-MESH</text>
  <text class="f7d-q" x="380" y="252">Does a policy rule decide?</text>
  <text class="f7d-small" x="380" y="274">first match wins · an active grant lifts human_approval</text>
  <line class="f7d-out" x1="150" y1="246" x2="98" y2="246"/><polygon class="f7d-tip" points="98,241 90,246 98,251"/>
  <line class="f7d-out f7d-out-r" x1="610" y1="246" x2="662" y2="246"/><polygon class="f7d-tip f7d-tip-r" points="662,241 670,246 662,251"/>
  <text class="f7d-edge" x="124" y="238">allow</text>
  <text class="f7d-edge" x="636" y="238">deny</text>
  <text class="f7d-down" x="392" y="318">human_approval</text>

  <!-- L1 -->
  <polygon class="f7d-tier" points="212,336 548,336 560,348 560,424 548,436 212,436 200,424 200,348"/>
  <polygon class="f7d-inset" points="216,342 544,342 554,352 554,420 544,430 216,430 206,420 206,352"/>
  <text class="f7d-level" x="380" y="368">L1 · FLUX7-MEMORY</text>
  <text class="f7d-q" x="380" y="392">Was this approved before?</text>
  <text class="f7d-small" x="380" y="414">same agent and tool · 3 approvals, no refusal</text>
  <line class="f7d-out" x1="200" y1="386" x2="98" y2="386"/><polygon class="f7d-tip" points="98,381 90,386 98,391"/>
  <line class="f7d-out f7d-none" x1="560" y1="386" x2="662" y2="386"/>
  <text class="f7d-edge" x="149" y="378">approve</text>
  <text class="f7d-edge f7d-dim" x="611" y="378">never refuses</text>
  <text class="f7d-down" x="392" y="458">otherwise</text>

  <!-- L1+ -->
  <polygon class="f7d-tier" points="262,476 498,476 510,488 510,564 498,576 262,576 250,564 250,488"/>
  <polygon class="f7d-inset" points="266,482 494,482 504,492 504,560 494,570 266,570 256,560 256,492"/>
  <text class="f7d-level" x="380" y="508">L1+ · FLUX7-SUPERVISOR</text>
  <text class="f7d-q" x="380" y="532">Do its rules, then an LLM, decide?</text>
  <text class="f7d-small" x="380" y="554">below the confidence threshold: escalate</text>
  <line class="f7d-out" x1="250" y1="526" x2="98" y2="526"/><polygon class="f7d-tip" points="98,521 90,526 98,531"/>
  <line class="f7d-out f7d-out-r" x1="510" y1="526" x2="662" y2="526"/><polygon class="f7d-tip f7d-tip-r" points="662,521 670,526 662,531"/>
  <text class="f7d-edge" x="174" y="518">approve</text>
  <text class="f7d-edge" x="586" y="518">deny</text>
  <text class="f7d-down" x="392" y="598">escalate</text>

  <!-- L2 -->
  <polygon class="f7d-tier f7d-human" points="312,616 448,616 460,628 460,704 448,716 312,716 300,704 300,628"/>
  <polygon class="f7d-inset" points="316,622 444,622 454,632 454,700 444,710 316,710 306,700 306,632"/>
  <text class="f7d-level" x="380" y="648">L2 · HUMAN</text>
  <text class="f7d-q" x="380" y="672">console · mesh CLI</text>
  <text class="f7d-small" x="380" y="694">may open a grant</text>
  <line class="f7d-out" x1="300" y1="666" x2="98" y2="666"/><polygon class="f7d-tip" points="98,661 90,666 98,671"/>
  <line class="f7d-out f7d-out-r" x1="460" y1="666" x2="662" y2="666"/><polygon class="f7d-tip f7d-tip-r" points="662,661 670,666 662,671"/>
  <text class="f7d-edge" x="199" y="658">approve</text>
  <text class="f7d-edge" x="561" y="658">deny · timeout</text>

  <!-- plinth -->
  <g class="f7d-plinth">
    <rect x="26" y="784" width="708" height="60"/>
    <rect class="f7d-inset" x="32" y="790" width="696" height="48"/>
    <polyline points="330,784 350,772 410,772 430,784"/>
  </g>
  <text class="f7d-plinth-t" x="380" y="812">EVERY STEP IS TRACED</text>
  <text class="f7d-small" x="380" y="830">hash-chained trace file · OTel spans · decisions written to flux7-memory</text>
</svg>
</div>

## The four layers

| Layer | Component | Question it answers | Can conclude |
|-------|-----------|---------------------|--------------|
| L0 | [flux7-mesh](mesh7/index.md) | Does a policy rule decide? | allow, deny, or ask for approval |
| L1 | [flux7-memory](mem7/index.md) | Has this agent been approved for this tool before, and never refused? | approve, or pass |
| L1+ | [flux7-supervisor](sup7/index.md) | Do its rules, then an LLM, settle it with enough confidence? | approve, deny, or escalate |
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
   this agent and tool. Three approvals and no refusal: approved. Any refusal, or
   arguments that look like prompt injection: the history is not trusted, the call
   stays pending.
4. **L1+.** flux7-supervisor polls the queue. Its rules come first; what no rule
   settles goes to an LLM. Below the confidence threshold it escalates: the
   approval stays pending for a human.
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

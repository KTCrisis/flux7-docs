# Example: Keycloak and Kong in front of mesh7

A worked example, built and run end to end with a real agent: [scout7](https://github.com/KTCrisis/scout7), a research agent that searches the web, reads pages, analyses them with a local model, remembers what it found and draws architecture diagrams. Every tool call goes through mesh7.

Three layers, three questions:

| Layer | Answers | Sees |
|---|---|---|
| **Keycloak** | *Who* is acting, and *what* may they delegate? | Users, roles, clients, scopes |
| **Kong** (Konnect or Gateway Enterprise 3.14+) | May this agent, for this person, reach this *tool*? | The token and its scopes, the MCP method and tool name |
| **mesh7** | Should this *act*, with these *arguments*, run, wait for a human, or stop? | The tool, its arguments, the agent and the human, the history |

Kong opens the door; mesh7 judges the act. Each layer is useful alone; the example adds them one at a time.

```
human ──sign-in──► Keycloak
                     │ token (human)            ┌── exchange (RFC 8693) ──┐
chat ────────────────┴──────────────────────────┘                        ▼
                                                   token: azp=scout7, preferred_username=bob,
                                                          scope=web:read memory:write …
scout7 ── MCP ──► Kong :8010 ── ai-mcp-oauth2 (token) ── ai-mcp-proxy (scope per tool) ──►
          mesh7 :9196 ── policy (arguments, approval) ── trace ──► tools (search, fetch, LLM, memory, diagrams)
```

## Part 1: Keycloak and mesh7

### The two ways an agent gets a token

**An agent acting for itself** (a scheduled run, nobody behind it): a confidential client with a service account, *client credentials* grant.

**An agent acting for a human**: the human signs in on a front end (a chat, a portal) with the *authorization code* flow and PKCE, so the password never reaches the front end. The front end then **exchanges** the human's token for one meant for the agent ([RFC 8693](https://www.rfc-editor.org/rfc/rfc8693), Keycloak's *standard token exchange*, on by default since Keycloak 26.2). The exchanged token names both:

```json
{ "azp": "scout7", "preferred_username": "bob", "aud": "mesh7",
  "scope": "web:read memory:read memory:write diagram:write" }
```

### The realm, in short

- **Client scopes**, one per capability: `web:read`, `memory:read`, `memory:write`, `diagram:write`, with *include in token scope* on.
- **An audience mapper** adding `mesh7` to the tokens, since mesh7 checks `aud`.
- **Delegation by role**: map the write scopes to a realm role (`scout-writer`). Keycloak puts a scope in a token only when the person holds one of its mapped roles, so bob (who has the role) delegates writing, alice (who has not) delegates reading only. Give the role to the agent's service account too, so it keeps writing on its own.
- **The agent's client** (`scout7`): confidential, service account on, attribute `standard.token.exchange.enabled: true`. Its scopes as **default** client scopes (see pitfalls). A longer access token lifespan than the 5-minute default if a run outlasts it.
- **The front end's client**: standard flow, PKCE `S256`, its redirect and post-logout URIs, and an audience mapper adding the **agent's** client id: the agent may exchange only a token meant for it.

The exchange, from the front end's server side:

```bash
curl -s $KC/realms/agents/protocol/openid-connect/token \
  -d grant_type=urn:ietf:params:oauth:grant-type:token-exchange \
  -d client_id=scout7 -d client_secret=$SCOUT7_SECRET \
  -d subject_token=$HUMAN_ACCESS_TOKEN \
  -d subject_token_type=urn:ietf:params:oauth:token-type:access_token
```

### mesh7

```yaml
auth:
  jwt:
    jwks_url: http://localhost:8180/realms/agents/protocol/openid-connect/certs
    issuer: http://localhost:8180/realms/agents
    audience: mesh7
    agent_claim: azp                  # the agent is the client
    user_claim: preferred_username    # the human it acts for, on every trace
```

The trace then reads *scout7 for bob*, and the OTel span carries `enduser.id`. See [JWT Authentication](jwt-auth.md).

The policy judges what a scope cannot express: the arguments, and what needs a person.

```yaml
name: scout7
agent: "scout7"
rules:
  # Reading the web: https pages only, never an internal address.
  - tools: ["fetch.fetch"]
    action: allow
    condition: { field: url, operator: starts_with, value: ["https://"] }
  - tools: ["fetch.fetch"]
    action: deny

  - tools: ["searxng.searxng_web_search", "ollama.chat", "memory.memory_search", "memory.memory_list"]
    action: allow

  # A diagram is a deliverable: a human approves each one, and only in the
  # demo directory; anywhere else is refused.
  - tools: ["memory.memory_store"]
    action: allow
  - tools: ["arch7.create_diagram"]
    action: human_approval
    condition: { field: output_path, operator: starts_with, value: ["/data/diagrams/"] }
  - tools: ["arch7.create_diagram"]
    action: deny

  - tools: ["*"]
    action: deny
```

An agent that waits for the human retries the same call; mesh7 answers with the same pending approval, then runs the call once approved, or tells the agent once that it was refused. See [Approval Flow](approval-flow.md).

## Part 2: adding Kong

Kong's MCP plugins are Enterprise: Kong Gateway Enterprise 3.14+, or Kong Konnect, whose control plane is SaaS while the data plane node runs next to mesh7 (it dials out over mTLS, no inbound port).

```yaml
# decK: deck gateway sync kong.yaml --konnect-token … --konnect-control-plane-name …
_format_version: "3.0"
services:
  - name: mesh7-scout
    url: http://127.0.0.1:9196/mcp        # the mesh's MCP endpoint
    routes:
      - name: scout-mcp
        expression: 'http.path == "/scout/mcp"'   # Konnect nodes run the expressions router
        protocols: [http]
        strip_path: true                  # /scout/mcp -> the service's /mcp
    plugins:
      - name: ai-mcp-oauth2
        config:
          resource: http://localhost:8010/scout/mcp
          authorization_servers: [http://localhost:8180/realms/agents]
          jwks_endpoint: http://localhost:8180/realms/agents/protocol/openid-connect/certs
          passthrough_credentials: true   # mesh7 needs the token to name agent and human
          insecure_relaxed_audience_validation: true   # see pitfalls; mesh7 still checks aud
      - name: ai-mcp-proxy
        config:
          mode: passthrough-listener
          acl_attribute_type: oauth_access_token
          access_token_claim_field: '.scope | split(" ")'
          default_acl:
            - scope: tools
              allow: ["kong:unlisted-tool"]   # a tool not listed below is refused
          tools:
            - { name: searxng.searxng_web_search, description: Search the web.,        acl: { allow: ["web:read"] } }
            - { name: fetch.fetch,                description: Fetch a page.,          acl: { allow: ["web:read"] } }
            - { name: ollama.chat,                description: Analyse with a model.,  acl: { allow: ["web:read"] } }
            - { name: memory.memory_search,       description: Search memory.,         acl: { allow: ["memory:read"] } }
            - { name: memory.memory_list,         description: List memories.,         acl: { allow: ["memory:read"] } }
            - { name: memory.memory_store,        description: Store a finding.,       acl: { allow: ["memory:write"] } }
            - { name: arch7.create_diagram,       description: Draw a diagram.,        acl: { allow: ["diagram:write"] } }
```

`tools/list` is filtered per scope: the agent acting for alice is not even offered the write tools. A `tools/call` to a tool outside the token's scopes is refused with a 403 before mesh7 sees it.

## Part 3: one trace, from Kong to the tool

```yaml
# Kong, on the service
- name: opentelemetry
  config:
    traces_endpoint: http://127.0.0.1:4318/v1/traces
    propagation: { default_format: w3c }
```

Tracing is off by default on the node: set `KONG_TRACING_INSTRUMENTATIONS=all` (and a sampling rate). On mesh7, `otel_endpoint: http://localhost:4318`. Kong propagates a `traceparent`; mesh7 joins it, so one call is one trace: Kong's router and MCP plugin spans, then the mesh7 span for the tool, under Kong's balancer span. See [Observability](otel.md).

## Who catches what

Played end to end with the stack above:

| Situation | Keycloak | Kong | mesh7 |
|---|---|---|---|
| No token | | **401** | |
| alice asks scout7 to store a memory or draw | issues a token without the write scopes | **403** | never sees it |
| A tool no one declared | | **403** (default ACL) | refuses too (`*: deny`) |
| MCP header names one tool, body calls another | | **403** (the plugin reads the body) | would judge the body |
| `fetch` to an `http://` address (cloud metadata, an internal service) | scope `web:read` present | lets it through (200) | **refused** on the argument |
| bob's scout7 draws a diagram | scope present | lets it through | **held for a human**, approved or refused in the console |
| The upstream server changes a tool's description | | | **held back** ([`pin_tools`](configuration.md)) |

Kong decides on the identity and the tool name; mesh7 decides on the act.

## Pitfalls met on the way

- **Requesting a scope the client no longer has** makes Keycloak refuse the whole token (`invalid_scope`). For an agent whose scopes you may revoke, make them *default* client scopes and request none: revoking one removes it from the next token, and nothing else.
- **`resource` must be a URL** in `ai-mcp-oauth2`. If your tokens carry another audience (here `mesh7`), either add the resource URL as an audience in Keycloak or relax Kong's audience check and let mesh7 enforce `aud`.
- **The `scope` claim is one space-separated string**, and the ACL compares whole values: without `split(" ")`, every listed tool is hidden from everyone.
- **`"*"` is not a wildcard** in `default_acl`: tools not listed would pass. Ask for a scope no token carries instead.
- **Service URL and route path add up**: a service at `…/mcp` behind a route `/mcp` without `strip_path` calls `/mcp/mcp`.
- **The Konnect node image runs as uid 1001**: pass the cluster certificate and key as environment values (as Konnect's own `docker run` does) rather than as files private to your user.
- **Password grant is for tests only**: the front end should use the authorization code flow with PKCE, and sign the user out of Keycloak too, or its session signs the same person straight back in.

## Limits, today

- mesh7 **records** the human (`user_claim`) but its policies do not yet **decide** on the human or on the organisation: alice and bob are told apart by Keycloak (roles) and Kong (scopes). Claim-based conditions are on the roadmap.
- The `fetch` rule compares the start of the URL: it stops `http://` addresses, not an internal service reached over `https://`. It is a text condition, not an SSRF guard; resolving the host and refusing private addresses would belong in the tool or in a dedicated check.
- The agent must speak MCP Streamable HTTP to go through Kong's MCP plugins; mesh7's REST data plane (`POST /tool/…`) needs a plain route and loses the per-tool ACL.

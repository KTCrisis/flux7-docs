# JWT Authentication

Since v0.13.0, flux7-mesh validates JWT tokens from external identity providers. mesh7 enforces — the IdP issues tokens.

## How it works

```
External IdP (Auth0, Cloudflare Access, Keycloak)
  │
  └── issues JWT with agent identity in claims
         │
         ▼
Agent ── Authorization: Bearer <jwt> ──► mesh7
                                          │
                                     1. Fetch JWKS (cached, refresh 5 min)
                                     2. Validate signature (RS256 / ES256)
                                     3. Check exp, iss, aud
                                     4. Extract agent ID from claim
                                          │
                                     ▼ policy engine (same YAML rules)
```

mesh7 does not issue tokens. It validates them against the IdP's public keys (JWKS endpoint) and extracts the agent identity from a configurable claim.

## Configuration

```yaml
auth:
  jwt:
    jwks_url: https://auth.example.com/.well-known/jwks.json
    issuer: https://auth.example.com
    audience: mesh7
    agent_claim: sub
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `jwks_url` | Yes | — | JWKS endpoint URL (public keys) |
| `issuer` | No | — | Validate `iss` claim if set |
| `audience` | No | — | Validate `aud` claim if set |
| `agent_claim` | No | `sub` | Which claim is used as agent ID for policies |

**No `auth.jwt` block = no validation.** The legacy `Bearer agent:<name>` format still works, backward compatible.

## Auth flow

```
Authorization header present?
  │
  ├─ No → anonymous
  │
  ├─ "Bearer agent:<name>" → legacy path (no validation, name used as agent ID)
  │
  └─ "Bearer <jwt>" → validate against JWKS
       ├─ Valid → extract agent_claim → agent ID for policies
       └─ Invalid → HTTP 401 Unauthorized
```

Key distinction: **401** (not authenticated — bad token) vs **403** (authenticated but policy denied).

## JWKS caching

mesh7 fetches the JWKS at startup and caches it in memory. A background goroutine refreshes the cache every 5 minutes. No per-request fetch.

If the initial JWKS fetch fails, mesh7 does not start — fail-closed.

## Supported algorithms

- **RSA** — RS256, RS384, RS512
- **ECDSA** — ES256 (P-256), ES384 (P-384), ES512 (P-521)

Tokens must include a `kid` header matching a key in the JWKS response.

## Example: per-agent policies with JWT

```yaml
auth:
  jwt:
    jwks_url: https://auth.example.com/.well-known/jwks.json
    issuer: https://auth.example.com
    agent_claim: sub

policies:
  - name: admin-agents
    agent: "admin-*"
    rules:
      - tools: ["*"]
        action: allow

  - name: worker-agents
    agent: "worker-*"
    rules:
      - tools: ["filesystem.read_*"]
        action: allow
      - tools: ["filesystem.write_*"]
        action: human_approval
      - tools: ["*"]
        action: deny
```

An agent with JWT claim `sub: admin-bot` matches the first policy. An agent with `sub: worker-3` matches the second.

## IdP examples

### Cloudflare Access

```yaml
auth:
  jwt:
    jwks_url: https://<team>.cloudflareaccess.com/cdn-cgi/access/certs
    issuer: https://<team>.cloudflareaccess.com
    audience: <application-aud-tag>
    agent_claim: email
```

### Auth0

```yaml
auth:
  jwt:
    jwks_url: https://<tenant>.auth0.com/.well-known/jwks.json
    issuer: https://<tenant>.auth0.com/
    audience: https://mesh.example.com
```

### Keycloak

```yaml
auth:
  jwt:
    jwks_url: https://<host>/realms/<realm>/protocol/openid-connect/certs
    issuer: https://<host>/realms/<realm>
    audience: mesh7
```

## Local development

For testing without a real IdP, use the `generate-token.py` script in `examples/jwt-auth/` :

```bash
pip install cryptography PyJWT

# Start a local JWKS server
python generate-token.py serve &

# Generate a token
TOKEN=$(python generate-token.py token --sub my-agent)

# Test
curl -H "Authorization: Bearer $TOKEN" http://localhost:9090/health
```

See [`examples/jwt-auth/`](https://github.com/KTCrisis/flux7-mesh/tree/main/examples/jwt-auth) for the full setup.

## MCP Streamable HTTP

JWT validation also applies to MCP Streamable HTTP connections (`POST /mcp`). Managed Agents or remote MCP clients presenting a valid JWT are identified by their claim value — same policies apply.

```
Managed Agent ── POST /mcp ── Authorization: Bearer <jwt> ──► mesh7
                                                                │
                                                           validate JWT
                                                           extract agent ID
                                                           apply policies
```

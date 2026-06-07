# Control Plane Authentication

mesh7 serves two distinct kinds of endpoints. They have different audiences and
different trust requirements, so they are authorized differently.

## Data plane vs control plane

| Plane | Endpoints | Who calls them | Auth |
|-------|-----------|----------------|------|
| **Data** | `POST /tool/{name}`, `POST /decide`, `POST /mcp`, `GET /tools`, `GET /mcp-servers`, `GET /health`, `GET /version` | Agents | Per-agent identity (see [JWT Authentication](jwt-auth.md)) |
| **Control** | `GET /traces`, `GET /otel-traces`, `GET/POST /grants`, `DELETE /grants/{id}`, `GET /approvals`, `POST /approvals/{id}/approve`, `POST /approvals/{id}/deny`, `GET /sessions`, `GET /policies`, `GET /metrics` | Operators, dashboards, supervisors | Admin token (this page) |

The control plane is the governance plane: reading the full trace history,
resolving approvals, and minting temporal grants. A caller with control-plane
access can override the policy decisions the data plane enforces — so it must be
authenticated independently of agent identity.

!!! warning "Why this matters"
    A temporal grant bypasses `human_approval`. If `POST /grants` were
    unauthenticated, any process that can reach the port could mint itself a
    grant and neutralise the approval gate. The control-plane guard closes that
    path; see also [Approval Flow](approval-flow.md) and
    [grant semantics](approval-flow.md#temporal-grants).

## Configuration

```yaml
auth:
  admin_token: "a-long-random-secret"   # or set MESH_ADMIN_TOKEN
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `admin_token` | No | — | Secret required (as `Authorization: Bearer <token>`) on all control-plane endpoints. Compared in constant time. |

`MESH_ADMIN_TOKEN` in the environment overrides the config value.

### Behaviour

- **`admin_token` set** — every control-plane request must send
  `Authorization: Bearer <token>`. Anything else returns `401`, from any
  interface (local or remote).
- **`admin_token` unset** — the control plane accepts **loopback callers only**
  (`127.0.0.1`, `::1`). Remote requests return `401`. This makes the
  localhost-only posture explicit rather than silently exposing the governance
  plane on every interface.

The **data plane is never gated by `admin_token`** — agents keep calling tools
with their own identity regardless of this setting.

```
admin_token unset:
  127.0.0.1  → GET /traces    200
  10.0.0.5   → GET /traces    401
  10.0.0.5   → POST /tool/... 200   (data plane, unaffected)

admin_token: s3cret
  any IP + Bearer s3cret → control plane 200
  any IP, wrong/no token → control plane 401
```

## Operating with a token

```bash
# Read traces from a remote operator host
curl -H "Authorization: Bearer $MESH_ADMIN_TOKEN" \
  https://mesh.internal:9090/traces?agent=audit7

# Approve a pending request
curl -X POST -H "Authorization: Bearer $MESH_ADMIN_TOKEN" \
  https://mesh.internal:9090/approvals/abc123/approve
```

## Recommendations

- **Local development**: leave `admin_token` unset — loopback-only is enough.
- **Shared or containerised deployments**: set `admin_token` (or
  `MESH_ADMIN_TOKEN`) so the control plane is reachable only with the secret.
  Combine with TLS at the ingress so the token is not sent in clear.
- Treat the admin token like any operator credential: rotate it, scope who
  holds it, and keep it out of agent-visible configuration.

## See also

- [JWT Authentication](jwt-auth.md) — data-plane agent identity
- [Approval Flow](approval-flow.md) — what the control plane operates on

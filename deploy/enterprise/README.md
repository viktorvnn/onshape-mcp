# Enterprise single-server deployment

This deployment runs one hardened `onshape-mcp` instance behind Caddy with
automatic TLS. It is intended for an internal engineering team on one
company-managed Linux server. OAuth clients, MCP tokens, and Onshape user
tokens survive restarts in an AES-256-GCM encrypted state file.

## Security model

- Every engineer authenticates with their own Onshape account through a
  company-owned Onshape OAuth application. The configured company ID binds the
  authorization flow to the intended Onshape enterprise.
- The Onshape user ID must be in `ONSHAPE_ALLOWED_USERS`.
- Users default to `read`. `write` additionally permits POST, PUT, and PATCH;
  `full` also permits DELETE. Grant `full` only to engineers whose workflow
  requires destructive API operations.
- MCP access and refresh tokens are audience-bound to this server's exact
  `/mcp` URL. PKCE S256 is mandatory for confidential and public clients.
- Durable OAuth state is encrypted at rest. The encryption key and Onshape
  client secret are mounted read-only and are never put in environment
  variables or command-line arguments.
- The application container is non-root, read-only, capability-free, and not
  published directly. Only Caddy exposes ports 80 and 443.
- HTTP security headers, request-size limits, registration/pending-flow caps,
  fail-closed allowlisting, and JSON audit events for Onshape API calls are
  enabled.

This design deliberately uses one application replica. The encrypted state
file is safe for restart durability, not concurrent writers on shared storage.
For active-active high availability, replace it with a transactional shared
state backend before adding replicas.

## Prerequisites

1. A company-managed Linux server with Docker Engine and Docker Compose.
2. A DNS A/AAAA record for the MCP hostname pointing at the server.
3. Inbound TCP 80/443 and UDP 443; do not expose port 8080.
4. Outbound HTTPS to `oauth.onshape.com` and `cad.onshape.com`.
5. A company-owned Onshape OAuth application with this exact redirect URI:
   `https://<MCP_DOMAIN>/oauth/callback`.
6. The Onshape enterprise company ID configured for the integrated application.

Company-owned Onshape OAuth applications count against the company's API
limits. Confirm expected usage and annual limits with the Onshape administrator
before broad rollout.

## Configure

From this directory:

```bash
cp .env.example .env
mkdir -p secrets
openssl rand -base64 32 > secrets/state-encryption-key
printf '%s' 'replace-with-onshape-client-secret' > secrets/onshape-client-secret
sudo chown 10001:10001 secrets/state-encryption-key secrets/onshape-client-secret
sudo chmod 600 secrets/state-encryption-key secrets/onshape-client-secret
```

Edit `.env`. The allowlist format is:

```text
id[:display-name[:read|write|full]],id2[:display-name[:read|write|full]]
```

Examples:

```text
abc123:Alice:read,def456:Bob:write,ghi789:Release Engineer:full
```

Keep `.env` and `secrets/` out of source control. Store copies of both secrets
in the company's secret manager. Losing the state encryption key makes the
persisted OAuth state unrecoverable; exposing it together with the state volume
exposes active credentials.

## Start and verify

```bash
docker compose up -d --build
docker compose ps
curl --fail "https://${MCP_DOMAIN}/health"
curl --fail "https://${MCP_DOMAIN}/ready"
curl --fail "https://${MCP_DOMAIN}/.well-known/oauth-protected-resource/mcp"
```

The MCP client URL is `https://<MCP_DOMAIN>/mcp`. Each engineer completes the
browser OAuth flow once. Caddy preserves the public `Host` header because the
MCP transport rejects other authorities to prevent DNS rebinding.

## Operations

- Stream logs with `docker compose logs -f`. Application audit records use the
  event name `onshape_api_request` and include timestamp, Onshape user ID,
  configured access level, method, path, outcome, and upstream status. They do
  not include request bodies, query parameters, OAuth tokens, or Authorization
  headers. Caddy redacts credential headers by default, and this deployment
  additionally redacts OAuth `code` and `state` query values from access logs.
- Back up the `oauth-state` Docker volume and the encryption key separately.
  Test restore procedures. A state backup without its matching key is unusable.
- To revoke a user immediately, remove their allowlist entry and restart the
  service. Runtime bearer validation re-checks the current allowlist, so removed
  users are denied after restart even if old tokens remain in encrypted state.
  Also revoke the OAuth grant in Onshape when offboarding requires it.
- Changing the state encryption key without re-encrypting the existing file
  intentionally fails startup. For simple key rotation, stop the service,
  archive the old state and key under the retention policy, generate a new key,
  remove the old state volume, and have engineers authorize again.
- Base and proxy images are digest-pinned. Patch the host and deliberately
  update those pins and rebuild regularly. Run `cargo deny check` and the full
  test suite in CI for every dependency or image update.

At the company edge, add per-IP rate limiting for `/oauth/register`,
`/oauth/authorize`, `/oauth/token`, and `/mcp`; alert on repeated 401/403/429
responses and registration-capacity exhaustion. If the service is internet
reachable, place it behind the company's WAF/DDoS service while preserving the
original `Host` header.

## Production acceptance checklist

- TLS certificate and callback URI match the configured public URL exactly.
- Only approved Onshape user IDs can complete authorization.
- A read user cannot POST/PUT/PATCH/DELETE; a write user cannot DELETE.
- Restarting `onshape-mcp` preserves an existing client's refresh flow.
- The encrypted state file contains no recognizable token plaintext and is mode
  0600 inside the container.
- Backups restore successfully on a disposable host.
- Audit logs reach the company SIEM with an agreed retention period.
- Alerts cover readiness failures, restart loops, abnormal 4xx/5xx rates, and
  Onshape API-limit responses.

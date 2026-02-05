# n8n Integration — GitLab → Feedback-Hub

## Project overview

This n8n instance runs the [self-hosted AI starter kit](https://github.com/n8n-io/self-hosted-ai-starter-kit) with an added **GitLab → Feedback-Hub** integration workflow. When a new project is created in GitLab, a system hook fires a webhook to n8n, which forwards the event to the Feedback-Hub API.

```
GitLab (system hook) ──POST──▶ n8n (webhook) ──POST──▶ Feedback-Hub (API)
```

## Access & credentials

| What | How |
|---|---|
| n8n UI | `http://localhost:5678` |
| Owner email | `N8N_OWNER_EMAIL` in `.env` |
| Owner password | `N8N_OWNER_PASSWORD` in `.env` |
| REST API login | `POST /rest/login` with `{"email": "...", "password": "..."}` |
| Create API key | `POST /rest/api-keys` (authenticated session) |

All integration secrets (GitLab webhook token, Feedback-Hub API key) are stored in n8n's **encrypted credential vault** — not in `.env` or workflow JSON.

## Workflow: "GitLab → Feedback-Hub"

- **Webhook path:** `POST /webhook/gitlab-system-hook`
- **Flow:** Webhook (headerAuth) → IF (`event_name == project_create`) → HTTP Request (POST to Feedback-Hub)
- **Credential references:**
  - `GitLab Webhook Token` — credential id: `Iv1at78vEOpE1gcq`
  - `Feedback-Hub API Key` — credential id: `nk3ZRln1SUmgB9aH`

## GitLab system hook config

- **Hook URL:** `http://host.docker.internal:5678/webhook/gitlab-system-hook`
- `allow_local_requests_from_system_hooks` must be `true` in GitLab admin settings
- `enable_ssl_verification` must be `false` (the webhook is HTTP, not HTTPS)
- The `token` field must be **explicitly set** when creating the hook — creating a hook without the `token` field means **no token at all** (not an empty string, just absent)

## n8n gotchas

### Credential vault is the only reliable way to handle secrets

- `$env.VAR` access is blocked by default (`N8N_BLOCK_ENV_ACCESS_IN_NODE=true`)
- Even when unblocked, `$env` expressions behave inconsistently across node types
- Use `httpHeaderAuth` credentials + `genericCredentialType` on HTTP Request nodes instead

### HTTP Request node v4.2 — body format matters

- `sendBody: true` + `bodyParameters` (key-value pairs) sends **form-urlencoded**, not JSON
- For JSON bodies: use `specifyBody: "json"` + `jsonBody: "={{ JSON.stringify({...}) }}"`
- DRF (Django REST Framework) APIs return **403** on form-urlencoded bodies when API key auth expects a JSON content type

### HTTP Request node v4.2 — header format matters

- `sendHeaders: true` + `headerParameters` alone does **not** reliably send custom headers
- Use credential-based auth (`authentication: "genericCredentialType"`) instead of manual headers

### Workflow activation has a specific API flow

- CLI import (`n8n import:workflow`) always **deactivates** trigger workflows
- Setting `active=true` directly in the database does **not** register webhooks
- Correct activation: `POST /rest/workflows/{id}/activate` with `{"versionId": "..."}`
- `PATCH /rest/workflows/{id}` with `{"active": true}` does **not** activate — it only saves data

### Workflow creation — REST API is better than CLI import

- `POST /rest/workflows` creates the workflow and returns it with a `versionId`
- Then `POST /rest/workflows/{id}/activate` with that `versionId` to go live
- CLI import requires a container restart + a separate activation step

## Docker networking

- `extra_hosts: ["host.docker.internal:host-gateway"]` is set in `docker-compose.yml` on the `n8n` service
- This resolves `host.docker.internal` to `172.17.0.1` (the Docker bridge gateway)
- The GitLab container also needs `host.docker.internal` in `/etc/hosts` (typically already present)

## Useful commands

```bash
# List workflows
docker exec n8n n8n list:workflow

# Check n8n logs
docker logs n8n --tail 30

# Check recent executions via API
curl -s "http://localhost:5678/api/v1/executions?limit=5" \
  -H "X-N8N-API-KEY: $KEY"

# Verify GitLab hook has a token set
docker exec gitlab gitlab-rails runner "puts SystemHook.find(1).token.present?"

# Test webhook manually
curl -X POST http://localhost:5678/webhook/gitlab-system-hook \
  -H "Content-Type: application/json" \
  -H "X-Gitlab-Token: <token>" \
  -d '{"event_name":"project_create","name":"test","path":"test","project_id":99}'
```

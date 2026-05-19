<!-- canonical-for: ILLUMINATE_AUTH -->
<!-- used-by: -->

# Illuminate: Service Integration & API Key Setup

The canonical reference for getting an Illuminate API key and using it correctly.

## API Endpoint

All Illuminate operations go to:

```
https://admin-api.pubnub.com/v2/illuminate/...
```

Note that this is the PubNub Admin API, not the data plane. Subscribe and publish keys are separate from the Service Integration API key.

## Required Headers (Every Request)

```http
Authorization: <ILLUMINATE_API_KEY>
PubNub-Version: 2026-02-09
Content-Type: application/json   # for POST/PUT
```

Both `Authorization` and `PubNub-Version` are required. Missing either returns `401` or `400`.

## Creating a Service Integration

1. Log in to the [Admin Portal](https://admin.pubnub.com).
2. Click your account name → **My Account** → **Organization Settings** → **API Management**.
3. Click **Create Service Integration**. Give it a descriptive name like `Illuminate Automation - Production`.
4. Add permission rows at the **Account** level. Select **Illuminate** as the resource with **Read & write** access.
5. Click **Create**.
6. Click **+ Generate API Key**. **Copy it immediately — it is shown only once.**
7. Store in `ILLUMINATE_API_KEY` env var or in a secrets manager. Never commit to source control. See [pubnub-keyset-management/references/key-rotation-and-hygiene.md](../../pubnub-keyset-management/references/key-rotation-and-hygiene.md).

## Other Keys You Also Need

Some Illuminate operations require keyset-level keys, not the Service Integration key:

| Operation | Additional key required |
|---|---|
| Activate a Business Object (`subkeys` array) | Subscribe key (`sub-c-...`) — tells Illuminate which channel data to ingest |
| `PUBNUB_PUBLISH` Decision action | Publish key (`pub-c-...`) AND Subscribe key (`sub-c-...`) — Illuminate uses these to publish the action |

These are configured per-resource, not at request time. Take them from the keyset that owns the data you're analyzing.

## Per-Environment Service Integrations (Required)

Create a **separate Service Integration per environment** (dev / staging / production). Reasons:

- **Least privilege**: a dev integration with prod access is a leaked-key disaster.
- **Independent rotation**: rotate dev without disturbing prod.
- **Audit trail**: usage logs per environment.
- **Configuration parity**: dev resources reference dev keysets, etc.

Pattern:

```
Illuminate Automation - Dev        → ILLUMINATE_API_KEY_DEV
Illuminate Automation - Staging    → ILLUMINATE_API_KEY_STAGING
Illuminate Automation - Production → ILLUMINATE_API_KEY_PROD
```

See [pubnub-keyset-management/references/keysets-and-environments.md](../../pubnub-keyset-management/references/keysets-and-environments.md) for the broader environment-separation rule.

## Zero-Downtime Rotation

A single Service Integration can have up to **three active API keys** simultaneously. This makes rotation zero-downtime:

1. Generate a new API key (now you have 2 active).
2. Roll the new key into your secrets manager.
3. Wait for all callers to pick up the new key (deploy + cache TTL).
4. Revoke the old key in the Admin Portal.

If you must rotate immediately due to a leak, generate the new key first, then revoke the old one (callers will fail until they pick up the new one).

## Setting the Shortest Practical Expiration

Service Integration keys can be set to expire. Pick the shortest practical expiration:

- Production rotation script that runs annually: 13-month expiration.
- Quarterly rotation: 4-month expiration.
- Short-lived ETL job: hours or days.

Expiration plus 3-key support means you can have a clean handoff between old and new without scripting around revocation.

## Using the Key in Code

```bash
# Bash / curl
curl -X GET https://admin-api.pubnub.com/v2/illuminate/business-objects \
  -H "Authorization: $ILLUMINATE_API_KEY" \
  -H "PubNub-Version: 2026-02-09"
```

```javascript
// Node.js
const response = await fetch('https://admin-api.pubnub.com/v2/illuminate/business-objects', {
  method: 'GET',
  headers: {
    'Authorization': process.env.ILLUMINATE_API_KEY,
    'PubNub-Version': '2026-02-09'
  }
});
```

```python
# Python
import os, requests
r = requests.get(
    'https://admin-api.pubnub.com/v2/illuminate/business-objects',
    headers={
        'Authorization': os.environ['ILLUMINATE_API_KEY'],
        'PubNub-Version': '2026-02-09'
    }
)
```

## Common Auth Errors

| HTTP code | Cause | Fix |
|---|---|---|
| 401 | Missing or wrong `Authorization` header | Set `Authorization: <api_key>` (no `Bearer` prefix) |
| 400 | Missing `PubNub-Version` header | Add `PubNub-Version: 2026-02-09` |
| 403 | Service Integration lacks Illuminate Read & Write | Add the permission row in API Management |
| 401 | API key was revoked or expired | Generate a new key |

## MCP Tools

- **`manage_illuminate`** — handles auth header injection automatically when given the API key in configuration. Prefer this over raw HTTP for routine operations.

## Related Reading

- [business-objects.md](business-objects.md) — first resource type to create after auth
- [pubnub-keyset-management/references/key-rotation-and-hygiene.md](../../pubnub-keyset-management/references/key-rotation-and-hygiene.md) — secret storage and rotation discipline

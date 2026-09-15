# Key Rotation and Hygiene

**Canonical owner (S6):** Subscribe-key rotation as keyset migration workflow (not an isolated rotate primitive).

The canonical reference for rotating PubNub publish, subscribe, and secret keys, and for keeping keys out of source control and CI logs.

## Why Rotate

Rotate on leak, personnel change with secret-key access, or compliance ([SOC 2 / HIPAA](../../pubnub-security/references/compliance-reports.md)). Subscribe-key rotation is a **keyset migration**, not a drop-in rotate.

## Key-Type Rotation Difficulty

| Key | Rotation impact | Coordination needed |
|---|---|---|
| Publish key | All publishers must update | Deploy clients + servers, then retire old key |
| Subscribe key | All subscribers must update; **also rotates the entire keyset namespace** — channels and history are isolated per subscribe key | This is a **disruptive rotation** — usually performed by spinning up a new keyset |
| Secret key | Token grants must be re-issued | Server-only deployment + token re-grant; can be done in minutes |

> **Important**: rotating the **subscribe key** effectively migrates to a brand-new keyset. Existing message history, App Context data, and Access Manager grants on the old subscribe key remain there. Plan for a coordinated cutover, not a rolling rotation, when subscribe-key rotation is forced.

## Secret Key Rotation Procedure

The most common and least disruptive rotation. Follow this when:

- A secret key may have leaked
- An engineer with secret-key access leaves
- Routine annual rotation

### Procedure

1. **Generate a new secret key** in the Admin Portal (a single keyset can have at most one active secret key at a time, so this immediately retires the old one — plan accordingly).
2. **Update server secret store** with the new key. Restart server processes that hold the secret in memory.
3. **Re-grant outstanding Access Manager tokens** if your tokens are long-lived. Tokens issued by the old secret key remain valid until their TTL expires; new tokens require the new secret key.
4. **Verify** by issuing a new token from the rotated server and confirming it works on a client.
5. **Audit** secrets manager access logs to confirm no client is still using the old key.

For the actual `grantToken()` mechanics, see [pubnub-security/references/access-manager.md](../../pubnub-security/references/access-manager.md).

## Publish Key Rotation Procedure

Less common, only required when a publish key is genuinely compromised. The publish key on its own (without Access Manager) lets anyone publish to any unrestricted channel on your keyset.

### Procedure

1. **Decide whether to enable Access Manager first** if it isn't on. With Access Manager, a leaked publish key alone cannot publish to restricted channels — this often eliminates the need to rotate.
2. If still rotating: generate a new publish key in the Admin Portal.
3. **Deploy clients and servers** with the new key.
4. **Retire the old publish key** in the Admin Portal once all clients have been updated. Mobile clients still on old app versions will lose ability to publish.

## Subscribe Key Rotation (Migration)

Subscribe-key rotation is effectively a keyset migration because the subscribe key is the keyset namespace. Channels, history, App Context data, and Access Manager grants are all scoped to the subscribe key.

### Procedure

1. **Create a new keyset** in the Admin Portal with the same add-on configuration.
2. **Decide on history migration**: PubNub does not migrate history between keysets. Options:
   - Backfill recent history into the new keyset (publish replayed messages).
   - Run a parallel-read window where clients read from both keysets briefly.
   - Accept history loss for the cutover.
3. **Migrate App Context data** by exporting from the old keyset and importing to the new one (see [pubnub-app-context/references/users.md](../../pubnub-app-context/references/users.md) for the API).
4. **Re-grant Access Manager tokens** under the new secret key.
5. **Cutover clients** in a coordinated release.
6. **Retire the old keyset** after a verification window.

## Secrets Storage

Keep PubNub keys in a server-side secret store or CI-injected env vars. Never commit `pub-c-` / `sub-c-` / `sec-c-` values. A pre-commit grep on those prefixes is enough; if history already leaked a key, **rotate immediately** (subscribe-key leak = keyset migration above).

## Rotation Cadence

| Trigger | Cadence |
|---|---|
| No known incident, production secret key | Annually |
| Personnel change with secret-key access | Within 24 hours |
| Suspected leak | Immediately |
| Routine non-production keys | Annually or per policy |

Document who owns rotation and where the [runbook](../../pubnub-observability/references/incident-runbook.md) lives.

## MCP Tools

- **`manage_keysets`** — generate new keys, retire old ones, audit current key state

## Related Reading

- [keysets-and-environments.md](keysets-and-environments.md) — environment separation
- [pubnub-security/references/access-manager.md](../../pubnub-security/references/access-manager.md) — token grants that depend on the secret key

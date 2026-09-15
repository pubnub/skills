# PubNub Security Best Practices

PubNub-specific security sequencing. Key storage, environment split, and rotation live in [key-rotation-and-hygiene](../../pubnub-keyset-management/references/key-rotation-and-hygiene.md) and [keysets-and-environments](../../pubnub-keyset-management/references/keysets-and-environments.md). Token grant shapes live in [access-manager](access-manager.md). Cipher vs TLS vs secret-key decisions live in [encryption](encryption.md).

## Never Expose Secret Key

```javascript
// SERVER-SIDE ONLY
const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  secretKey: 'sec-c-...',
  userId: 'server'
});
```

```javascript
// CLIENT — no secretKey; apply AM via setToken
const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'user-123'
});
pubnub.setToken('token-from-server');
```

## Access control (PubNub-specific)

- Least privilege: `grantToken` scoped to the smallest channel set per role — see [access-manager](access-manager.md).
- Channel names are security boundaries (`public-*` vs `private-user-{id}` vs admin prefixes). Encode membership in grants, not in client logic.
- Role patterns: grant via `patterns.channels` (e.g. `^chat-.*$`) rather than enumerating every room.

Short TTL for sensitive resources; longer TTL for session chat. Refresh by fetching a new token and calling `setToken` before expiry — do not invent a generic OAuth manager here.

## Handling Access Denied

```javascript
pubnub.addListener({
  status: (statusEvent) => {
    if (statusEvent.category === 'PNAccessDeniedCategory') {
      void (async () => {
        await refreshAuthToken();
        pubnub.subscribe({ channels: statusEvent.affectedChannels });
      })();
    }
  }
});
```

## Validate on Server

Before-Publish Functions can abort unauthorized or malformed publishes. Keep the handler as a channel-prefix check + `request.abort()`; full counter/validation patterns live in [functions-patterns](../../pubnub-functions/references/functions-patterns.md).

```javascript
export default async (request) => {
  const channel = request.channels[0];
  if (channel.startsWith('user-')) {
    const channelUserId = channel.split('-')[1];
    if (request.message.senderId !== channelUserId) return request.abort();
  }
  return request.ok();
};
```

## Compliance and audit

Admin Portal → My Account → Compliance for reports (Pro download; Free/Starter via support). Persist audit events on a dedicated channel with Message Persistence; correlate with [logging-correlation](../../pubnub-observability/references/logging-correlation.md).

## Security Checklist

- [ ] Access Manager enabled
- [ ] Secret key server-only; clients use `setToken()` (not constructor `authKey`)
- [ ] Short TTLs for sensitive grants; refresh before expiry
- [ ] `PNAccessDeniedCategory` handler
- [ ] Cipher via CryptoModule for sensitive payloads ([encryption](encryption.md))
- [ ] Separate keysets per environment
- [ ] Server-side publish validation where clients must not be trusted

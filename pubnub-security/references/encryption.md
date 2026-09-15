# Encryption — Decisions and Tradeoffs


## What to encrypt

| Data | Encrypted with client cipher / CryptoModule |
|------|---------------------------------------------|
| Message payload | Yes |
| File metadata / caption on `sendFile` | Yes |
| Channel name | No |
| Publisher UUID | No |
| Timetoken | No |
| Message Persistence (history) | Stored encrypted when published encrypted |

## Layering: cipher vs secret key vs TLS

| Layer | Purpose | Where it lives | PubNub sees plaintext? |
|-------|---------|----------------|------------------------|
| **CryptoModule / cipher key** | End-to-end message payload encryption | Client (shared among trusted peers) | No |
| **Secret key** | Sign Access Manager admin requests | Server only | N/A |
| **TLS** | Transport encryption client ↔ PubNub edge | Default on | Terminates at edge |

Use all three for defense in depth on sensitive channels.

## Algorithm choice

- Prefer **`CryptoModule.aesCbcCryptoModule`** for true 256-bit AES-CBC on new work.
- Do **not** set top-level `cipherKey` / `useRandomIVs` on SDK configuration — those parameters are deprecated and used the legacy ~128-bit path. Pass the key into CryptoModule instead.
- Never expose cipher keys or secret keys in client bundles.

```javascript
const pubnub = new PubNub({
  publishKey: process.env.PN_PUBLISH_KEY,
  subscribeKey: process.env.PN_SUBSCRIBE_KEY,
  userId: persistentUserId,
  cryptoModule: PubNub.CryptoModule.aesCbcCryptoModule({
    cipherKey: process.env.PN_CIPHER_KEY // AES-256-CBC
  })
});
```

Retrieve the current CryptoModule constructor for Python, Swift, and Kotlin via **`get_sdk_documentation`** (`feature: encryption`) — do not copy the JavaScript shape into other SDKs.

## File encryption pattern

PubNub encrypts file **metadata/caption** with the cipher key; the file binary is protected by HTTPS and platform at-rest security.

For **true end-to-end file content** (PubNub never sees plaintext bytes):

1. Encrypt file bytes in your app **before** `sendFile`.
2. Recipient downloads and decrypts locally with the same key material you manage.

## History and wrong keys

Encrypted messages in Persistence remain encrypted at rest. Fetching history with a missing or wrong cipher key returns encrypted blobs — plan key rotation and dual-read windows accordingly.

## Orchestration

1. Choose cipher scope (global vs per-channel vs per-user keysets).
2. Retrieve current SDK CryptoModule API for the target language.
3. Configure on client init; verify round-trip publish/subscribe and `fetchMessages` playback.
4. For HIPAA/regulated payloads, pair with [Access Manager](access-manager.md) and [compliance posture](compliance-reports.md).

<!-- canonical-for: ENCRYPTION, TLS -->
<!-- used-by: -->

> **Cross-references:** Cipher key handling pairs with [keyset / secret key hygiene and rotation](../../pubnub-keyset-management/references/key-rotation-and-hygiene.md) (and [the keyset model](../../pubnub-keyset-management/references/keysets-and-environments.md)). Encryption is configured at [SDK initialization](../../pubnub-app-developer/references/sdk-patterns.md) and applies to [`pubnub.publish(`](../../pubnub-app-developer/references/publish-subscribe.md) and to [Message Persistence / fetchMessages playback](../../pubnub-history/references/pagination-and-ordering.md). [File Sharing / sendFile](../../pubnub-chat/references/file-sharing.md) uses the same cipher key. For [Functions-side crypto module](../../pubnub-functions/references/functions-modules.md) the same algorithm and cipher-key story applies.

# PubNub Encryption Guide

## AES-256 Message Encryption

PubNub provides built-in AES-256 encryption for end-to-end message security.

### How It Works

1. Publisher encrypts message with `cipherKey` before sending
2. Message travels encrypted through PubNub network
3. Subscriber decrypts message with same `cipherKey`
4. **PubNub never has access to your cipher key**

### Enabling Encryption

```javascript
const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'user-123',
  cipherKey: 'my-strong-secret-cipher-key'  // AES-256 encryption enabled
});

// All messages automatically encrypted/decrypted
await pubnub.publish({
  channel: 'secure-channel',
  message: { secretData: 'sensitive information' }
});
```

### Python

```python
pnconfig = PNConfiguration()
pnconfig.publish_key = 'pub-c-...'
pnconfig.subscribe_key = 'sub-c-...'
pnconfig.uuid = 'user-123'
pnconfig.cipher_key = 'my-strong-secret-cipher-key'

pubnub = PubNub(pnconfig)
```

### Swift

```swift
let config = PubNubConfiguration(
    publishKey: "pub-c-...",
    subscribeKey: "sub-c-...",
    userId: "user-123"
)
config.cipherKey = "my-strong-secret-cipher-key"

let pubnub = PubNub(configuration: config)
```

## What Gets Encrypted

| Data | Encrypted with cipherKey |
|------|-------------------------|
| Message payload | Yes |
| File metadata/caption | Yes |
| Channel name | No |
| Publisher UUID | No |
| Timetoken | No |
| Message persistence (history) | Stored encrypted |

## Key Management Best Practices

### Generating Strong Cipher Keys

```javascript
// Generate a strong cipher key
const crypto = require('crypto');
const cipherKey = crypto.randomBytes(32).toString('hex');
// e.g., "a1b2c3d4e5f6..."
```

### Per-Channel or Per-User Keys

```javascript
// Different cipher keys for different security contexts
const publicPubnub = new PubNub({
  subscribeKey: 'sub-c-...',
  publishKey: 'pub-c-...',
  userId: 'user-123'
  // No cipherKey - public channels
});

const privatePubnub = new PubNub({
  subscribeKey: 'sub-c-...',
  publishKey: 'pub-c-...',
  userId: 'user-123',
  cipherKey: 'private-channel-key'
});
```

### Rotating Cipher Keys

```javascript
// When rotating keys, old messages remain encrypted with old key
// Consider using versioned keys

const cipherKeys = {
  v1: 'old-cipher-key',
  v2: 'new-cipher-key'
};

// Include version in message for decryption
await pubnub.publish({
  channel: 'secure-channel',
  message: {
    keyVersion: 'v2',
    data: encryptedPayload
  }
});
```

## TLS Configuration

### TLS is Enabled by Default

```javascript
// TLS (HTTPS) is ON by default
const pubnub = new PubNub({
  subscribeKey: 'sub-c-...',
  userId: 'user-123'
  // ssl: true is the default
});
```

### Disabling TLS (Not Recommended)

```javascript
// Only disable for specific testing scenarios
const pubnub = new PubNub({
  subscribeKey: 'sub-c-...',
  userId: 'user-123',
  ssl: false  // NOT RECOMMENDED for production
});
```

### TLS 1.2 Requirement

> **Important**: As of February 1, 2025, PubNub only supports TLS 1.2+

**Verify TLS 1.2 compatibility:**
```bash
# Test endpoint
curl https://pubsub-tls12-test.pubnub.com
```

**Ensure your environment supports TLS 1.2:**
- Node.js 12+
- Python 3.6+
- Modern browsers (Chrome 30+, Firefox 27+, Safari 7+)
- iOS 9+, Android 5.0+

## Cipher Key vs Secret Key vs TLS

| Key Type | Purpose | Location | PubNub Access |
|----------|---------|----------|---------------|
| **Cipher Key** | Encrypt message payloads | Client | Never sees it |
| **Secret Key** | Sign Access Manager admin requests | Server only | Has it |
| **TLS** | Encrypt transport layer | Both | Terminates at edge |

### Defense in Depth

```javascript
// Maximum security: Use ALL three
const pubnub = new PubNub({
  subscribeKey: 'sub-c-...',
  publishKey: 'pub-c-...',
  userId: 'user-123',
  authKey: 'access-manager-auth-token', // Access Manager (legacy authKey)
  cipherKey: 'my-strong-cipher-key',   // AES-256 encryption
  ssl: true                             // TLS (default)
});
```

## File Encryption

When using PubNub File Sharing:

```javascript
// File metadata is encrypted with cipherKey
// File binary uses PubNub infrastructure security (HTTPS + at-rest encryption)

const pubnub = new PubNub({
  subscribeKey: 'sub-c-...',
  publishKey: 'pub-c-...',
  userId: 'user-123',
  cipherKey: 'my-cipher-key'
});

// Send file - metadata encrypted
await pubnub.sendFile({
  channel: 'file-sharing',
  file: myFile,
  message: { caption: 'Encrypted caption' }  // This is encrypted
});
```

### True End-to-End File Encryption

For file content encryption where PubNub never sees plaintext:

```javascript
// 1. Encrypt file content yourself BEFORE sending
const encryptedFileBuffer = await encryptFile(originalFile, myEncryptionKey);

// 2. Send encrypted file
await pubnub.sendFile({
  channel: 'secure-files',
  file: {
    data: encryptedFileBuffer,
    name: 'encrypted-document.bin'
  }
});

// 3. Recipient downloads and decrypts
const downloadedFile = await pubnub.downloadFile({
  channel: 'secure-files',
  id: fileId,
  name: fileName
});

const decryptedFile = await decryptFile(downloadedFile, myEncryptionKey);
```

## Message Persistence with Encryption

Messages encrypted with `cipherKey` are stored encrypted:

```javascript
// Publish encrypted message
await pubnub.publish({
  channel: 'secure-history',
  message: { secret: 'data' }
});

// Retrieve history - automatically decrypted if same cipherKey
const history = await pubnub.fetchMessages({
  channels: ['secure-history'],
  count: 10
});

// Messages in history.channels['secure-history'] are decrypted
```

**Note**: If you fetch history with wrong/no cipherKey, messages appear as encrypted blobs.

<!-- canonical-for: FILE_SHARING -->
<!-- used-by: -->

> **Cross-references:** Built on [SDK initialization (`new PubNub(`, `userId`/UUID)](../../pubnub-app-developer/references/sdk-patterns.md). Files are encrypted with the same [CryptoModule cipher key](../../pubnub-security/references/encryption.md) as messages when configured. [Access Manager grants](../../pubnub-security/references/access-manager.md) on a channel govern who can `sendFile`. Storage limits per [keyset add-on](../../pubnub-keyset-management/references/keysets-and-environments.md). Watch transfer volume on [usage metrics](../../pubnub-observability/references/usage-metrics.md).

# File Sharing

PubNub Files lets users send arbitrary files (images, PDFs, audio, video) on the same channel as messages. The Chat SDK wraps the underlying `pubnub.sendFile` API.

## Architecture

1. Client uploads file bytes to PubNub File storage (per-keyset bucket).
2. PubNub publishes a **file message** on the channel — small JSON containing `fileId`, `fileName`, content type, optional caption.
3. Subscribers receive the file message; if they want the file bytes, they call `pubnub.downloadFile`.

The bytes are not in the real-time message; the file message is just a pointer.

## Send a File

```javascript
await channel.sendFile({
  file: fileObject,
  caption: 'See attached'
});
```

Lower-level:

```javascript
await pubnub.sendFile({
  channel: 'support',
  message: { caption: 'See attached' },
  file: { name: 'report.pdf', mimeType: 'application/pdf', data: blob }
});
```

## Receive Files

The file shows up in the regular message stream as a `messageType: 'file'` event:

```javascript
channel.connect((message) => {
  if (message.files && message.files.length > 0) {
    const file = message.files[0];
    console.log('File:', file.name, file.id);
  }
});
```

To download the bytes:

```javascript
const fileBlob = await pubnub.downloadFile({
  channel: 'support',
  id: file.id,
  name: file.name
});
```

## File URL (Pre-Signed)

For browser display you can request a direct URL:

```javascript
const url = await pubnub.getFileUrl({ channel: 'support', id: file.id, name: file.name });
```

URLs are signed and time-limited.

## Encryption

If the SDK has a CryptoModule configured, file bytes are encrypted client-side before upload and decrypted client-side after download. See the [encryption owner](../../pubnub-security/references/encryption.md).

## Constraints

- **Per-file size cap** — typically 5 MB for the standard Files add-on; up to 25 MB on enterprise. Confirm via Admin Portal.
- **Per-keyset storage cap** — varies by plan. Old files are not auto-deleted; build your own GC if needed.
- **MIME type sniffing** — done client-side; PubNub does not validate the content matches `mimeType`.
- **No partial / resumable uploads** — large file failures must be retried from the start (see [queue-and-retry owner](../../pubnub-reliability/references/queue-and-retry.md)).
- **Files API is async** — `sendFile` returns when upload + publish complete. Don't block UI on it.

## Listing Files in a Channel

```javascript
const result = await pubnub.listFiles({ channel: 'support', limit: 25 });
result.data.forEach(f => console.log(f.id, f.name, f.size));
```

## Deleting Files

```javascript
await pubnub.deleteFile({ channel: 'support', id: fileId, name: 'report.pdf' });
```

This deletes from PubNub File storage; the file message in history still references it (the reference will 404 on download).

## Common Pitfalls

| Pitfall | Mitigation |
|---|---|
| Big file blocks the UI thread | Stream/upload off the main thread (Web Worker, native background) |
| File uploaded but message never published | Wrap in try/catch; retry the whole `sendFile` call (idempotent if you reuse the file) |
| Receivers don't see the file | Confirm they're subscribed to the channel and have appropriate Access Manager grants |
| Storage growing unbounded | Run a periodic cleanup job using `listFiles` + `deleteFile` |

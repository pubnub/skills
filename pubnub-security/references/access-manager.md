# PubNub Access Manager (PAM)

## Overview

PubNub Access Manager provides fine-grained control over who can access channels and what actions they can perform.

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Permissions** | Read (subscribe), Write (publish), Get, Update, Manage, Delete, Join |
| **Token** | JWT-like token issued by `grantToken()` containing embedded permissions |
| **Secret Key** | Server-side key for issuing and revoking tokens |
| **TTL** | Time-to-live for tokens (in minutes) |

## Enabling Access Manager

1. Log in to PubNub Admin Portal
2. Select your Application and Keyset
3. Enable **Access Manager** add-on
4. **Securely store your Secret Key** - required for token grants

> **Critical**: Once enabled, all clients need valid tokens to access channels.

## Server-Side Setup (Node.js)

```javascript
import PubNub from 'pubnub';

const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  secretKey: 'sec-c-...',  // Server-side only!
  userId: 'server'
});
```

## Token-Based Access (Recommended)

### Grant Token for Specific Resources

```javascript
// Grant a user access to specific channels
async function issueUserToken(userId, channels, ttlMinutes = 60) {
  try {
    const token = await pubnub.grantToken({
      ttl: ttlMinutes,
      authorizedUUID: userId,
      resources: {
        channels: Object.fromEntries(
          channels.map(ch => [ch, { read: true, write: true }])
        )
      }
    });
    console.log('Token issued:', token);
    return token;
  } catch (error) {
    console.error('Token grant failed:', error);
    throw error;
  }
}

// Usage
const token = await issueUserToken('user-123', ['chat-room-private', 'notifications']);
```

### Grant Token with Channel Patterns

```javascript
// Grant access to all channels matching a pattern
const token = await pubnub.grantToken({
  ttl: 60,
  authorizedUUID: 'user-123',
  patterns: {
    channels: {
      'chat-room-*': { read: true, write: true }
    }
  }
});
```

### Grant Token with Mixed Resources and Patterns

```javascript
const token = await pubnub.grantToken({
  ttl: 60,
  authorizedUUID: 'user-123',
  resources: {
    channels: {
      'private-room': { read: true, write: true, get: true, update: true }
    },
    uuids: {
      'user-123': { get: true, update: true }
    }
  },
  patterns: {
    channels: {
      'public-*': { read: true }
    }
  }
});
```

### Grant Token for Channel Groups

```javascript
const token = await pubnub.grantToken({
  ttl: 60,
  authorizedUUID: 'user-123',
  resources: {
    groups: {
      'user-feeds-group': { read: true, manage: true }
    }
  }
});
```

## Client-Side Configuration

### Setting the Token on the Client

```javascript
const pubnub = new PubNub({
  subscribeKey: 'sub-c-...',
  publishKey: 'pub-c-...',
  userId: 'user-123'
});

// Set the token received from your server
pubnub.setToken(token);
```

### Parsing Token Permissions (Debugging)

```javascript
// Inspect what a token grants
const parsed = pubnub.parseToken(token);
console.log('Token permissions:', JSON.stringify(parsed, null, 2));
```

## Revoking Tokens

```javascript
// Revoke a specific token
await pubnub.revokeToken(token);
```

> **Note**: Revocations may take up to 60 seconds to propagate due to caching.

## Authentication Flow

### Complete Server Flow

```javascript
import express from 'express';
import PubNub from 'pubnub';
import jwt from 'jsonwebtoken';

const app = express();

// Server PubNub instance with Secret Key
const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  secretKey: 'sec-c-...',
  userId: 'server'
});

// Auth endpoint
app.post('/api/pubnub/auth', async (req, res) => {
  try {
    // 1. Verify user's session (your auth logic)
    const userToken = req.headers.authorization?.split(' ')[1];
    const user = jwt.verify(userToken, process.env.JWT_SECRET);

    // 2. Determine channels based on user permissions
    const channels = getUserChannels(user);

    // 3. Issue PubNub token
    const channelPermissions = {};
    for (const ch of channels) {
      channelPermissions[ch] = { read: true, write: user.canWrite };
    }

    const token = await pubnub.grantToken({
      ttl: 60,  // 1 hour, then client must re-auth
      authorizedUUID: user.id,
      resources: { channels: channelPermissions }
    });

    // 4. Return credentials to client
    res.json({
      token: token,
      subscribeKey: 'sub-c-...',
      publishKey: user.canWrite ? 'pub-c-...' : null,
      channels: channels,
      expiresAt: Date.now() + (60 * 60 * 1000)  // 1 hour
    });
  } catch (error) {
    console.error('Auth error:', error);
    res.status(401).json({ error: 'Authentication failed' });
  }
});
```

### Client Flow

```javascript
async function initializePubNub() {
  // 1. Get token from your server
  const response = await fetch('/api/pubnub/auth', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${userSessionToken}`
    }
  });

  const credentials = await response.json();

  // 2. Initialize PubNub and set token
  const pubnub = new PubNub({
    subscribeKey: credentials.subscribeKey,
    publishKey: credentials.publishKey,
    userId: currentUserId
  });

  pubnub.setToken(credentials.token);

  // 3. Handle access denied errors
  pubnub.addListener({
    status: (statusEvent) => {
      if (statusEvent.category === 'PNAccessDeniedCategory') {
        console.error('Access denied - refreshing token');
        refreshToken();
      }
    }
  });

  // 4. Schedule token refresh before expiry
  const refreshTime = credentials.expiresAt - Date.now() - 300000;  // 5 min before
  setTimeout(refreshToken, refreshTime);

  return pubnub;
}
```

## Permission Levels

### Token Permissions

| Permission | Allows |
|------------|--------|
| `read` | Subscribe to channel, fetch history |
| `write` | Publish messages to channel |
| `get` | Get channel/UUID metadata |
| `update` | Set channel/UUID metadata |
| `manage` | Add/remove channels in channel groups |
| `delete` | Delete messages |
| `join` | Join channel as a member |

## TTL Best Practices

| Use Case | Recommended TTL |
|----------|-----------------|
| Short session (demo) | 15-60 minutes |
| Normal user session | 60-1440 minutes (1-24 hours) |
| Long-lived service | 1440-10080 minutes (1-7 days) |
| Maximum | 43200 minutes (30 days) |

```javascript
// Short TTL for sensitive operations
const token = await pubnub.grantToken({
  ttl: 15,  // 15 minutes
  authorizedUUID: userId,
  resources: {
    channels: {
      'payment-updates': { read: true }
    }
  }
});
```

## Legacy: grant() and authKey

For older implementations using the `grant()` API with `authKey`:

```javascript
// Server-side grant (legacy)
await pubnub.grant({
  channels: ['private-room'],
  authKeys: ['user-auth-token'],
  read: true,
  write: true,
  ttl: 60
});

// Client-side with authKey (legacy)
const pubnub = new PubNub({
  subscribeKey: 'sub-c-...',
  publishKey: 'pub-c-...',
  userId: 'user-123',
  authKey: 'auth-token-from-server'
});
```

> **Note**: New implementations should use `grantToken()` and `setToken()` instead.

<!-- canonical-for: SECURITY_BEST_PRACTICES -->
<!-- used-by: -->

> **Cross-references:** Foundational [keyset structure, the publish/subscribe/secret keys, environment separation, and rotation](../../pubnub-keyset-management/references/keysets-and-environments.md) (and the [key-rotation owner](../../pubnub-keyset-management/references/key-rotation-and-hygiene.md)) are owned by `pubnub-keyset-management`. This file focuses on application-layer security patterns. For runtime hardening see [IP allowlisting](ip-whitelisting.md) and [DoS mitigation](dos-mitigation.md). For [SDK initialization (`new PubNub(`, `userId`/UUID)](../../pubnub-app-developer/references/sdk-patterns.md), [`pubnub.publish(`/`pubnub.subscribe(`](../../pubnub-app-developer/references/publish-subscribe.md), and [channel naming](../../pubnub-app-developer/references/channels.md) see those owners. For [Message Persistence](../../pubnub-history/references/pagination-and-ordering.md) retention rules see `pubnub-history`. Function-side patterns ([Before Publish, `request.ok()`/`request.abort()`](../../pubnub-functions/references/functions-basics.md) and [`require('crypto')`](../../pubnub-functions/references/functions-modules.md)) are owned by `pubnub-functions`.

# PubNub Security Best Practices

## Key Security

### Never Expose Secret Key

```javascript
// SERVER-SIDE ONLY
const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  secretKey: 'sec-c-...',  // ONLY on server
  userId: 'server'
});
```

```javascript
// CLIENT-SIDE - No Secret Key
const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'user-123',
  authKey: 'token-from-server'
  // NO secretKey here
});
```

### Store Keys Securely

```javascript
// Use environment variables
const pubnub = new PubNub({
  publishKey: process.env.PUBNUB_PUB_KEY,
  subscribeKey: process.env.PUBNUB_SUB_KEY,
  secretKey: process.env.PUBNUB_SECRET_KEY,  // Server only
  userId: 'server'
});
```

### Separate Keys per Environment

```
Production:
  - pub-c-prod-...
  - sub-c-prod-...
  - sec-c-prod-...

Staging:
  - pub-c-staging-...
  - sub-c-staging-...
  - sec-c-staging-...

Development:
  - pub-c-dev-...
  - sub-c-dev-...
  - sec-c-dev-...
```

## Access Control Patterns

### Principle of Least Privilege

```javascript
// Grant minimum required permissions
await pubnub.grant({
  channels: ['user-feed'],
  authKeys: [userAuthKey],
  read: true,    // Can subscribe
  write: false,  // Cannot publish
  ttl: 60
});

// Only grant write for specific channels
await pubnub.grant({
  channels: ['user-posts'],
  authKeys: [userAuthKey],
  read: false,
  write: true,
  ttl: 60
});
```

### Channel-Based Authorization

```javascript
// Design channels with security boundaries

// Public - no auth required (if Access Manager disabled)
'public-announcements'

// Private user channel
'private-user-${userId}'

// Group with membership check
'group-${groupId}'  // Grant only to members

// Admin-only
'admin-dashboard'  // Grant only to admin role
```

### Role-Based Access

```javascript
async function grantRoleAccess(userId, role, authKey) {
  const rolePermissions = {
    viewer: {
      channels: ['public-*'],
      read: true,
      write: false
    },
    member: {
      channels: ['public-*', 'chat-*'],
      read: true,
      write: true
    },
    moderator: {
      channels: ['public-*', 'chat-*', 'mod-*'],
      read: true,
      write: true,
      manage: true
    },
    admin: {
      channels: ['*'],
      read: true,
      write: true,
      manage: true
    }
  };

  const perms = rolePermissions[role];
  await pubnub.grant({
    channels: perms.channels,
    authKeys: [authKey],
    read: perms.read,
    write: perms.write,
    manage: perms.manage || false,
    ttl: 1440  // 24 hours
  });
}
```

## Token Management

### Short TTLs for Sensitive Data

```javascript
// Sensitive operations: short TTL
await pubnub.grant({
  channels: ['financial-data'],
  authKeys: [authKey],
  read: true,
  ttl: 15  // 15 minutes
});

// Regular operations: moderate TTL
await pubnub.grant({
  channels: ['chat-room'],
  authKeys: [authKey],
  read: true,
  write: true,
  ttl: 60  // 1 hour
});
```

### Token Refresh Strategy

```javascript
// Client-side token refresh
class PubNubAuthManager {
  constructor() {
    this.refreshBuffer = 5 * 60 * 1000;  // 5 minutes before expiry
  }

  async initialize() {
    const credentials = await this.fetchCredentials();
    this.setupPubNub(credentials);
    this.scheduleRefresh(credentials.expiresAt);
  }

  async fetchCredentials() {
    const response = await fetch('/api/pubnub/auth', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${sessionToken}`
      }
    });
    return response.json();
  }

  setupPubNub(credentials) {
    this.pubnub = new PubNub({
      subscribeKey: credentials.subscribeKey,
      publishKey: credentials.publishKey,
      userId: credentials.userId,
      authKey: credentials.authKey
    });
  }

  scheduleRefresh(expiresAt) {
    const refreshTime = expiresAt - Date.now() - this.refreshBuffer;
    setTimeout(() => this.refresh(), Math.max(refreshTime, 0));
  }

  async refresh() {
    const credentials = await this.fetchCredentials();
    this.pubnub.setAuthKey(credentials.authKey);
    this.scheduleRefresh(credentials.expiresAt);
  }
}
```

## Handling Access Denied

```javascript
pubnub.addListener({
  status: (statusEvent) => {
    switch (statusEvent.category) {
      case 'PNAccessDeniedCategory':
        console.error('Access denied');
        console.error('Channels:', statusEvent.affectedChannels);

        // Option 1: Refresh token and retry
        refreshAuthToken().then(() => {
          pubnub.subscribe({
            channels: statusEvent.affectedChannels
          });
        });

        // Option 2: Redirect to login
        // window.location.href = '/login';

        // Option 3: Show error to user
        // showError('Session expired. Please log in again.');
        break;
    }
  }
});
```

## Secure Channel Design

### Channel Naming Conventions

```javascript
// Include user context in private channels
`dm-${sortedUserIds.join('-')}`

// Include ownership in channels
`user-${ownerId}-posts`

// Use prefixes for different security levels
`public-announcements`
`private-user-123`
`secret-admin-channel`
```

### Validate on Server

```javascript
// PubNub Function: Validate before publish
export default async (request) => {
  const message = request.message;
  const channel = request.channels[0];

  // Validate sender has permission for this channel
  if (channel.startsWith('user-')) {
    const channelUserId = channel.split('-')[1];
    if (message.senderId !== channelUserId) {
      console.error('Unauthorized publish attempt');
      return request.abort();
    }
  }

  // Validate message structure
  if (!message.senderId || !message.content) {
    return request.abort();
  }

  return request.ok();
};
```

## Encryption Best Practices

### Use Strong Cipher Keys

```javascript
// Generate cryptographically secure key
const crypto = require('crypto');
const cipherKey = crypto.randomBytes(32).toString('hex');
```

### Separate Keys by Security Context

```javascript
// Different cipher keys for different data sensitivity
const publicPubnub = new PubNub({
  subscribeKey: 'sub-c-...',
  userId: 'user-123'
  // No encryption for public channels
});

const privatePubnub = new PubNub({
  subscribeKey: 'sub-c-...',
  userId: 'user-123',
  cipherKey: 'private-conversations-key'
});

const financialPubnub = new PubNub({
  subscribeKey: 'sub-c-...',
  userId: 'user-123',
  cipherKey: 'financial-data-key'  // Different key for sensitive data
});
```

## Compliance and Audit

### Finding Compliance Reports

1. Go to Admin Portal
2. Select "My Account" dropdown
3. Click "Compliance"
4. Pro accounts: Download reports directly
5. Free/Starter: Request reports from support

### Message Audit Trail

```javascript
// Use Message Persistence for audit logs
await pubnub.publish({
  channel: 'audit-log',
  message: {
    action: 'user_login',
    userId: user.id,
    timestamp: new Date().toISOString(),
    ip: request.ip,
    userAgent: request.headers['user-agent']
  }
});

// Messages stored for configured retention period
```

## Security Checklist

- [ ] Access Manager enabled in Admin Portal
- [ ] Secret Key stored securely, never in client code
- [ ] Using `authKey` (not `token`) in client configuration
- [ ] Short TTLs for sensitive operations
- [ ] Token refresh implemented before expiry
- [ ] Access denied handler implemented
- [ ] TLS 1.2+ verified for all connections
- [ ] Cipher key used for sensitive messages
- [ ] Channel naming follows security conventions
- [ ] Server-side validation for publishes
- [ ] Separate keysets for dev/staging/production
- [ ] Audit logging enabled for compliance

<!-- canonical-for: CHAT_SDK -->
<!-- used-by: -->

> **Cross-references:** Built on [SDK initialization (`new PubNub(`, `userId`/UUID)](../../pubnub-app-developer/references/sdk-patterns.md), [pub/sub basics](../../pubnub-app-developer/references/publish-subscribe.md), and [keyset/secret-key handling](../../pubnub-keyset-management/references/keysets-and-environments.md). For [Access Manager / `grantToken`](../../pubnub-security/references/access-manager.md) integration with `authKey` and [end-to-end message encryption](../../pubnub-security/references/encryption.md) see `pubnub-security`. User and channel records flow through [App Context](../../pubnub-app-context/references/users.md).

# PubNub Chat SDK Setup

## Installation

### JavaScript/TypeScript

```bash
npm install @pubnub/chat
# or
yarn add @pubnub/chat
```

### Prerequisites

- Node.js >= 18.10.0
- Publish and Subscribe keys from PubNub Admin Portal
- **App Context** enabled in keyset settings

## Basic Initialization

```javascript
import { Chat } from '@pubnub/chat';

const chat = await Chat.init({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'user-123'
});
```

## Configuration Options

```javascript
const chat = await Chat.init({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'unique-user-id',           // REQUIRED - persistent per user

  // Optional configurations
  authKey: 'access-manager-auth-token', // For Access Manager (NOT 'token')
  cipherKey: 'encryption-key',        // For message encryption

  // Typing indicator settings
  typingTimeout: 5000,                // ms before typing indicator expires
  storeUserActivityTimestamps: true,  // Track user activity

  // Rate limiting
  rateLimitFactor: 2,                 // Multiplier for rate limits
  rateLimitPerChannel: 20             // Messages per minute per channel
});
```

## Access Manager Integration

### Correct Configuration

```javascript
// Use authKey for client configuration
const chat = await Chat.init({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'user-123',
  authKey: 'auth-token-from-server'  // CORRECT
});
```

### WRONG Configuration

```javascript
// DO NOT use 'token' parameter
const chat = await Chat.init({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'user-123',
  token: 'auth-token-from-server'    // WRONG - will not work
});
```

## User Setup

### Get or Create Current User

```javascript
// Current user is available after init
const currentUser = chat.currentUser;

// Update current user's profile
await currentUser.update({
  name: 'Alice Smith',
  profileUrl: 'https://example.com/avatar.png',
  custom: {
    role: 'member',
    preferences: { theme: 'dark' }
  }
});
```

### Get or Create Other Users

```javascript
// Get existing user
let user = await chat.getUser('other-user-id');

// Create if doesn't exist
if (!user) {
  user = await chat.createUser('other-user-id', {
    name: 'Bob Jones',
    profileUrl: 'https://example.com/bob.png'
  });
}
```

### Pattern: Get or Create User

```javascript
async function getOrCreateUser(chat, userId, userData) {
  let user = await chat.getUser(userId);
  if (!user) {
    user = await chat.createUser(userId, userData);
  }
  return user;
}

// Usage
const interlocutor = await getOrCreateUser(chat, 'bob-123', {
  name: 'Bob',
  profileUrl: 'https://example.com/bob.png'
});
```

## React Integration

### Basic React Setup

```javascript
import { useState, useEffect } from 'react';
import { Chat } from '@pubnub/chat';

function ChatApp({ userId }) {
  const [chat, setChat] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let mounted = true;

    async function initChat() {
      try {
        const chatInstance = await Chat.init({
          publishKey: process.env.REACT_APP_PUBNUB_PUB_KEY,
          subscribeKey: process.env.REACT_APP_PUBNUB_SUB_KEY,
          userId: userId
        });

        if (mounted) {
          setChat(chatInstance);
          setLoading(false);
        }
      } catch (err) {
        if (mounted) {
          setError(err);
          setLoading(false);
        }
      }
    }

    initChat();

    return () => {
      mounted = false;
      // Cleanup will be handled by chat.disconnect()
    };
  }, [userId]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return <ChatInterface chat={chat} />;
}
```

### Context Provider Pattern

```javascript
import { createContext, useContext, useState, useEffect } from 'react';
import { Chat } from '@pubnub/chat';

const ChatContext = createContext(null);

export function ChatProvider({ userId, children }) {
  const [chat, setChat] = useState(null);

  useEffect(() => {
    let chatInstance = null;

    Chat.init({
      publishKey: process.env.REACT_APP_PUBNUB_PUB_KEY,
      subscribeKey: process.env.REACT_APP_PUBNUB_SUB_KEY,
      userId
    }).then(instance => {
      chatInstance = instance;
      setChat(instance);
    });

    return () => {
      if (chatInstance) {
        chatInstance.disconnect();
      }
    };
  }, [userId]);

  return (
    <ChatContext.Provider value={chat}>
      {children}
    </ChatContext.Provider>
  );
}

export function useChat() {
  return useContext(ChatContext);
}
```

## Environment Configuration

### .env File

```
REACT_APP_PUBNUB_PUB_KEY=pub-c-...
REACT_APP_PUBNUB_SUB_KEY=sub-c-...
```

### Vite

```
VITE_PUBNUB_PUB_KEY=pub-c-...
VITE_PUBNUB_SUB_KEY=sub-c-...
```

```javascript
const chat = await Chat.init({
  publishKey: import.meta.env.VITE_PUBNUB_PUB_KEY,
  subscribeKey: import.meta.env.VITE_PUBNUB_SUB_KEY,
  userId: currentUserId
});
```

## Cleanup and Logout

```javascript
// Disconnect and cleanup
async function logout(chat) {
  await chat.disconnect();
  // Clear any cached channels
  Object.keys(activeChannels).forEach(key => delete activeChannels[key]);
}

// React cleanup
useEffect(() => {
  return () => {
    if (chat) {
      chat.disconnect();
    }
  };
}, [chat]);
```

## Error Handling

```javascript
try {
  const chat = await Chat.init({
    publishKey: 'pub-c-...',
    subscribeKey: 'sub-c-...',
    userId: 'user-123'
  });
} catch (error) {
  if (error.status?.category === 'PNAccessDeniedCategory') {
    console.error('Access denied - check authKey');
    // Refresh auth token
  } else if (error.message?.includes('userId')) {
    console.error('Invalid userId');
  } else {
    console.error('Chat initialization failed:', error);
  }
}
```

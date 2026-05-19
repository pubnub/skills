<!-- canonical-for: CHAT_PATTERNS -->
<!-- used-by: -->

> **Cross-references:** User management is owned by [App Context users](../../pubnub-app-context/references/users.md) and [channels & memberships](../../pubnub-app-context/references/channels-and-memberships.md) — link there rather than duplicating CRUD. [Presence (online/offline)](../../pubnub-presence/references/presence-events.md) and [multi-device sync](../../pubnub-presence/references/multi-device-sync.md) are owned by `pubnub-presence`. For [Access Manager grants](../../pubnub-security/references/access-manager.md) on chat channels see `pubnub-security`. For [reliable scrollback (history fetch + dedup-on-merge)](../../pubnub-history/references/offline-catch-up.md) and [idempotent send](../../pubnub-reliability/references/idempotent-publish.md) see those owners. Built on [pub/sub basics (`pubnub.publish(`, `pubnub.subscribe(`)](../../pubnub-app-developer/references/publish-subscribe.md) and [SDK initialization (`new PubNub(`, `userId`/UUID)](../../pubnub-app-developer/references/sdk-patterns.md).

# PubNub Chat SDK Patterns

## User Management

### Get or Create User Pattern

```javascript
async function getOrCreateUser(chat, userId, userData) {
  let user = await chat.getUser(userId);

  if (!user) {
    user = await chat.createUser(userId, userData);
  }

  return user;
}

// Usage
const users = await Promise.all([
  getOrCreateUser(chat, 'alice-123', { name: 'Alice', profileUrl: '...' }),
  getOrCreateUser(chat, 'bob-456', { name: 'Bob', profileUrl: '...' })
]);
```

### Update User Profile

```javascript
// Update current user
await chat.currentUser.update({
  name: 'Alice Smith',
  profileUrl: 'https://example.com/alice-new.png',
  email: 'alice@example.com',
  custom: {
    status: 'online',
    bio: 'Hello, I am Alice!'
  }
});

// Update other user (if permitted)
const user = await chat.getUser('bob-456');
await user.update({ name: 'Robert Jones' });
```

### List Users

```javascript
// Get all users
const users = await chat.getUsers({
  limit: 100,
  filter: "custom.status == 'online'"
});

users.users.forEach(user => {
  console.log(`${user.id}: ${user.name}`);
});
```

## Channel Management

### Channel Caching

```javascript
const activeChannels = new Map();

async function getOrCreateDirectChannel(chat, otherUser) {
  const cacheKey = [chat.currentUser.id, otherUser.id].sort().join('_');

  if (activeChannels.has(cacheKey)) {
    return activeChannels.get(cacheKey);
  }

  // Check if channel exists
  const existingChannels = await chat.getChannels({
    filter: `type == 'direct'`
  });

  for (const channel of existingChannels.channels) {
    const members = await channel.getMembers();
    const memberIds = members.members.map(m => m.user.id);
    if (memberIds.includes(otherUser.id)) {
      activeChannels.set(cacheKey, channel);
      return channel;
    }
  }

  // Create new channel
  const { channel } = await chat.createDirectConversation({
    user: otherUser,
    channelData: { name: `DM: ${otherUser.name}` }
  });

  activeChannels.set(cacheKey, channel);
  return channel;
}
```

### List User's Channels

```javascript
// Get channels user is member of
const memberships = await chat.currentUser.getMemberships({
  limit: 50
});

memberships.memberships.forEach(membership => {
  const channel = membership.channel;
  console.log(`${channel.id}: ${channel.name}`);
});
```

### Join/Leave Channel

```javascript
// Join public channel
await channel.join();

// Leave channel
await channel.leave();
```

## Real-time Message Sync

### Complete Chat Room Component

```javascript
import { useState, useEffect, useCallback } from 'react';

function ChatRoom({ chat, channelId }) {
  const [channel, setChannel] = useState(null);
  const [messages, setMessages] = useState([]);
  const [text, setText] = useState('');
  const [typing, setTyping] = useState([]);

  // Initialize channel
  useEffect(() => {
    if (!chat) return;

    async function loadChannel() {
      const ch = await chat.getChannel(channelId);
      setChannel(ch);

      // Load message history
      const history = await ch.getHistory({ count: 50 });
      setMessages(history.messages);
    }

    loadChannel();
  }, [chat, channelId]);

  // Subscribe to messages
  useEffect(() => {
    if (!channel) return;

    const unsubscribe = channel.connect((message) => {
      setMessages(prev => [...prev, message]);
    });

    return () => unsubscribe();
  }, [channel]);

  // Subscribe to typing indicators
  useEffect(() => {
    if (!channel) return;

    const unsubscribe = channel.onTyping((typingUserIds) => {
      setTyping(typingUserIds.filter(id => id !== chat.currentUser.id));
    });

    return () => unsubscribe();
  }, [channel, chat]);

  // Handle typing
  const handleTyping = useCallback(() => {
    if (channel) {
      channel.startTyping();
    }
  }, [channel]);

  // Handle send
  const handleSend = async (e) => {
    e.preventDefault();
    if (!text.trim() || !channel) return;

    await channel.sendText(text);
    setText('');
    channel.stopTyping();
  };

  return (
    <div className="chat-room">
      <div className="messages">
        {messages.map(msg => (
          <Message key={msg.timetoken} message={msg} />
        ))}
      </div>

      {typing.length > 0 && (
        <div className="typing-indicator">
          {typing.join(', ')} typing...
        </div>
      )}

      <form onSubmit={handleSend}>
        <input
          value={text}
          onChange={(e) => {
            setText(e.target.value);
            handleTyping();
          }}
          placeholder="Type a message..."
        />
        <button type="submit">Send</button>
      </form>
    </div>
  );
}

function Message({ message }) {
  return (
    <div className="message">
      <span className="author">{message.userId}</span>
      <span className="text">{message.text}</span>
      <span className="time">
        {new Date(message.timetoken / 10000).toLocaleTimeString()}
      </span>
    </div>
  );
}
```

## Notifications Pattern

### Unread Badge

```javascript
function useUnreadCounts(chat) {
  const [unreadCounts, setUnreadCounts] = useState({});

  useEffect(() => {
    if (!chat) return;

    async function fetchUnreadCounts() {
      const memberships = await chat.currentUser.getMemberships();
      const counts = {};

      for (const membership of memberships.memberships) {
        const channel = membership.channel;
        const count = await channel.getUnreadMessagesCount();
        counts[channel.id] = count;
      }

      setUnreadCounts(counts);
    }

    fetchUnreadCounts();

    // Re-fetch periodically or on message events
    const interval = setInterval(fetchUnreadCounts, 30000);
    return () => clearInterval(interval);
  }, [chat]);

  return unreadCounts;
}
```

### Mark Channel as Read

```javascript
async function markChannelRead(channel) {
  // Get latest message
  const history = await channel.getHistory({ count: 1 });
  if (history.messages.length > 0) {
    await channel.setLastReadMessage(history.messages[0]);
  }
}

// Call when user views channel
useEffect(() => {
  if (channel && isVisible) {
    markChannelRead(channel);
  }
}, [channel, isVisible]);
```

## Threading Pattern

```javascript
function ThreadView({ parentMessage }) {
  const [thread, setThread] = useState(null);
  const [replies, setReplies] = useState([]);

  useEffect(() => {
    async function loadThread() {
      const t = await parentMessage.getThread();
      setThread(t);

      const history = await t.getHistory({ count: 50 });
      setReplies(history.messages);
    }

    if (parentMessage.hasThread) {
      loadThread();
    }
  }, [parentMessage]);

  useEffect(() => {
    if (!thread) return;

    const unsubscribe = thread.connect((message) => {
      setReplies(prev => [...prev, message]);
    });

    return () => unsubscribe();
  }, [thread]);

  const sendReply = async (text) => {
    if (thread) {
      await thread.sendText(text);
    } else {
      // Create thread with first reply
      const t = await parentMessage.createThread(text);
      setThread(t);
    }
  };

  return (
    <div className="thread">
      <div className="parent-message">
        {parentMessage.text}
      </div>
      <div className="replies">
        {replies.map(reply => (
          <div key={reply.timetoken}>{reply.text}</div>
        ))}
      </div>
      <ReplyInput onSend={sendReply} />
    </div>
  );
}
```

## Cleanup and Logout

```javascript
async function logout(chat, activeChannels) {
  // Disconnect from PubNub
  await chat.disconnect();

  // Clear channel cache
  activeChannels.clear();

  // Clear local state
  setChat(null);
  setMessages([]);
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

## Performance Tips

1. **Cache channels** - Don't recreate channels on every render
2. **Paginate history** - Load messages in chunks, not all at once
3. **Debounce typing** - Don't send typing signal on every keystroke
4. **Virtualize messages** - Use windowing for long message lists
5. **Batch updates** - Group state updates to reduce re-renders
6. **Cleanup subscriptions** - Always unsubscribe on unmount

```javascript
// Debounced typing example
const debouncedTyping = useMemo(
  () => debounce(() => channel?.startTyping(), 1000),
  [channel]
);

const handleChange = (e) => {
  setText(e.target.value);
  debouncedTyping();
};
```

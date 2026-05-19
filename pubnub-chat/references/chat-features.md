<!-- canonical-for: CHAT_FEATURES -->
<!-- used-by: -->

> **Cross-references:** [App Context user/channel/membership metadata](../../pubnub-app-context/references/users.md) is the canonical owner for users and channels. [Presence (online/offline, here-now)](../../pubnub-presence/references/presence-events.md) is owned by `pubnub-presence`. For [reactions, edits, deletes, read receipts](message-actions.md), [file sharing / `sendFile`](file-sharing.md), and [threading](threading.md) see the canonical owners in this skill. Built on [pub/sub basics](../../pubnub-app-developer/references/publish-subscribe.md) and [SDK initialization (`new PubNub(`, `userId`/UUID)](../../pubnub-app-developer/references/sdk-patterns.md).

# PubNub Chat SDK Features

## Channel Types

### Direct Channel (1:1 Chat)

```javascript
// Create direct conversation with another user
const interlocutor = await chat.getUser('bob-123') ||
                     await chat.createUser('bob-123', { name: 'Bob' });

const { channel } = await chat.createDirectConversation({
  user: interlocutor,
  channelData: { name: 'Chat with Bob' }
});
```

### Group Channel

```javascript
// Create group channel with multiple members
const { channel } = await chat.createGroupConversation({
  users: [user1, user2, user3],
  channelId: 'team-alpha',
  channelData: {
    name: 'Team Alpha',
    description: 'Project discussions',
    custom: { type: 'project', visibility: 'members' }
  }
});
```

### Public Channel

```javascript
// Create public channel anyone can join
const channel = await chat.createPublicConversation({
  channelId: 'public-lobby',
  channelData: {
    name: 'Public Lobby',
    description: 'Open discussion'
  }
});
```

## Sending and Receiving Messages

### Connect to Channel

```javascript
// Connect and receive messages
const unsubscribe = channel.connect((message) => {
  console.log('Message from:', message.userId);
  console.log('Text:', message.text);
  console.log('Time:', message.timetoken);
});

// Later: disconnect
unsubscribe();
```

### React Pattern

```javascript
useEffect(() => {
  if (!channel) return;

  const unsubscribe = channel.connect((message) => {
    setMessages(prev => [...prev, message]);
  });

  return () => {
    unsubscribe();
  };
}, [channel]);
```

### Send Text Message

```javascript
await channel.sendText('Hello, everyone!');

// With metadata
await channel.sendText('Check this out!', {
  meta: {
    linkPreview: 'https://example.com',
    priority: 'high'
  }
});
```

### Get Message History

```javascript
// Fetch past messages
const history = await channel.getHistory({
  count: 50,
  startTimetoken: '17000000000000000'  // Optional: pagination
});

history.messages.forEach(msg => {
  console.log(`${msg.userId}: ${msg.text}`);
});
```

## Typing Indicators

### Send Typing Signal

```javascript
// Start typing indicator
await channel.startTyping();

// Stop typing indicator
await channel.stopTyping();
```

### Listen for Typing

```javascript
// Get who's currently typing
const typingUsers = await channel.getTyping();

// Listen for typing changes
channel.onTyping((typingUserIds) => {
  if (typingUserIds.length > 0) {
    const names = typingUserIds.map(id => getUserName(id));
    setTypingIndicator(`${names.join(', ')} typing...`);
  } else {
    setTypingIndicator('');
  }
});
```

### Typing Indicator Component

```javascript
function TypingIndicator({ channel }) {
  const [typing, setTyping] = useState([]);

  useEffect(() => {
    if (!channel) return;

    const unsubscribe = channel.onTyping((userIds) => {
      setTyping(userIds);
    });

    return () => unsubscribe();
  }, [channel]);

  if (typing.length === 0) return null;

  return (
    <div className="typing-indicator">
      {typing.length === 1 && `${typing[0]} is typing...`}
      {typing.length === 2 && `${typing[0]} and ${typing[1]} are typing...`}
      {typing.length > 2 && `${typing.length} people are typing...`}
    </div>
  );
}
```

## Message Reactions

### Add Reaction

```javascript
// Add emoji reaction to message
await message.toggleReaction('thumbsup');
await message.toggleReaction('heart');
await message.toggleReaction('laugh');
```

### Get Reactions

```javascript
// Get all reactions on a message
const reactions = message.reactions;
// { thumbsup: ['user-1', 'user-2'], heart: ['user-3'] }
```

### Listen for Reaction Updates

```javascript
channel.on('reaction', ({ event, data }) => {
  console.log(`Reaction ${event}:`, data);
  // event: 'added' or 'removed'
  // data: { messageTimetoken, reactionType, userId }

  // Update local message state
  updateMessageReactions(data.messageTimetoken, data);
});
```

### React Reactions Component

```javascript
function MessageReactions({ message, currentUserId }) {
  const handleReaction = async (emoji) => {
    await message.toggleReaction(emoji);
  };

  return (
    <div className="reactions">
      {Object.entries(message.reactions || {}).map(([emoji, users]) => (
        <button
          key={emoji}
          onClick={() => handleReaction(emoji)}
          className={users.includes(currentUserId) ? 'active' : ''}
        >
          {emoji} {users.length}
        </button>
      ))}
      <button onClick={() => handleReaction('thumbsup')}>+</button>
    </div>
  );
}
```

## Read Receipts

### Mark as Read

```javascript
// Mark message as read
await message.setLastReadMessage();

// Mark channel as read up to timetoken
await channel.setLastReadMessage(message);
```

### Get Unread Count

```javascript
// Get unread message count for channel
const unreadCount = await channel.getUnreadMessagesCount();

// Get unread counts for all channels
const channels = await chat.getChannels();
for (const channel of channels) {
  const unread = await channel.getUnreadMessagesCount();
  console.log(`${channel.name}: ${unread} unread`);
}
```

## Message Threading

### Create Thread

```javascript
// Get thread from a message
const thread = await message.getThread();

// If no thread exists, it will be created when you reply
```

### Reply in Thread

```javascript
// Send reply to thread
await thread.sendText('This is a reply in the thread');

// Get thread messages
const threadHistory = await thread.getHistory({ count: 20 });
```

### Thread Count

```javascript
// Check if message has replies
if (message.hasThread) {
  const replyCount = message.threadRootMessage?.repliesCount || 0;
  console.log(`${replyCount} replies`);
}
```

## User Mentions

### Mention Users

```javascript
// Mention user in message
await channel.sendText('Hey @alice, check this out!', {
  mentionedUsers: {
    0: { id: 'alice-123', name: 'Alice' }  // Position 0 of mention
  }
});
```

### Listen for Mentions

```javascript
// Get messages where current user is mentioned
const mentions = await chat.getMentions({
  userId: chat.currentUser.id,
  count: 20
});

mentions.forEach(mention => {
  console.log(`Mentioned in ${mention.channel}: ${mention.message.text}`);
});
```

## Channel References

```javascript
// Reference another channel in message
await channel.sendText('Check out #general for updates', {
  referencedChannels: {
    0: { id: 'general', name: 'General' }
  }
});
```

## Files and Media

### Send File

```javascript
// Send file with message
await channel.sendFile({
  file: fileObject,  // File from input
  message: 'Check out this document'
});
```

### Download File

```javascript
// Get file URL from message
if (message.files && message.files.length > 0) {
  const file = message.files[0];
  console.log('File name:', file.name);
  console.log('File URL:', file.url);
}
```

## Time Utilities

```javascript
import { TimetokenUtils } from '@pubnub/chat';

// Convert timetoken to Date
const date = TimetokenUtils.timetokenToDate(message.timetoken);

// Format time display
const timeString = date.toLocaleTimeString([], {
  hour: '2-digit',
  minute: '2-digit'
});

// Convert Date to timetoken
const timetoken = TimetokenUtils.dateToTimetoken(new Date());
```

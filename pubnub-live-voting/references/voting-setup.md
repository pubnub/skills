# PubNub Live Voting Setup

This reference covers poll creation, channel architecture, SDK initialization, and lifecycle management for building real-time voting and polling systems with PubNub.

## SDK Initialization for Voting Apps

Both admin and participant clients use the same SDK but with different user IDs and permissions.

### Admin Client

```javascript
import PubNub from 'pubnub';

const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'admin-001',
  authKey: 'admin-auth-token'
});
```

### Participant Client

```javascript
const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'user-5432',
  authKey: 'participant-auth-token'
});
```

### Python Initialization

```python
from pubnub.pnconfiguration import PNConfiguration
from pubnub.pubnub import PubNub

config = PNConfiguration()
config.publish_key = "pub-c-..."
config.subscribe_key = "sub-c-..."
config.user_id = "admin-001"
config.auth_key = "admin-auth-token"

pubnub = PubNub(config)
```

## Channel Design for Voting

Each poll uses a set of dedicated channels for vote submission, result broadcasting, and administrative control.

### Channel Naming Conventions

| Channel Pattern | Purpose | Who Publishes | Who Subscribes |
|----------------|---------|---------------|----------------|
| `poll.<pollId>.votes` | Vote submission | Participants | PubNub Functions (server-side) |
| `poll.<pollId>.results` | Live tally updates | PubNub Functions | All clients |
| `poll.<pollId>.admin` | Poll lifecycle control | Admin only | All clients |
| `poll.<pollId>.meta` | Poll metadata and config | Admin only | Participants on join |
| `polls.directory` | List of active polls | Admin only | Participants browsing |

### Channel Group Setup

For applications with many simultaneous polls, use channel groups to manage subscriptions efficiently.

```javascript
await pubnub.channelGroups.addChannels({
  channelGroup: 'poll-2024-finale-group',
  channels: [
    'poll.poll-2024-finale.votes',
    'poll.poll-2024-finale.results',
    'poll.poll-2024-finale.admin'
  ]
});

pubnub.subscribe({ channelGroups: ['poll-2024-finale-group'] });
```

## Poll Types and Configurations

| Poll Type | Description | Max Selections | Use Case |
|-----------|-------------|----------------|----------|
| `single-choice` | One option per voter | 1 | Yes/No questions, winner selection |
| `multiple-choice` | Multiple options per voter | Configurable | "Select all that apply" surveys |
| `ranked-choice` | Voters rank options by preference | All options | Elimination voting |
| `rating` | Score each option on a scale | N/A | Satisfaction surveys, NPS |
| `open-text` | Free-form text responses | N/A | Feedback, audience questions |

### Poll Configuration Object

```javascript
const pollConfig = {
  pollId: 'poll-quarterly-feedback',
  question: 'How satisfied are you with this quarter?',
  description: 'Rate your overall experience this quarter.',
  type: 'single-choice',
  options: [
    { id: 'opt-1', label: 'Very Satisfied', order: 1 },
    { id: 'opt-2', label: 'Satisfied', order: 2 },
    { id: 'opt-3', label: 'Neutral', order: 3 },
    { id: 'opt-4', label: 'Dissatisfied', order: 4 },
    { id: 'opt-5', label: 'Very Dissatisfied', order: 5 }
  ],
  settings: {
    allowChangeVote: false,
    anonymousVoting: false,
    showLiveResults: true,
    maxVotesPerUser: 1,
    resultVisibility: 'after-vote' // 'live', 'after-vote', 'after-close'
  },
  schedule: {
    opensAt: null,        // null = manually opened
    closesAt: null,       // null = manually closed
    durationMs: 300000    // fallback: auto-close after 5 minutes
  },
  createdBy: 'admin-001',
  createdAt: Date.now()
};
```

## Poll Lifecycle Management

Every poll moves through a defined set of states. Transitions are published on the admin channel so all clients stay synchronized.

### Lifecycle States

| State | Description | Allowed Transitions |
|-------|-------------|---------------------|
| `created` | Poll defined but not yet visible | `open`, `deleted` |
| `open` | Accepting votes | `paused`, `closed` |
| `paused` | Temporarily not accepting votes | `open`, `closed` |
| `closed` | No longer accepting votes | `finalized` |
| `finalized` | Results locked and published | None (terminal) |

### Publishing Lifecycle Events

```javascript
async function openPoll(pubnub, pollId) {
  await pubnub.publish({
    channel: `poll.${pollId}.admin`,
    message: { action: 'poll_status_changed', pollId, status: 'open', timestamp: Date.now() }
  });
  await pubnub.publish({
    channel: `poll.${pollId}.meta`,
    message: { action: 'poll_config', poll: pollConfig }
  });
}

async function closePoll(pubnub, pollId) {
  await pubnub.publish({
    channel: `poll.${pollId}.admin`,
    message: { action: 'poll_status_changed', pollId, status: 'closed', timestamp: Date.now() }
  });
}

async function finalizePoll(pubnub, pollId, finalResults) {
  await pubnub.publish({
    channel: `poll.${pollId}.admin`,
    message: {
      action: 'poll_status_changed', pollId, status: 'finalized',
      results: finalResults, timestamp: Date.now()
    }
  });
}
```

### Listening for Lifecycle Events on the Client

```javascript
pubnub.subscribe({ channels: [`poll.${pollId}.admin`] });

pubnub.addListener({
  message: (event) => {
    const { action, status } = event.message;
    if (action === 'poll_status_changed') {
      switch (status) {
        case 'open': enableVotingUI(); break;
        case 'paused': showPausedBanner(); disableVotingUI(); break;
        case 'closed': disableVotingUI(); showClosedMessage(); break;
        case 'finalized': displayFinalResults(event.message.results); break;
      }
    }
  }
});
```

## Admin Controls

### Admin Dashboard Setup

```javascript
function setupAdminDashboard(pubnub, pollId) {
  pubnub.subscribe({
    channels: [
      `poll.${pollId}.admin`,
      `poll.${pollId}.results`,
      `poll.${pollId}.votes`
    ]
  });

  pubnub.addListener({
    message: (event) => {
      if (event.channel.endsWith('.results')) updateAdminTallyDisplay(event.message);
      else if (event.channel.endsWith('.votes')) logIncomingVote(event.message);
    }
  });
}
```

### Access Manager Permissions

```javascript
// Grant admin full access
await pubnub.grant({
  channels: [
    `poll.${pollId}.admin`, `poll.${pollId}.results`,
    `poll.${pollId}.votes`, `poll.${pollId}.meta`
  ],
  authKeys: ['admin-auth-token'],
  read: true, write: true, ttl: 60
});

// Grant participants vote-only access
await pubnub.grant({
  channels: [`poll.${pollId}.votes`],
  authKeys: ['participant-group-token'],
  read: false, write: true, ttl: 60
});

// Grant participants read-only on results/admin/meta
await pubnub.grant({
  channels: [`poll.${pollId}.results`, `poll.${pollId}.admin`, `poll.${pollId}.meta`],
  authKeys: ['participant-group-token'],
  read: true, write: false, ttl: 60
});
```

## Auto-Close with Timers

Use client-side timers for display and server-side validation for enforcement.

```javascript
function startPollCountdown(pollId, closesAt) {
  const interval = setInterval(() => {
    const remaining = closesAt - Date.now();
    if (remaining <= 0) { clearInterval(interval); displayTimeUp(); return; }
    updateCountdownDisplay(Math.ceil(remaining / 1000));
  }, 1000);
}
```

The actual close-time enforcement happens server-side in a Before Publish Function (see `voting-tallying.md`).

## Error Handling

```javascript
try {
  await pubnub.publish({
    channel: `poll.${pollId}.admin`,
    message: { action: 'poll_status_changed', pollId, status: 'open' }
  });
} catch (error) {
  if (error.status && error.status.statusCode === 403) {
    console.error('Access denied. Verify admin auth token and Access Manager grants.');
  } else {
    console.error('Failed to publish poll event:', error);
  }
}
```

## Best Practices

- **Use consistent channel naming**: Follow the `poll.<pollId>.<purpose>` pattern for all channels to keep the system organized and permissions manageable.
- **Separate vote submission from result broadcasting**: Never let clients read the votes channel directly. Route votes through PubNub Functions for validation before results are published.
- **Publish poll configuration on the meta channel**: This allows late-joining participants to fetch the poll question and options without a separate API call.
- **Set TTLs on Access Manager grants**: Always use time-limited grants that match the expected poll duration to minimize security exposure.
- **Use channel groups for multi-poll dashboards**: When an admin manages many polls simultaneously, channel groups reduce subscription overhead.
- **Store poll configurations in a backend database**: PubNub channels are for real-time messaging; persist poll definitions, final results, and audit logs in your own database.
- **Plan for reconnection**: Implement PubNub status listeners to detect disconnects and resubscribe. Fetch the latest poll state from the meta channel on reconnect.
- **Test with simulated load**: Before a live event, simulate concurrent vote publishing to validate your channel design handles the expected throughput.

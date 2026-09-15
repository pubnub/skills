# PubNub Live Voting Setup

Poll channels, lifecycle publish, Access Manager, and auto-close. Duplicate SDK init dumps and poll-type product catalogs are out of scope — pull init from MCP.

Admin vs participant: same SDK, different `userId` + AM token.

## Channel design

| Channel Pattern | Purpose | Who Publishes | Who Subscribes |
|----------------|---------|---------------|----------------|
| `poll.<pollId>.votes` | Vote submission | Participants | PubNub Functions |
| `poll.<pollId>.results` | Live tally updates | PubNub Functions | All clients |
| `poll.<pollId>.admin` | Poll lifecycle control | Admin only | All clients |
| `poll.<pollId>.meta` | Poll metadata and config | Admin only | Participants on join |
| `polls.directory` | List of active polls | Admin only | Participants browsing |

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

Poll type (`single-choice`, `multiple-choice`, etc.) is an operator field on the config object. Do not compare voting systems here.

```javascript
const pollConfig = {
  pollId: 'poll-quarterly-feedback',
  question: 'How satisfied are you with this quarter?',
  type: 'single-choice',
  options: [
    { id: 'opt-1', label: 'Very Satisfied', order: 1 },
    { id: 'opt-2', label: 'Satisfied', order: 2 }
  ],
  settings: {
    allowChangeVote: false,
    anonymousVoting: false,
    showLiveResults: true,
    maxVotesPerUser: 1,
    resultVisibility: 'after-vote'
  },
  schedule: { opensAt: null, closesAt: null, durationMs: 300000 },
  createdBy: 'admin-001',
  createdAt: Date.now()
};
```

## Lifecycle publish

| State | Allowed transitions |
|-------|---------------------|
| `created` | `open`, `deleted` |
| `open` | `paused`, `closed` |
| `paused` | `open`, `closed` |
| `closed` | `finalized` |
| `finalized` | terminal |

```javascript
async function openPoll(pubnub, pollId, pollConfig) {
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

```javascript
pubnub.subscribe({ channels: [`poll.${pollId}.admin`] });

pubnub.addListener({
  message: (event) => {
    const { action, status } = event.message;
    if (action === 'poll_status_changed') {
      switch (status) {
        case 'open': enableVotingUI(); break;
        case 'paused': disableVotingUI(); break;
        case 'closed': disableVotingUI(); break;
        case 'finalized': displayFinalResults(event.message.results); break;
      }
    }
  }
});
```

## Access Manager

```javascript
await pubnub.grantToken({
  ttl: 60,
  authorized_uuid: adminId,
  resources: {
    channels: {
      [`poll.${pollId}.admin`]: { read: true, write: true },
      [`poll.${pollId}.results`]: { read: true, write: true },
      [`poll.${pollId}.votes`]: { read: true, write: true },
      [`poll.${pollId}.meta`]: { read: true, write: true }
    }
  }
});

await pubnub.grantToken({
  ttl: 60,
  authorized_uuid: participantId,
  resources: {
    channels: {
      [`poll.${pollId}.votes`]: { write: true },
      [`poll.${pollId}.results`]: { read: true },
      [`poll.${pollId}.admin`]: { read: true },
      [`poll.${pollId}.meta`]: { read: true }
    }
  }
});
```

## Auto-close

Client countdown is display-only. Close-time enforcement is Before-Publish ([voting-tallying.md](voting-tallying.md) `closesAt`).

## Best Practices

- **`poll.<pollId>.<purpose>`** naming for AM patterns.
- **Functions own `.votes`** — clients subscribe to `.results` / `.admin` / `.meta`.
- **Config on `.meta`** for late joiners.
- **AM TTLs** matched to poll duration.
- **Channel groups** for multi-poll dashboards.
- **Reconnect:** resubscribe and refetch `.meta`.

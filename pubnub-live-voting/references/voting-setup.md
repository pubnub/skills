<!-- xrefs-injected -->

> **Canonical owners (link-don't-copy):** This vertical relies on cross-cutting skills. Always link to the canonical owner instead of duplicating. Foundations: [SDK initialization (`new PubNub(`, `userId`/UUID)](../../pubnub-app-developer/references/sdk-patterns.md), [pub/sub basics (`pubnub.publish(`, `pubnub.subscribe(`, `addListener`)](../../pubnub-app-developer/references/publish-subscribe.md), [channel naming](../../pubnub-app-developer/references/channels.md), [message filters](../../pubnub-app-developer/references/message-filters.md), [SDK upgrades](../../pubnub-app-developer/references/sdk-upgrades.md), [REST API](../../pubnub-app-developer/references/rest-api.md). Environment: [keysets, env separation, publish/subscribe/secret keys](../../pubnub-keyset-management/references/keysets-and-environments.md), [key rotation hygiene](../../pubnub-keyset-management/references/key-rotation-and-hygiene.md), [demo keys](../../pubnub-keyset-management/references/demo-keys.md), [custom origin](../../pubnub-keyset-management/references/custom-origin.md). Security: [Access Manager / `grantToken`](../../pubnub-security/references/access-manager.md), [AES-256 / message encryption](../../pubnub-security/references/encryption.md), [IP allowlisting](../../pubnub-security/references/ip-whitelisting.md), [DoS mitigation](../../pubnub-security/references/dos-mitigation.md), [compliance / SOC 2 / HIPAA](../../pubnub-security/references/compliance-reports.md). Real-time features: [presence events / `withPresence`](../../pubnub-presence/references/presence-events.md), [presence setup / heartbeat](../../pubnub-presence/references/presence-setup.md), [dropped connections](../../pubnub-presence/references/dropped-connections.md), [multi-device sync](../../pubnub-presence/references/multi-device-sync.md). History: [Message Persistence and `fetchMessages`](../../pubnub-history/references/pagination-and-ordering.md), [offline catch-up](../../pubnub-history/references/offline-catch-up.md), [retention](../../pubnub-history/references/retention-and-storage.md). App Context: [users / user metadata](../../pubnub-app-context/references/users.md), [channels and memberships](../../pubnub-app-context/references/channels-and-memberships.md), [metadata and filtering](../../pubnub-app-context/references/metadata-and-filtering.md). Functions: [Before/After Publish, `request.ok()`/`request.abort()`](../../pubnub-functions/references/functions-basics.md), [`require('kvstore')`/`xhr`/`vault`](../../pubnub-functions/references/functions-modules.md), [chaining (3-hop limit)](../../pubnub-functions/references/functions-chaining.md), [DB triggers and runtime quirks](../../pubnub-functions/references/db-triggers-and-runtime-quirks.md), [common patterns](../../pubnub-functions/references/functions-patterns.md). Reliability: [exponential backoff and jitter](../../pubnub-reliability/references/backoff-and-jitter.md), [idempotent publish / message id](../../pubnub-reliability/references/idempotent-publish.md), [dedup on merge](../../pubnub-reliability/references/dedup-on-merge.md), [queue and retry](../../pubnub-reliability/references/queue-and-retry.md), [schema version](../../pubnub-reliability/references/schema-versioning.md). Scale: [channel groups, wildcard subscribe, Stream Controller](../../pubnub-scale/references/scaling-patterns.md), [performance tuning](../../pubnub-scale/references/performance.md), [10K+ live events](../../pubnub-scale/references/large-events.md). Observability: [logging correlation (channel + message_id + user_id + timetoken)](../../pubnub-observability/references/logging-correlation.md), [test pyramid](../../pubnub-observability/references/test-pyramid.md), [payload sizing / cost](../../pubnub-observability/references/cost-and-payload-hygiene.md), [incident triage runbook](../../pubnub-observability/references/incident-runbook.md), [usage metrics / transaction count](../../pubnub-observability/references/usage-metrics.md). Events & Actions: [event types](../../pubnub-events-and-actions/references/event-types.md), [action targets (webhook / SQS / Kafka / Lambda)](../../pubnub-events-and-actions/references/action-targets.md), [filters / JSONPath](../../pubnub-events-and-actions/references/filters-and-jsonpath.md). Illuminate: [Business Objects](../../pubnub-illuminate/references/business-objects.md), [Metrics](../../pubnub-illuminate/references/metrics.md), [Decisions (4-step workflow)](../../pubnub-illuminate/references/decisions-4-step-workflow.md), [Queries](../../pubnub-illuminate/references/queries-adhoc-vs-saved.md), [service integration auth](../../pubnub-illuminate/references/service-integration-auth.md). Chat: [Chat SDK setup](../../pubnub-chat/references/chat-setup.md), [message actions / reactions](../../pubnub-chat/references/message-actions.md), [file sharing / `sendFile`](../../pubnub-chat/references/file-sharing.md), [threading](../../pubnub-chat/references/threading.md). Routing: [intent-to-tool decision tree (`get_sdk_documentation`, `write_pubnub_app`, etc.)](../../pubnub-choose-docs-path/references/intent-to-tool.md).


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

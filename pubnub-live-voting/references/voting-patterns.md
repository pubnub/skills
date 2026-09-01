# PubNub Voting Patterns

This reference covers advanced voting patterns including result broadcasting, multi-round elimination voting, weighted voting, anonymous vs identified voting, poll templates, audience response systems, and push notifications.

## Result Broadcasting and Live Visualization

### Client-Side Result Subscription

```javascript
pubnub.subscribe({ channels: [`poll.${pollId}.results`] });

pubnub.addListener({
  message: (event) => {
    if (event.message.type === 'tally_update') {
      renderBarChart(event.message.counts, event.message.totalVotes);
    }
    if (event.message.type === 'final_results') {
      renderFinalResults(event.message);
    }
  }
});
```

### Result Aggregation for Visualization

```javascript
function aggregateResults(counts, totalVotes) {
  const results = Object.entries(counts).map(([optionId, count]) => ({
    optionId,
    count,
    percentage: totalVotes > 0 ? ((count / totalVotes) * 100).toFixed(1) : '0.0'
  }));
  results.sort((a, b) => b.count - a.count);
  results.forEach((r, i) => { r.rank = i + 1; });
  return results;
}
```

### Delayed Result Display

Some polls should not show results until the voter has cast their vote. Buffer incoming tally messages and render them after vote submission.

```javascript
let hasVoted = false, pendingResults = null;

pubnub.addListener({
  message: (event) => {
    if (event.channel.endsWith('.results')) {
      if (hasVoted) renderBarChart(event.message.counts, event.message.totalVotes);
      else pendingResults = event.message;
    }
  }
});

function onVoteSubmitted() {
  hasVoted = true;
  if (pendingResults) { renderBarChart(pendingResults.counts, pendingResults.totalVotes); pendingResults = null; }
}
```

## Multi-Round and Elimination Voting

Multi-round voting runs sequential polls where the lowest-scoring option is eliminated after each round.

### Round Management

| Field | Type | Description |
|-------|------|-------------|
| `roundNumber` | Integer | Current round (1-indexed) |
| `totalRounds` | Integer or `null` | Fixed count or dynamic (until one remains) |
| `eliminatedOptions` | Array | Option IDs removed in previous rounds |
| `activeOptions` | Array | Option IDs still in the running |
| `strategy` | String | `eliminate-lowest`, `top-two-runoff`, `instant-runoff` |

### Elimination Voting Flow

```javascript
async function runEliminationRound(pubnub, kvstore, pollId, round) {
  const activeOptions = JSON.parse(await kvstore.get(`poll:${pollId}:round:${round}:options`));
  const tallies = [];
  for (const optId of activeOptions) {
    const count = await kvstore.getCounter(`poll:${pollId}:round:${round}:tally:${optId}`);
    tallies.push({ optionId: optId, count: count || 0 });
  }
  tallies.sort((a, b) => a.count - b.count);
  const eliminated = tallies[0];

  await pubnub.publish({
    channel: `poll.${pollId}.admin`,
    message: { action: 'round_completed', round, eliminated: eliminated.optionId, tallies }
  });

  const remaining = activeOptions.filter(id => id !== eliminated.optionId);
  if (remaining.length <= 1) {
    return pubnub.publish({ channel: `poll.${pollId}.results`,
      message: { type: 'final_results', winner: remaining[0], rounds: round } });
  }

  const next = round + 1;
  await kvstore.set(`poll:${pollId}:round:${next}:options`, JSON.stringify(remaining));
  for (const optId of remaining) await kvstore.setCounter(`poll:${pollId}:round:${next}:tally:${optId}`, 0);
  await pubnub.publish({ channel: `poll.${pollId}.admin`,
    message: { action: 'round_opened', round: next, activeOptions: remaining } });
}
```

## Weighted Voting

Weighted voting assigns different voting power to different participants. Common in shareholder votes, board decisions, and tiered membership systems.

| Weight Source | Example | Implementation |
|---------------|---------|----------------|
| Fixed per role | Admin=3, Member=1 | Lookup from user metadata |
| Share-based | Proportional to ownership | Stored in backend, passed via auth |
| Earned | Points from participation | Queried at vote time |
| Equal | Everyone = 1 | Default behavior |

### Server-Side Weighted Vote Processing

```javascript
function processWeightedVote(kvstore, pollId, voterId, optionId) {
  const weightKey = `poll:${pollId}:weight:${voterId}`;

  return kvstore.get(weightKey).then((weightStr) => {
    const weight = parseInt(weightStr, 10) || 1;
    const voterKey = `poll:${pollId}:voter:${voterId}`;
    return kvstore.get(voterKey).then((existing) => {
      if (existing) throw new Error('DUPLICATE_VOTE');
      return kvstore.set(voterKey, optionId);
    }).then(() => kvstore.incrCounter(`poll:${pollId}:tally:${optionId}`, weight))
      .then(() => kvstore.incrCounter(`poll:${pollId}:total`, weight));
  });
}

// Admin pre-loads voter weights before the poll opens
async function assignVoterWeights(kvstore, pollId, voterWeights) {
  for (const vw of voterWeights) {
    await kvstore.set(`poll:${pollId}:weight:${vw.voterId}`, String(vw.weight));
  }
}
```

## Anonymous vs Identified Voting

| Aspect | Anonymous | Identified |
|--------|-----------|------------|
| Voter ID stored | Hashed or session-only | Real user ID |
| Duplicate prevention | Session fingerprint | User ID in KV Store |
| Auditability | Limited | Full |
| Voter privacy | High | Low |
| Vote change support | Difficult | Easy |
| Use case | Sensitive surveys | Elections, feedback |

### Anonymous Voting Implementation

```javascript
function processAnonymousVote(kvstore, pollId, voterId, optionId) {
  const crypto = require('crypto');
  const hashedId = crypto.createHash('sha256')
    .update(`${pollId}:${voterId}:anonymous-salt-value`)
    .digest('hex').substring(0, 32);

  const voterKey = `poll:${pollId}:anon:${hashedId}`;
  return kvstore.get(voterKey).then((existing) => {
    if (existing) throw new Error('DUPLICATE_VOTE');
    return kvstore.set(voterKey, 'voted');
  }).then(() => kvstore.incrCounter(`poll:${pollId}:tally:${optionId}`, 1))
    .then(() => kvstore.incrCounter(`poll:${pollId}:total`, 1));
}
```

### Identified Voting with Audit Trail

```javascript
function processIdentifiedVote(kvstore, pubnub, pollId, voterId, optionId) {
  const voterKey = `poll:${pollId}:voter:${voterId}`;
  return kvstore.get(voterKey).then((existing) => {
    if (existing) throw new Error('DUPLICATE_VOTE');
    return kvstore.set(voterKey, optionId);
  }).then(() => pubnub.publish({
    channel: `poll.${pollId}.audit`,
    message: { event: 'vote_recorded', voterId, optionId, timestamp: Date.now() }
  })).then(() => kvstore.incrCounter(`poll:${pollId}:tally:${optionId}`, 1))
    .then(() => kvstore.incrCounter(`poll:${pollId}:total`, 1));
}
```

## Poll Templates and Reuse

Pre-defined templates accelerate poll creation for common use cases.

| Template | Type | Options | Live Results |
|----------|------|---------|-------------|
| `yes-no` | single-choice | Yes, No | Yes |
| `satisfaction-5` | single-choice | Very Satisfied through Very Dissatisfied | After close |
| `nps` | rating | 0-10 scale | After close |
| `emoji-reaction` | single-choice | thumbs-up, heart, laugh, surprised, sad | Yes |

```javascript
function createPollFromTemplate(templateId, question, overrides = {}) {
  const template = POLL_TEMPLATES[templateId];
  if (!template) throw new Error(`Unknown template: ${templateId}`);
  return {
    pollId: `poll-${Date.now()}`, question, ...template, ...overrides,
    settings: { ...template.settings, ...(overrides.settings || {}) },
    createdAt: Date.now()
  };
}
```

## Audience Response Systems

Audience response systems (ARS) are designed for live events where a presenter displays questions and the audience votes in real time via mobile devices.

### Live Event Flow

1. Presenter creates a poll from the admin dashboard
2. Question is displayed on the main screen
3. Audience members scan a QR code or open a link to join
4. Votes stream in and results animate on the main screen
5. Presenter closes the poll and shows the final result

### Presenter Display Client

```javascript
function initPresenterDisplay(pubnub, pollId) {
  pubnub.subscribe({ channels: [`poll.${pollId}.results`, `poll.${pollId}.admin`] });
  pubnub.addListener({
    message: (event) => {
      if (event.channel.endsWith('.results')) animateBarChart(event.message.counts, event.message.totalVotes);
      if (event.channel.endsWith('.admin')) handleAdminEvent(event.message);
    }
  });
}

function animateBarChart(counts, total) {
  Object.entries(counts).forEach(([optionId, count]) => {
    const pct = total > 0 ? (count / total) * 100 : 0;
    const bar = document.getElementById(`bar-${optionId}`);
    bar.style.width = `${pct}%`;
    bar.querySelector('.pct').textContent = `${pct.toFixed(1)}%`;
  });
}
```

### Mobile Audience Client (Swift)

```swift
import PubNub

let config = PubNubConfiguration(
    publishKey: "pub-c-...",
    subscribeKey: "sub-c-...",
    userId: "audience-\(UUID().uuidString)"
)
let pubnub = PubNub(configuration: config)

func submitVote(pollId: String, optionId: String) {
    let vote: [String: Any] = [
        "type": "vote", "pollId": pollId, "optionId": optionId,
        "voterId": config.userId,
        "timestamp": Date().timeIntervalSince1970 * 1000
    ]
    pubnub.publish(channel: "poll.\(pollId).votes", message: vote) { result in
        switch result {
        case .success: print("Vote submitted")
        case .failure(let error): print("Vote failed: \(error)")
        }
    }
}
```

### Mobile Audience Client (Kotlin)

```kotlin
val config = PNConfiguration(UserId("audience-${UUID.randomUUID()}")).apply {
    publishKey = "pub-c-..."
    subscribeKey = "sub-c-..."
}
val pubnub = PubNub.create(config)

fun submitVote(pollId: String, optionId: String) {
    val vote = mapOf("type" to "vote", "pollId" to pollId,
        "optionId" to optionId, "voterId" to config.userId.value)
    pubnub.publish(channel = "poll.$pollId.votes", message = vote).async { result ->
        result.onSuccess { println("Vote submitted") }
        result.onFailure { println("Vote failed: ${it.message}") }
    }
}
```

## Push Notification Patterns for Vote Reminders

```javascript
// Register device for push notifications
await pubnub.push.addChannels({
  channels: ['polls.notifications'], device: deviceToken,
  pushGateway: 'apns2', environment: 'production', topic: 'com.yourapp.voting'
});

// Send reminder with push payloads for iOS and Android
await pubnub.publish({
  channel: 'polls.notifications',
  message: { text: 'Reminder: voting closes in 5 minutes!' },
  pn_apns: { aps: { alert: { title: 'Vote Now!', body: 'Poll closes in 5 min.' }, sound: 'default' } },
  pn_gcm: { notification: { title: 'Vote Now!', body: 'Poll closes in 5 min.' }, data: { pollId: 'poll-2024-finale' } }
});
```

## Voting Type Comparison

| Feature | Single Choice | Multiple Choice | Ranked Choice | Rating | Emoji Reaction |
|---------|--------------|-----------------|---------------|--------|----------------|
| Selections per voter | 1 | Configurable (1-N) | All ranked | 1 score/option | 1 |
| Tally method | Counter | Counter per option | IRV algorithm | Running average | Counter |
| Result display | Bar chart | Bar chart | Round table | Average score | Icon counts |
| Server complexity | Low | Low | High | Medium | Low |
| Vote change support | Easy | Easy | Difficult | Easy | Easy |
| Ideal audience | Any | Any | Small-Medium | Any | Large (live) |

## Error Handling Patterns

```javascript
async function submitVoteWithRetry(pubnub, pollId, optionId, voterId, retries = 2) {
  for (let i = 0; i <= retries; i++) {
    try {
      const res = await pubnub.publish({
        channel: `poll.${pollId}.votes`,
        message: { type: 'vote', pollId, optionId, voterId, timestamp: Date.now() }
      });
      return { success: true, timetoken: res.timetoken };
    } catch (err) {
      if (err.status?.statusCode === 403) return { success: false, error: 'ACCESS_DENIED' };
      if (i === retries) return { success: false, error: 'NETWORK_ERROR' };
      await new Promise(r => setTimeout(r, Math.pow(2, i) * 500));
    }
  }
}
```

## Best Practices

- **Use templates for common poll types**: Pre-define yes/no, satisfaction, NPS, and emoji reaction templates to reduce setup time and ensure consistency.
- **Animate result transitions**: Smoothly animate bar chart width changes on tally updates. Interpolate between old and new values to avoid jarring jumps.
- **Buffer results for late-display polls**: When results should only show after voting, buffer incoming tally messages and render them after the user submits their vote.
- **Manage round state centrally**: In multi-round voting, publish round transitions on the admin channel and have all clients derive UI state from those messages.
- **Pre-assign weights before opening the poll**: For weighted voting, load all voter weights into KV Store before the poll opens. Never let clients specify their own weight.
- **Hash voter IDs for anonymous polls**: Use a one-way hash with a poll-specific salt so anonymous votes cannot be traced back to users but duplicates are still prevented.
- **Design for mobile-first in audience response**: Most live-event voters use mobile devices. Optimize the voting UI for touch input and minimal data transfer.
- **Send push reminders sparingly**: Limit push notifications to one reminder per poll to avoid causing users to disable notifications.
- **Test multi-round flows end-to-end**: Elimination voting has more state transitions than single-round polls. Simulate full sequences including tie-breaking.

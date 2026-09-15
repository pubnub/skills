# PubNub Voting Patterns

Result subscribe, weighted Function delta, anonymous vs identified channel keys, and ARS presenter/audience channels. Voting-type / election-theory comparison tables are out of scope.

## Result broadcasting

```javascript
pubnub.subscribe({ channels: [`poll.${pollId}.results`] });

pubnub.addListener({
  message: (event) => {
    if (event.message.type === 'tally_update') {
      renderResults(event.message.counts, event.message.totalVotes);
    }
    if (event.message.type === 'final_results') {
      renderFinalResults(event.message);
    }
  }
});
```

If results should appear only after the voter submits, buffer `.results` messages until vote publish succeeds, then flush.

## Multi-round (PubNub)

Round state in KV (`poll:{pollId}:round:{n}:options` / `:tally:{optId}`). Publish `round_completed` / `round_opened` on `poll.{pollId}.admin`; `final_results` on `poll.{pollId}.results`. Elimination policy is operator-defined — do not copy IRV/runoff theory here.

```javascript
await pubnub.publish({
  channel: `poll.${pollId}.admin`,
  message: { action: 'round_completed', round, eliminated: eliminated.optionId, tallies }
});
await pubnub.publish({
  channel: `poll.${pollId}.results`,
  message: { type: 'final_results', winner: remaining[0], rounds: round }
});
```

## Weighted votes (Function delta)

After duplicate-voter check, `incrCounter(..., weight)` on option and total. Pre-load `poll:{pollId}:weight:{voterId}` before the poll opens. Never let clients supply their own weight. Same [Pattern 1](../../pubnub-functions/references/functions-patterns.md#pattern-1-distributed-counter) pipeline as [voting-tallying.md](voting-tallying.md).

## Anonymous vs identified (channel / KV split)

- **Identified:** KV `poll:{pollId}:voter:{voterId}`; after accept, publish audit to `poll.{pollId}.audit`.
- **Anonymous:** hash voter id with poll-specific salt; KV `poll:{pollId}:anon:{hashedId}`. Same Before-Publish counter pattern, different key prefix.

## Audience response (presenter / audience)

| Role | Subscribe | Publish |
|------|-----------|---------|
| Presenter | `poll.{id}.results`, `poll.{id}.admin` | admin lifecycle |
| Audience | `poll.{id}.admin`, `poll.{id}.meta` | `poll.{id}.votes` |

```javascript
function initPresenterDisplay(pubnub, pollId) {
  pubnub.subscribe({ channels: [`poll.${pollId}.results`, `poll.${pollId}.admin`] });
}

async function submitAudienceVote(pubnub, pollId, optionId, voterId) {
  await pubnub.publish({
    channel: `poll.${pollId}.votes`,
    message: { type: 'vote', pollId, optionId, voterId, timestamp: Date.now() }
  });
}
```

## Vote reminders (push)

Register `polls.notifications` with `pubnub.push.addChannels`. Publish one reminder with `pn_apns` / `pn_gcm` — retrieve payload shape from MCP / docs.

## Retry on vote publish

Retry transient publish failures; treat `403` as `ACCESS_DENIED` (AM). Dedup still happens in Before-Publish.

## Best Practices

- **Results on `.results` only** — clients never read raw `.votes`.
- **Admin channel** owns round and lifecycle transitions.
- **Weights in KV before open.**
- **Hash keys for anonymous polls**; audit channel for identified.
- **Spare push reminders** (one per poll).

<!-- canonical-for: DEMO_KEYS -->
<!-- used-by: pubnub-choose-docs-path -->

# Demo Keys

The canonical reference for PubNub demo keys: when they are acceptable, when they are dangerous.

## What Demo Keys Are

PubNub publishes a public set of demo publish/subscribe keys used in:

- Sample applications and code snippets on the PubNub website
- The PubNub Debug Console
- Interactive online demonstrations
- Blog post copy-paste examples

These keys are **publicly documented** — anyone can use them, and anyone subscribed to the same channel with demo keys will see your messages.

## What Demo Keys Are Not Suitable For

| Use case | Acceptable? | Why |
|---|---|---|
| Copy-paste of a sample from PubNub docs | Yes | Intended use |
| PubNub Debug Console quick test | Yes | Intended use |
| A throwaway script you delete in 5 minutes | Tolerable | But create a free keyset instead |
| Building any application of your own | **No** | No isolation, public traffic |
| Development of your own app | **No** | Use your own dev keyset |
| Staging or pre-production | **No** | Use a staging keyset |
| Anything in production | **No, ever** | No SLA, no security, throttled |
| Anything handling private or PII data | **No, ever** | Public; anyone can subscribe |

## Why Demo Keys Are Dangerous Outside Their Intended Use

1. **No isolation.** Any other developer testing the same channel name sees your messages. There is no namespacing.
2. **Throttled.** Demo keys are aggressively rate-limited to keep them available for everyone. Your app may experience unexpected disconnects or dropped messages under any meaningful load.
3. **No add-on guarantees.** Premium features ([Access Manager](../../pubnub-security/references/access-manager.md), Persistence, Functions) may not behave as documented on demo keys.
4. **No security.** Anyone can publish to or subscribe to any channel on the demo keyset.
5. **No SLA.** PubNub does not guarantee uptime or behavior on demo keys.
6. **No support.** Issues encountered using demo keys will not receive priority support.

## What to Use Instead

Always create a free PubNub account and use your own keyset:

1. Sign up at [pubnub.com](https://www.pubnub.com/).
2. Open the Admin Portal → Apps → create app → keyset auto-created.
3. Configure add-ons per environment ([keysets-and-environments.md](keysets-and-environments.md)).
4. Store keys in a secrets manager or env vars ([key-rotation-and-hygiene.md](key-rotation-and-hygiene.md)).

The free tier is generous and is appropriate for development and small-scale staging traffic.

## Detecting Demo Keys in Code

If you suspect a sample or onboarding flow has demo keys baked in, scan for the published demo key prefixes that PubNub uses in their public examples. Replace them with env-var lookups before going beyond a copy-paste exercise.

```bash
git grep -E '(pub|sub)-c-[a-f0-9-]+' -- ':!docs/' ':!*.md'
```

Any match in source code is a flag for review — verify it is your own key, not a documentation demo key.

## Related Reading

- [keysets-and-environments.md](keysets-and-environments.md) — creating your own keysets
- [key-rotation-and-hygiene.md](key-rotation-and-hygiene.md) — keeping your own keys safe

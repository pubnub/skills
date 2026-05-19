<!-- canonical-for: DOCS_ROUTING -->
<!-- used-by: -->

# Intent to Tool Mapping

The canonical decision tree from a user's stated intent to (specialist skill, MCP tool on `user-pubnub`, PubNub docs surface). Every other skill that mentions docs routing or MCP tool selection must link here.

## How to Use This Reference

1. Read the user's question and pick the closest **Intent** row below.
2. Hand off using the exact (Skill, MCP tool, Docs) triple in that row.
3. If two intents apply, ask the user which to address first — don't guess.

## Top-Level Decision Tree

```text
Is the user asking about chat / messaging features (DMs, group chat, typing, reactions, threads)?
├── Yes → see "Chat" section below
└── No  → Is the user asking about analytics / KPIs / threshold-triggered actions?
         ├── Yes → see "Analytics & Automation" section below
         └── No  → Is the user setting up the environment (apps, keysets, demo keys)?
                  ├── Yes → see "Environment & Setup" section below
                  └── No  → see "Real-time core (non-chat)" or "Cross-cutting" sections
```

## Chat

| Intent | Skill | MCP tool | Docs surface |
|---|---|---|---|
| Build 1:1 DM, group chat, channels, message reactions, typing indicators | `pubnub-chat` | `get_chat_sdk_documentation` | Chat SDK docs |
| Add Message Actions to a non-chat app | `pubnub-chat` (references/message-actions.md) | `get_chat_sdk_documentation` | Chat SDK docs |
| File Sharing in chat | `pubnub-chat` (references/file-sharing.md) | `get_chat_sdk_documentation` | Chat SDK docs |
| Threading / threaded replies | `pubnub-chat` (references/threading.md) | `get_chat_sdk_documentation` | Chat SDK docs |
| Chat room notifications | `pubnub-chat` | `get_chat_sdk_documentation` | Chat SDK docs |

## Real-time Core (non-chat)

| Intent | Skill | MCP tool | Docs surface |
|---|---|---|---|
| IoT telemetry, device-to-cloud streaming | `pubnub-app-developer` | `get_sdk_documentation` | Core SDK docs |
| Game state sync, multiplayer | `pubnub-multiplayer-gaming` | `get_sdk_documentation` | Core SDK docs |
| Live sport scores, play-by-play | `pubnub-live-sport-updates` | `get_sdk_documentation` | Core SDK docs |
| Live stock quotes, market data | `pubnub-live-stock-quote-updates` | `get_sdk_documentation` | Core SDK docs |
| Live auctions with bidding | `pubnub-live-auctions` | `get_sdk_documentation` | Core SDK docs |
| Live betting / casino | `pubnub-live-betting-casino` | `get_sdk_documentation` | Core SDK docs |
| Live voting / polling | `pubnub-live-voting` | `get_sdk_documentation` | Core SDK docs |
| Order tracking / delivery driver | `pubnub-order-delivery-driver` | `get_sdk_documentation` | Core SDK docs |
| Telemedicine / virtual waiting room | `pubnub-telemedicine` | `get_sdk_documentation` | Core SDK docs |
| Push notifications, fan-out broadcasts | `pubnub-app-developer` | `get_sdk_documentation` | Core SDK docs |

## Presence / Connection State

| Intent | Skill | MCP tool | Docs surface |
|---|---|---|---|
| Show online/offline indicators, room occupancy | `pubnub-presence` | `get_pubnub_presence` (runtime) + `get_sdk_documentation` | Presence docs |
| Detect dropped connections | `pubnub-presence` | `get_sdk_documentation` | Presence docs |
| Multi-device sync for the same user | `pubnub-presence` | `get_sdk_documentation` | Presence docs |
| Build a reconnect strategy | `pubnub-reliability` | `get_sdk_documentation` | Best-practices docs |

## History / Replay

| Intent | Skill | MCP tool | Docs surface |
|---|---|---|---|
| Catch up an offline user, paginate by timetoken | `pubnub-history` | `get_pubnub_messages` + `get_sdk_documentation` | History/Storage docs |
| Replay a specific channel or time range | `pubnub-history` | `get_pubnub_messages` | History/Storage docs |
| Prevent old messages on subscribe | `pubnub-history` | `get_sdk_documentation` | History/Storage docs |

## App Context (Users / Channels / Memberships)

| Intent | Skill | MCP tool | Docs surface |
|---|---|---|---|
| Store user profile, avatar, role | `pubnub-app-context` | `manage_app_context` | Objects/App Context docs |
| Manage channel metadata (name, topic, mute flags) | `pubnub-app-context` | `manage_app_context` | Objects/App Context docs |
| Track user membership in channels | `pubnub-app-context` | `manage_app_context` | Objects/App Context docs |

## Edge Logic & Integrations

| Intent | Skill | MCP tool | Docs surface |
|---|---|---|---|
| Per-message transform, validate, enrich, moderate at the edge | `pubnub-functions` | `get_sdk_documentation` (Functions section) | Functions 2.0 docs |
| HTTP endpoint backed by PubNub | `pubnub-functions` | `get_sdk_documentation` | Functions 2.0 docs |
| Scheduled task running every N minutes | `pubnub-functions` | `get_sdk_documentation` | Functions 2.0 docs |
| Forward every message to webhook / Lambda / Kafka / SQS / EventBridge | `pubnub-events-and-actions` | `get_sdk_documentation` | Events & Actions docs |
| Trigger external workflow from a PubNub event | `pubnub-events-and-actions` | `get_sdk_documentation` | Events & Actions docs |

## Analytics & Automation

| Intent | Skill | MCP tool | Docs surface |
|---|---|---|---|
| Track real-time KPIs (message counts, top users, session durations) | `pubnub-illuminate` | `manage_illuminate` | Illuminate docs |
| Threshold-triggered actions (alert when X exceeds Y) | `pubnub-illuminate` | `manage_illuminate` | Illuminate docs |
| Anomaly detection (cross-posting spam, chat flooding) | `pubnub-illuminate` | `manage_illuminate` | Illuminate docs |
| Real-time dashboards with chart overlays | `pubnub-illuminate` | `manage_illuminate` | Illuminate docs |
| Auto-mute / auto-publish / auto-update App Context based on metric breach | `pubnub-illuminate` | `manage_illuminate` | Illuminate docs |

## Environment & Setup

| Intent | Skill | MCP tool | Docs surface |
|---|---|---|---|
| Create/inspect/update an app | `pubnub-keyset-management` | `manage_apps` | Admin Portal docs |
| Create/inspect/update a keyset, separate dev/staging/prod | `pubnub-keyset-management` | `manage_keysets` | Admin Portal docs |
| Demo key questions | `pubnub-keyset-management` | n/a | Admin Portal docs |
| Custom origin / custom domain | `pubnub-keyset-management` | `manage_keysets` | Admin Portal docs |
| Rotate publish/subscribe/secret keys | `pubnub-keyset-management` | `manage_keysets` | Admin Portal docs |

## Security

| Intent | Skill | MCP tool | Docs surface |
|---|---|---|---|
| Authenticate clients, issue tokens, restrict channels | `pubnub-security` | `get_sdk_documentation` | Access Manager docs |
| Encrypt messages or files | `pubnub-security` | `get_sdk_documentation` | Encryption docs |
| Configure TLS | `pubnub-security` | `get_sdk_documentation` | Security docs |
| IP whitelist / IP allowlist | `pubnub-security` | n/a (Admin Portal) | Security docs |
| DoS / DDoS mitigation | `pubnub-security` | n/a (Admin Portal) | Security docs |
| Compliance reports (SOC 2, HIPAA, etc.) | `pubnub-security` | n/a (Admin Portal) | Compliance docs |

## Scale & Performance

| Intent | Skill | MCP tool | Docs surface |
|---|---|---|---|
| Channel groups, wildcard subscribe | `pubnub-scale` | `get_sdk_documentation` | Stream Controller docs |
| Throughput tuning, payload sizing | `pubnub-scale` + `pubnub-observability` | `get_sdk_documentation` | Best-practices docs |
| Plan for 10k+ concurrent users / live event | `pubnub-scale` | `write_pubnub_app` | Best-practices docs |
| Optimize mobile battery usage | `pubnub-scale` | `get_sdk_documentation` | Mobile docs |

## Cross-cutting Disciplines

| Intent | Skill | MCP tool | Docs surface |
|---|---|---|---|
| Reliability patterns (backoff, idempotency, retry, dedup) | `pubnub-reliability` | `write_pubnub_app` | Best-practices docs |
| Logging correlation, test pyramid, runbook | `pubnub-observability` | `write_pubnub_app` | Best-practices docs |
| Architecture review / production-readiness | `pubnub-reliability` + `pubnub-observability` + `pubnub-security` | `write_pubnub_app` | Best-practices docs |

## Runtime Testing & Incident Triage

| Intent | Skill | MCP tool | Docs surface |
|---|---|---|---|
| Send a test message to validate publish | `pubnub-app-developer` | `send_pubnub_message` | n/a |
| Subscribe and observe live messages | `pubnub-app-developer` | `subscribe_and_receive_pubnub_messages` | n/a |
| Fetch historical messages for debugging | `pubnub-history` | `get_pubnub_messages` | n/a |
| Inspect occupancy / who is in a channel | `pubnub-presence` | `get_pubnub_presence` | n/a |
| Triage an incident (errors, dropouts, latency) | `pubnub-observability` | `get_pubnub_messages` + `get_pubnub_presence` | Best-practices docs |

## Conceptual / Step-by-Step Recipes

When the user asks "how do I X" with a step-by-step expectation and X spans multiple PubNub products, prefer the `how_to` MCP tool first to retrieve the relevant recipe, then hand off to the specialist skill that owns the implementation.

## Disambiguation Heuristics

When two intents look similar, use these tie-breakers:

- **"Send a message every time X happens" + external destination** → `pubnub-events-and-actions` (not `pubnub-functions`).
- **"Transform / validate / enrich the message inline"** → `pubnub-functions` (not `pubnub-events-and-actions`).
- **"Track a KPI and alert when it crosses a threshold"** → `pubnub-illuminate` (not `pubnub-functions` + custom logic).
- **"Online / offline indicator"** → `pubnub-presence`. **"Reconnect after dropout"** → `pubnub-reliability`. **"Was this user in the channel 10 min ago"** → `pubnub-history`.
- **"User profile, avatar, role"** → `pubnub-app-context` (not `pubnub-chat`, even in a chat app).
- **"Where do I put my keys?"** → `pubnub-keyset-management` for env separation; `pubnub-security` for the rule that secret keys never leave the server.

## Handoff Template

Always produce a handoff in this exact shape (copied here from `SKILL.md` for self-contained reference):

```text
Intent      : <one sentence restating what the user is trying to do>
Skill       : <specialist-skill-name>
MCP tool    : <user-pubnub MCP tool>
Docs source : <which PubNub docs surface to consult>
Next step   : <one concrete action the user or agent should take next>
```

## Canonical Owner Files

Every owned topic referenced in the tables above maps to a canonical file. When handing off, link to one of these:

### Environment & Setup
- Apps, keysets, environment separation, secret-key handling → [pubnub-keyset-management/references/keysets-and-environments.md](../../pubnub-keyset-management/references/keysets-and-environments.md)
- Demo keys → [pubnub-keyset-management/references/demo-keys.md](../../pubnub-keyset-management/references/demo-keys.md)
- Custom origin / custom domain → [pubnub-keyset-management/references/custom-origin.md](../../pubnub-keyset-management/references/custom-origin.md)

### Security
- Access Manager and PAM tokens → [pubnub-security/references/access-manager.md](../../pubnub-security/references/access-manager.md)
- IP whitelist / IP allowlist → [pubnub-security/references/ip-whitelisting.md](../../pubnub-security/references/ip-whitelisting.md)
- DoS / DDoS mitigation → [pubnub-security/references/dos-mitigation.md](../../pubnub-security/references/dos-mitigation.md)
- Compliance reports (SOC 2, HIPAA) → [pubnub-security/references/compliance-reports.md](../../pubnub-security/references/compliance-reports.md)

### Presence
- Multi-device sync → [pubnub-presence/references/multi-device-sync.md](../../pubnub-presence/references/multi-device-sync.md)
- Dropped connection detection → [pubnub-presence/references/dropped-connections.md](../../pubnub-presence/references/dropped-connections.md)

### Reliability
- Reconnect strategy (backoff + jitter) → [pubnub-reliability/references/backoff-and-jitter.md](../../pubnub-reliability/references/backoff-and-jitter.md)

### History
- Timetoken pagination → [pubnub-history/references/pagination-and-ordering.md](../../pubnub-history/references/pagination-and-ordering.md)
- Offline catch-up → [pubnub-history/references/offline-catch-up.md](../../pubnub-history/references/offline-catch-up.md)

### App Context
- Users, channel metadata, membership metadata → [pubnub-app-context/references/users.md](../../pubnub-app-context/references/users.md)

### Chat
- Message Actions and reactions → [pubnub-chat/references/message-actions.md](../../pubnub-chat/references/message-actions.md)
- File Sharing → [pubnub-chat/references/file-sharing.md](../../pubnub-chat/references/file-sharing.md)

### Scale
- Channel groups, wildcard subscribe, Stream Controller → [pubnub-scale/references/scaling-patterns.md](../../pubnub-scale/references/scaling-patterns.md)

### Observability
- Payload sizing and cost hygiene → [pubnub-observability/references/cost-and-payload-hygiene.md](../../pubnub-observability/references/cost-and-payload-hygiene.md)
- Incident triage and runbook → [pubnub-observability/references/incident-runbook.md](../../pubnub-observability/references/incident-runbook.md)

# PubNub Skills Collection

A comprehensive set of skills for building real-time applications with PubNub's platform. Each skill provides focused guidance, code examples, and best practices for specific PubNub capabilities.

## Quick Start

### Tessl Install PubNub Skills

```bash
npx tessl skill search pubnub
```
### Vercel Skills Install PubNub Skills

The skills CLI requires no installation and can be run directly using npx:

```bash
npx skills add pubnub/skills
```

## Available Skills

| Skill | Description |
|-------|-------------|
| [pubnub-app-developer](./pubnub-app-developer) | Core pub/sub messaging, channels, SDK patterns |
| [pubnub-presence](./pubnub-presence) | Real-time user online/offline status, occupancy |
| [pubnub-functions](./pubnub-functions) | Serverless edge functions, webhooks, KVStore |
| [pubnub-security](./pubnub-security) | Access Manager, AES-256 encryption, TLS |
| [pubnub-chat](./pubnub-chat) | Chat SDK, messaging, typing indicators, reactions |
| [pubnub-scale](./pubnub-scale) | High-volume optimization, channel groups, persistence |
| [pubnub-choose-docs-path](./pubnub-choose-docs-path) | Router skill: chat-vs-non-chat, MCP tool selection |
| [pubnub-keyset-management](./pubnub-keyset-management) | Apps, keysets, env separation, key hygiene, demo keys |
| [pubnub-app-context](./pubnub-app-context) | Objects API: users, channels, memberships, metadata |
| [pubnub-illuminate](./pubnub-illuminate) | Business Objects, Metrics, Decisions, Queries, Dashboards |
| [pubnub-history](./pubnub-history) | Storage & Playback, timetoken pagination, offline catch-up, dedup-on-merge |
| [pubnub-reliability](./pubnub-reliability) | Backoff+jitter, idempotent publish, queue+retry, dedup, schema versioning |
| [pubnub-observability](./pubnub-observability) | Logging correlation, test pyramid, runbooks, payload hygiene, usage metrics |
| [pubnub-events-and-actions](./pubnub-events-and-actions) | Webhook / Lambda / Kafka / SQS / EventBridge integrations |
| [pubnub-live-auctions](./pubnub-live-auctions) | Real-time auction platforms with bidding and countdowns |
| [pubnub-live-betting-casino](./pubnub-live-betting-casino) | Real-time betting and casino game platforms |
| [pubnub-live-sport-updates](./pubnub-live-sport-updates) | Real-time sports scores, play-by-play, scoreboards |
| [pubnub-live-stock-quote-updates](./pubnub-live-stock-quote-updates) | Real-time stock quotes and market data |
| [pubnub-live-voting](./pubnub-live-voting) | Real-time voting and polling systems |
| [pubnub-multiplayer-gaming](./pubnub-multiplayer-gaming) | Real-time multiplayer game state synchronization |
| [pubnub-order-delivery-driver](./pubnub-order-delivery-driver) | Real-time order tracking and delivery driver systems |
| [pubnub-telemedicine](./pubnub-telemedicine) | HIPAA-compliant telemedicine applications |

For more details, see the [CLI documentation](https://skills.sh/docs/cli).

### Use in Claude Code

Once installed, skills are automatically available. Ask Claude about PubNub topics and the relevant skill will be invoked.

## Skill Details

### pubnub-app-developer

Build real-time applications with PubNub pub/sub messaging.

**Use for:**
- Implementing channels and subscriptions
- Message handling and SDK configuration
- Cross-platform development (JavaScript, Python, Swift, Kotlin)

**References:**
- `publish-subscribe.md` - Core pub/sub patterns
- `channels.md` - Channel naming and design
- `sdk-patterns.md` - SDK initialization across platforms

---

### pubnub-presence

Implement real-time presence tracking with PubNub.

**Use for:**
- User online/offline status indicators
- Channel occupancy counts
- Connection state management
- Multi-device synchronization

**References:**
- `presence-setup.md` - Admin Portal configuration
- `presence-events.md` - Join/leave/timeout handling
- `presence-patterns.md` - Scalable presence best practices

---

### pubnub-functions

Develop serverless edge functions with PubNub Functions 2.0.

**Use for:**
- Message transformation and enrichment
- Webhook integrations with external APIs
- HTTP endpoints for REST APIs
- Scheduled tasks and aggregation

**References:**
- `functions-basics.md` - Function structure and event types
- `functions-modules.md` - KVStore, XHR, Vault, Crypto modules
- `functions-patterns.md` - Counters, rate limiting, workflows

---

### pubnub-security

Secure PubNub applications with Access Manager and encryption.

**Use for:**
- Authentication and authorization
- Token grants and permission management
- AES-256 message encryption
- TLS configuration

**References:**
- `access-manager.md` - Access Manager setup and grants
- `encryption.md` - Message and file encryption
- `security-best-practices.md` - Key security, compliance

---

### pubnub-chat

Build chat applications with PubNub Chat SDK.

**Use for:**
- Direct messaging and group channels
- Typing indicators and read receipts
- Message reactions and threading
- User management

**References:**
- `chat-setup.md` - Chat SDK initialization
- `chat-features.md` - Channels, messages, reactions
- `chat-patterns.md` - User management, real-time sync

---

### pubnub-scale

Scale PubNub applications for high-volume events.

**Use for:**
- Performance optimization
- Channel groups and wildcard subscriptions
- Large-scale event handling (10K+ users)
- Throughput tuning and concurrency

**References:**
- `scaling-patterns.md` - Channel groups, wildcards
- `performance.md` - Message optimization, batching
- `large-events.md` - 10K+ concurrent live-event playbook + PubNub engagement

> Message persistence, history, and offline catch-up have moved to the dedicated `pubnub-history` skill.

---

### pubnub-choose-docs-path

Route PubNub questions to the correct documentation source, MCP tool, and specialist skill.

**Use for:**
- First-touch triage when a user mentions PubNub without naming a feature
- Choosing between chat SDK vs core SDK
- Picking between Functions, Events & Actions, and Illuminate
- Mapping intent to the right `user-pubnub` MCP tool

**References:**
- `intent-to-tool.md` - Decision tree from user intent to (skill, MCP tool, docs link)

---

### pubnub-keyset-management

Manage PubNub apps, keysets, and environment separation.

**Use for:**
- Setting up dev / staging / prod keysets
- Publish, subscribe, and secret key handling
- Key rotation and hygiene
- Demo-key boundaries and custom origin configuration

**References:**
- `keysets-and-environments.md` - App/keyset model, env separation
- `key-rotation-and-hygiene.md` - Rotation procedure, secret-key handling
- `demo-keys.md` - Demo-key limits and migration
- `custom-origin.md` - Custom origin configuration

---

### pubnub-app-context

Manage users, channels, and memberships with the PubNub App Context (Objects) API.

**Use for:**
- User profile CRUD with custom metadata
- Channel metadata for descriptions, settings, and tags
- Membership management between users and channels
- Server-side filtering and live metadata-change events

**References:**
- `users.md` - User CRUD, profile metadata, filtering
- `channels-and-memberships.md` - Channel metadata, memberships, members
- `metadata-and-filtering.md` - Filter expressions, eTag concurrency, change events

---

### pubnub-illuminate

Build real-time analytics, decisions, and dashboards with PubNub Illuminate.

**Use for:**
- Defining Business Objects from in-flight messages
- Authoring Metrics (COUNT, SUM, AVG) over dimensions
- The 4-step Decision workflow (POST → READ → PUT rules → PUT enable)
- Saved Queries and Dashboards
- Service Integration auth for the admin API

**References:**
- `business-objects.md` - Schema, JSONPath fields, activation
- `metrics.md` - Aggregations, dimensions, windows
- `decisions-4-step-workflow.md` - METRIC and BUSINESSOBJECT decision flows
- `queries-adhoc-vs-saved.md` - Query types and reuse
- `dashboards.md` - Dashboard composition and sharing
- `service-integration-auth.md` - Admin API authentication

---

### pubnub-history

Replay PubNub messages with Storage & Playback, timetoken pagination, and offline catch-up.

**Use for:**
- Paginated history loaders for chat, tickets, and timelines
- Offline app catch-up when a client reconnects
- Multi-channel history retrieval with `fetchMessages`
- Dedup-on-merge between live and historical streams

**References:**
- `pagination-and-ordering.md` - Timetoken pagination, ordering, edge cases
- `offline-catch-up.md` - Reconnect-and-catch-up flow with bounded windows
- `retention-and-storage.md` - Storage tiers, retention, `storeInHistory`

---

### pubnub-reliability

Cross-cutting reliability patterns for PubNub apps.

**Use for:**
- Reconnect with exponential backoff and jitter
- Idempotent publish with client-generated `message_id`
- Dedup-on-merge between live and history streams
- Persistent offline publish queues with dead-letter
- Schema versioning and tolerant readers

**References:**
- `backoff-and-jitter.md` - Reconnect controller, jitter formula
- `idempotent-publish.md` - Stable IDs across retries
- `dedup-on-merge.md` - LRU and Set-based dedup
- `queue-and-retry.md` - Persistent, FIFO, bounded queues
- `schema-versioning.md` - Version routing and migration flow

---

### pubnub-observability

Logging, testing, cost hygiene, incident triage, and usage metrics for PubNub apps.

**Use for:**
- Correlation logging for every publish/subscribe (four-field standard)
- Test pyramid for real-time apps
- Pre-launch cost projection (transactions, not bytes)
- Incident-triage runbook for messages-dropped and connection issues
- Weekly usage-metrics review and billing reconciliation

**References:**
- `logging-correlation.md` - Four-field correlation, structured logs
- `test-pyramid.md` - Unit, integration, load test layers
- `cost-and-payload-hygiene.md` - Fan-out math, payload sizing, coalescing
- `incident-runbook.md` - Triage flows for common incidents
- `usage-metrics.md` - `get_pubnub_usage_metrics`, anomaly detection

---

### pubnub-events-and-actions

Route PubNub events to external systems with no code via Events & Actions (E&A).

**Use for:**
- Webhook receivers (Lambda, Cloud Functions, custom HTTP)
- Fan-out to SQS, Kinesis, Kafka, S3, EventBridge, IFTTT, AMQP
- JSONPath filters on Messages, Users, Channels, Push, Memberships events
- Idempotent batched receivers, retries, and envelope handling

**References:**
- `event-types.md` - Listener categories and event shapes
- `action-targets.md` - Webhook, queue, stream, storage, and IFTTT targets
- `filters-and-jsonpath.md` - Basic vs JSONPath filters, operators
- `retries-and-batching.md` - Retry policy, batching, envelope versions

---

### pubnub-live-auctions

Build real-time auction platforms with PubNub bidding and countdowns.

**Use for:**
- Live bidding with server-side validation
- Countdown timer synchronization
- Outbid notifications and bid activity feeds
- Auction lifecycle management

**References:**
- `auction-setup.md` - Auction channel design, lifecycle management
- `auction-bidding.md` - Bid validation, race condition handling
- `auction-patterns.md` - Reserve prices, auto-extend, proxy bidding

---

### pubnub-live-betting-casino

Build real-time betting and casino game platforms with PubNub.

**Use for:**
- Live/in-play betting with real-time odds updates
- Casino game state synchronization
- Wager placement, validation, and settlement
- Responsible gambling features

**References:**
- `betting-setup.md` - Platform initialization, market channels, odds broadcasting
- `betting-wagers.md` - Wager validation, bet settlement, cash-out
- `betting-patterns.md` - Casino game sync, in-play patterns, compliance

---

### pubnub-live-sport-updates

Deliver real-time sports scores, play-by-play, and scoreboards with PubNub.

**Use for:**
- Live scoreboards and score tickers
- Play-by-play and timeline feeds
- Multi-sport dashboards
- Fan engagement features

**References:**
- `sport-updates-setup.md` - Channel hierarchy, data models, subscription patterns
- `sport-updates-events.md` - Game event types, scoring logic, play-by-play
- `sport-updates-patterns.md` - Multi-sport dashboards, fan engagement, scaling

---

### pubnub-live-stock-quote-updates

Deliver real-time stock quotes and market data with PubNub.

**Use for:**
- Live stock quote streaming
- Portfolio tracking and watchlists
- Price alert systems
- Ticker displays and charting dashboards

**References:**
- `stock-quotes-setup.md` - Channel design, quote broadcasting, ingestion
- `stock-quotes-portfolio.md` - Watchlist management, price alerts
- `stock-quotes-patterns.md` - Ticker displays, charting, market hours

---

### pubnub-live-voting

Build real-time voting and polling systems with PubNub.

**Use for:**
- Live audience polling and voting
- Real-time vote tallying with duplicate prevention
- Survey systems with live results
- Multi-round voting workflows

**References:**
- `voting-setup.md` - Poll creation, channel design, lifecycle management
- `voting-tallying.md` - Duplicate prevention, atomic counters, fraud detection
- `voting-patterns.md` - Result broadcasting, multi-round voting, weighted votes

---

### pubnub-multiplayer-gaming

Build real-time multiplayer games with PubNub game state sync.

**Use for:**
- Game lobbies and room management
- Game state synchronization with delta updates
- Matchmaking systems
- Player connection and reconnection handling

**References:**
- `gaming-setup.md` - Game room creation, lobby management
- `gaming-state-sync.md` - Game state synchronization, delta updates
- `gaming-patterns.md` - Matchmaking, turn-based/real-time patterns, anti-cheat

---

### pubnub-order-delivery-driver

Build real-time order tracking and delivery driver systems with PubNub.

**Use for:**
- GPS location streaming from driver apps
- Order status pipeline management
- Dispatch coordination and driver assignment
- Customer-facing tracking pages

**References:**
- `delivery-setup.md` - Channel design, GPS publishing, SDK initialization
- `delivery-status.md` - Order lifecycle states, ETA calculation, geofencing
- `delivery-patterns.md` - Dispatch coordination, fleet dashboards, proof of delivery

---

### pubnub-telemedicine

Build HIPAA-compliant telemedicine apps with PubNub real-time messaging.

**Use for:**
- Secure patient-provider communication
- Virtual waiting rooms and patient queues
- WebRTC video consultation signaling
- Audit logging and consent management

**References:**
- `telemedicine-setup.md` - HIPAA configuration, encryption, Access Manager
- `telemedicine-features.md` - Patient queue management, notifications, consent
- `telemedicine-patterns.md` - Consultation workflows, WebRTC signaling, audit logging

## Publishing & Updating Skills

### Initial Publish

To publish a new skill to the `pubnub` workspace:

```bash
tessl skill publish --workspace pubnub --public ./skills/<skill-name>
```

### Publish All Skills (admin-only)

To publish or update all PubNub skills:

```bash
# From the tessl directory
npm i -g @tessl/cli
tessl login
tessl skill publish --workspace pubnub --public ./skills/pubnub-app-developer
tessl skill publish --workspace pubnub --public ./skills/pubnub-presence
tessl skill publish --workspace pubnub --public ./skills/pubnub-functions
tessl skill publish --workspace pubnub --public ./skills/pubnub-security
tessl skill publish --workspace pubnub --public ./skills/pubnub-chat
tessl skill publish --workspace pubnub --public ./skills/pubnub-scale
tessl skill publish --workspace pubnub --public ./skills/pubnub-choose-docs-path
tessl skill publish --workspace pubnub --public ./skills/pubnub-keyset-management
tessl skill publish --workspace pubnub --public ./skills/pubnub-app-context
tessl skill publish --workspace pubnub --public ./skills/pubnub-illuminate
tessl skill publish --workspace pubnub --public ./skills/pubnub-history
tessl skill publish --workspace pubnub --public ./skills/pubnub-reliability
tessl skill publish --workspace pubnub --public ./skills/pubnub-observability
tessl skill publish --workspace pubnub --public ./skills/pubnub-events-and-actions
tessl skill publish --workspace pubnub --public ./skills/pubnub-live-auctions
tessl skill publish --workspace pubnub --public ./skills/pubnub-live-betting-casino
tessl skill publish --workspace pubnub --public ./skills/pubnub-live-sport-updates
tessl skill publish --workspace pubnub --public ./skills/pubnub-live-stock-quote-updates
tessl skill publish --workspace pubnub --public ./skills/pubnub-live-voting
tessl skill publish --workspace pubnub --public ./skills/pubnub-multiplayer-gaming
tessl skill publish --workspace pubnub --public ./skills/pubnub-order-delivery-driver
tessl skill publish --workspace pubnub --public ./skills/pubnub-telemedicine
```

### Updating a Skill

1. **Edit the skill files** - Update `SKILL.md` or reference files in the skill directory

2. **Bump the version** - Update the version in `tile.json`:
   ```json
   {
     "version": "0.2.0"
   }
   ```

3. **Re-publish** - Run the publish command again:
   ```bash
   tessl skill publish --workspace pubnub --public ./skills/<skill-name>
   ```

### Skill Structure

Each skill follows this directory structure:

```
skills/<skill-name>/
├── SKILL.md           # Main skill definition with frontmatter
├── tile.json          # Generated manifest (auto-created on first publish)
└── references/        # Supporting documentation
    ├── <topic-1>.md
    ├── <topic-2>.md
    └── <topic-3>.md
```

### SKILL.md Frontmatter

Each `SKILL.md` requires YAML frontmatter:

```yaml
---
name: skill-name
description: Short description for CLI display
license: PubNub
metadata:
  author: pubnub
  version: "1.0.0"
  domain: real-time
  triggers: comma, separated, keywords
  role: specialist
  scope: implementation
  output-format: code
---
```

### Verify Publication

After publishing, verify the skill is available:

```bash
tessl skill search pubnub
```

## Workspace

All skills are published to the `pubnub` workspace on tessl.io.

## Version

Catalog versions:

- **8 vertical skills at `0.2.0`** (refreshed in this release): `pubnub-live-auctions`, `pubnub-live-betting-casino`, `pubnub-live-sport-updates`, `pubnub-live-stock-quote-updates`, `pubnub-live-voting`, `pubnub-multiplayer-gaming`, `pubnub-order-delivery-driver`, `pubnub-telemedicine`.
- **5 foundational skills at `0.2.0`**: `pubnub-app-developer`, `pubnub-presence`, `pubnub-security`, `pubnub-chat`, `pubnub-scale`.
- **`pubnub-functions` at `0.2.2`** (pending separate update).
- **8 cross-cutting skills at `0.1.0`** (initial release): `pubnub-choose-docs-path`, `pubnub-keyset-management`, `pubnub-app-context`, `pubnub-illuminate`, `pubnub-history`, `pubnub-reliability`, `pubnub-observability`, `pubnub-events-and-actions`.

## License

PubNub

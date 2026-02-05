# PubNub Skills Collection

A comprehensive set of skills for building real-time applications with PubNub's platform. Each skill provides focused guidance, code examples, and best practices for specific PubNub capabilities.

## Available Skills

| Skill | Description | Install |
|-------|-------------|---------|
| [pubnub-app-developer](./pubnub-app-developer) | Core pub/sub messaging, channels, SDK patterns | `tessl skill install pubnub:pubnub-app-developer` |
| [pubnub-presence](./pubnub-presence) | Real-time user online/offline status, occupancy | `tessl skill install pubnub:pubnub-presence` |
| [pubnub-functions](./pubnub-functions) | Serverless edge functions, webhooks, KVStore | `tessl skill install pubnub:pubnub-functions` |
| [pubnub-security](./pubnub-security) | Access Manager, AES-256 encryption, TLS | `tessl skill install pubnub:pubnub-security` |
| [pubnub-chat](./pubnub-chat) | Chat SDK, messaging, typing indicators, reactions | `tessl skill install pubnub:pubnub-chat` |
| [pubnub-scale](./pubnub-scale) | High-volume optimization, channel groups, persistence | `tessl skill install pubnub:pubnub-scale` |

## Quick Start

### Install the CLI

The skills CLI requires no installation and can be run directly using npx:

```bash
npx skills add <skill-name>
```

Alternatively, install globally:

```bash
npm install -g skills
```

For more details, see the [CLI documentation](https://skills.sh/docs/cli).

### Search for Skills

```bash
tessl skill search pubnub
```

### Install a Skill

Using the tessl CLI:

```bash
tessl skill install pubnub:pubnub-app-developer
```

Using npx (no installation required):

```bash
npx skills add pubnub/pubnub-app-developer
npx skills add pubnub/pubnub-presence
npx skills add pubnub/pubnub-functions
npx skills add pubnub/pubnub-security
npx skills add pubnub/pubnub-chat
npx skills add pubnub/pubnub-scale
```

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
- `access-manager.md` - PAM setup and grants
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
- Message persistence and history
- Large-scale event handling (10K+ users)

**References:**
- `scaling-patterns.md` - Channel groups, wildcards
- `performance.md` - Message optimization, batching
- `persistence.md` - History API, catch-up patterns

## Publishing & Updating Skills

### Initial Publish

To publish a new skill to the `pubnub` workspace:

```bash
tessl skill publish --workspace pubnub --public ./skills/<skill-name>
```

### Publish All Skills

To publish or update all PubNub skills:

```bash
# From the tessl directory
tessl skill publish --workspace pubnub --public ./skills/pubnub-app-developer
tessl skill publish --workspace pubnub --public ./skills/pubnub-presence
tessl skill publish --workspace pubnub --public ./skills/pubnub-functions
tessl skill publish --workspace pubnub --public ./skills/pubnub-security
tessl skill publish --workspace pubnub --public ./skills/pubnub-chat
tessl skill publish --workspace pubnub --public ./skills/pubnub-scale
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
license: MIT
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

All skills are currently at version `0.1.0`.

## License

MIT

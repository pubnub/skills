---
name: pubnub-telemedicine
description: Build HIPAA-compliant telemedicine apps with PubNub real-time messaging
license: PubNub
metadata:
  author: pubnub
  version: "0.2.0"
  domain: real-time
  triggers: pubnub, telemedicine, hipaa, telehealth, patient queue, healthcare, video chat, appointment
  role: specialist
  scope: implementation
  output-format: code
---

<!-- xrefs-injected -->

> **Canonical owners (link-don't-copy):** This vertical relies on cross-cutting skills. Always link to the canonical owner instead of duplicating. Foundations: [SDK initialization (`new PubNub(`, `userId`/UUID)](../pubnub-app-developer/references/sdk-patterns.md), [pub/sub basics (`pubnub.publish(`, `pubnub.subscribe(`, `addListener`)](../pubnub-app-developer/references/publish-subscribe.md), [channel naming](../pubnub-app-developer/references/channels.md), [message filters](../pubnub-app-developer/references/message-filters.md), [SDK upgrades](../pubnub-app-developer/references/sdk-upgrades.md), [REST API](../pubnub-app-developer/references/rest-api.md). Environment: [keysets, env separation, publish/subscribe/secret keys](../pubnub-keyset-management/references/keysets-and-environments.md), [key rotation hygiene](../pubnub-keyset-management/references/key-rotation-and-hygiene.md), [demo keys](../pubnub-keyset-management/references/demo-keys.md), [custom origin](../pubnub-keyset-management/references/custom-origin.md). Security: [Access Manager / `grantToken`](../pubnub-security/references/access-manager.md), [AES-256 / message encryption](../pubnub-security/references/encryption.md), [IP allowlisting](../pubnub-security/references/ip-whitelisting.md), [DoS mitigation](../pubnub-security/references/dos-mitigation.md), [compliance / SOC 2 / HIPAA](../pubnub-security/references/compliance-reports.md). Real-time features: [presence events / `withPresence`](../pubnub-presence/references/presence-events.md), [presence setup / heartbeat](../pubnub-presence/references/presence-setup.md), [dropped connections](../pubnub-presence/references/dropped-connections.md), [multi-device sync](../pubnub-presence/references/multi-device-sync.md). History: [Message Persistence and `fetchMessages`](../pubnub-history/references/pagination-and-ordering.md), [offline catch-up](../pubnub-history/references/offline-catch-up.md), [retention](../pubnub-history/references/retention-and-storage.md). App Context: [users / user metadata](../pubnub-app-context/references/users.md), [channels and memberships](../pubnub-app-context/references/channels-and-memberships.md), [metadata and filtering](../pubnub-app-context/references/metadata-and-filtering.md). Functions: [Before/After Publish, `request.ok()`/`request.abort()`](../pubnub-functions/references/functions-basics.md), [`require('kvstore')`/`xhr`/`vault`](../pubnub-functions/references/functions-modules.md), [chaining (3-hop limit)](../pubnub-functions/references/functions-chaining.md), [DB triggers and runtime quirks](../pubnub-functions/references/db-triggers-and-runtime-quirks.md), [common patterns](../pubnub-functions/references/functions-patterns.md). Reliability: [exponential backoff and jitter](../pubnub-reliability/references/backoff-and-jitter.md), [idempotent publish / message id](../pubnub-reliability/references/idempotent-publish.md), [dedup on merge](../pubnub-reliability/references/dedup-on-merge.md), [queue and retry](../pubnub-reliability/references/queue-and-retry.md), [schema version](../pubnub-reliability/references/schema-versioning.md). Scale: [channel groups, wildcard subscribe, Stream Controller](../pubnub-scale/references/scaling-patterns.md), [performance tuning](../pubnub-scale/references/performance.md), [10K+ live events](../pubnub-scale/references/large-events.md). Observability: [logging correlation (channel + message_id + user_id + timetoken)](../pubnub-observability/references/logging-correlation.md), [test pyramid](../pubnub-observability/references/test-pyramid.md), [payload sizing / cost](../pubnub-observability/references/cost-and-payload-hygiene.md), [incident triage runbook](../pubnub-observability/references/incident-runbook.md), [usage metrics / transaction count](../pubnub-observability/references/usage-metrics.md). Events & Actions: [event types](../pubnub-events-and-actions/references/event-types.md), [action targets (webhook / SQS / Kafka / Lambda)](../pubnub-events-and-actions/references/action-targets.md), [filters / JSONPath](../pubnub-events-and-actions/references/filters-and-jsonpath.md). Illuminate: [Business Objects](../pubnub-illuminate/references/business-objects.md), [Metrics](../pubnub-illuminate/references/metrics.md), [Decisions (4-step workflow)](../pubnub-illuminate/references/decisions-4-step-workflow.md), [Queries](../pubnub-illuminate/references/queries-adhoc-vs-saved.md), [service integration auth](../pubnub-illuminate/references/service-integration-auth.md). Chat: [Chat SDK setup](../pubnub-chat/references/chat-setup.md), [message actions / reactions](../pubnub-chat/references/message-actions.md), [file sharing / `sendFile`](../pubnub-chat/references/file-sharing.md), [threading](../pubnub-chat/references/threading.md). Routing: [intent-to-tool decision tree (`get_sdk_documentation`, `write_pubnub_app`, etc.)](../pubnub-choose-docs-path/references/intent-to-tool.md).


# PubNub Telemedicine Specialist

You are a specialist in building HIPAA-compliant telemedicine applications using PubNub's real-time messaging infrastructure. You help developers implement secure patient-provider communication, virtual waiting rooms, video consultation signaling, appointment notifications, and healthcare data exchange — all while meeting strict regulatory requirements for protected health information (PHI).

## When to Use This Skill

Invoke this skill when:

- Building a telemedicine or telehealth application that requires real-time messaging between patients and healthcare providers
- Implementing HIPAA-compliant communication channels that handle protected health information (PHI)
- Creating virtual waiting rooms and patient queue management systems
- Setting up WebRTC video consultation signaling through PubNub channels
- Designing appointment scheduling, reminders, and provider availability tracking
- Implementing audit logging, message retention policies, and consent management for healthcare compliance

## Core Workflow

1. **Assess Healthcare Requirements** — Identify the specific telemedicine use case, compliance requirements (HIPAA, BAA), patient/provider roles, and PHI data flows that the application must support.

2. **Configure Secure Infrastructure** — Set up PubNub with AES-256 encryption, Access Manager token-based authorization, and audit logging to establish a HIPAA-compliant foundation. Reference `telemedicine-setup.md` for detailed configuration.

3. **Implement Patient-Provider Channels** — Design channel architecture for one-on-one consultations, group consultations, waiting rooms, and notification delivery using healthcare-specific naming conventions and access controls.

4. **Build Telemedicine Features** — Implement patient queue management, real-time notifications, provider availability tracking, consent management, and secure file sharing. Reference `telemedicine-features.md` for feature implementation details.

5. **Integrate Consultation Patterns** — Wire up consultation workflows including check-in, waiting room, video signaling, multi-provider sessions, emergency escalation, and follow-up. Reference `telemedicine-patterns.md` for architectural patterns.

6. **Validate Compliance and Test** — Verify encryption is active on all PHI channels, confirm Access Manager policies enforce least-privilege, validate audit logs capture all required events, and test message retention and deletion policies.

## Reference Guide

| Reference | Purpose |
|-----------|---------|
| [telemedicine-setup.md](references/telemedicine-setup.md) | HIPAA configuration, encryption setup, Access Manager for healthcare roles, BAA requirements, and SDK initialization |
| [telemedicine-features.md](references/telemedicine-features.md) | Patient queue management, real-time notifications, provider availability, consent management, and secure file sharing |
| [telemedicine-patterns.md](references/telemedicine-patterns.md) | Consultation workflows, WebRTC video signaling, audit logging, multi-provider sessions, and emergency escalation |

## Key Implementation Requirements

### HIPAA-Compliant PubNub Configuration

Every telemedicine application must initialize PubNub with encryption enabled and Access Manager enforcing role-based access. PHI must never traverse unencrypted channels.

```javascript
import PubNub from 'pubnub';

const pubnub = new PubNub({
  publishKey: process.env.PUBNUB_PUBLISH_KEY,
  subscribeKey: process.env.PUBNUB_SUBSCRIBE_KEY,
  secretKey: process.env.PUBNUB_SECRET_KEY, // Server-side only
  userId: currentUser.id,
  cryptoModule: PubNub.CryptoModule.aesCbcCryptoModule({
    cipherKey: process.env.PUBNUB_CIPHER_KEY
  }),
  ssl: true,
  logVerbosity: false // Disable in production to prevent PHI leaks in logs
});
```

### Encrypted Messaging for PHI

All messages containing patient data must be published on encrypted channels with proper access tokens. Message payloads should minimize PHI exposure.

```javascript
async function sendSecureMessage(channelId, message, senderRole) {
  const payload = {
    id: crypto.randomUUID(),
    type: message.type,
    content: message.content,
    sender: {
      id: message.senderId,
      role: senderRole // 'provider' | 'patient' | 'nurse'
    },
    timestamp: new Date().toISOString(),
    metadata: {
      encrypted: true,
      consentVerified: true,
      auditRef: crypto.randomUUID()
    }
  };

  try {
    const result = await pubnub.publish({
      channel: channelId,
      message: payload,
      storeInHistory: true,
      meta: {
        senderRole: senderRole,
        messageType: message.type
      }
    });
    await logAuditEvent('MESSAGE_SENT', channelId, payload.metadata.auditRef);
    return result;
  } catch (error) {
    await logAuditEvent('MESSAGE_FAILED', channelId, payload.metadata.auditRef);
    throw new Error(`Secure message delivery failed: ${error.message}`);
  }
}
```

### Access Manager for Healthcare Roles

Use PubNub Access Manager to enforce role-based access. Providers can access consultation channels, patients can only access their own channels, and administrative staff have scoped permissions.

```javascript
async function grantProviderAccess(providerId, consultationChannelId, ttlMinutes = 60) {
  const token = await pubnub.grantToken({
    ttl: ttlMinutes,
    authorizedUUID: providerId,
    resources: {
      channels: {
        [consultationChannelId]: {
          read: true,
          write: true,
          get: true,
          update: true
        },
        [`${consultationChannelId}.files`]: {
          read: true,
          write: true
        }
      }
    },
    patterns: {
      channels: {
        [`consultation.${providerId}.*`]: {
          read: true,
          write: true
        }
      }
    }
  });
  return token;
}

async function grantPatientAccess(patientId, consultationChannelId, ttlMinutes = 30) {
  const token = await pubnub.grantToken({
    ttl: ttlMinutes,
    authorizedUUID: patientId,
    resources: {
      channels: {
        [consultationChannelId]: {
          read: true,
          write: true
        }
      }
    }
  });
  return token;
}
```

## Constraints

- All channels transmitting PHI must use AES-256 encryption via PubNub's CryptoModule — never send unencrypted health data
- A signed Business Associate Agreement (BAA) with PubNub must be in place before handling any PHI in production
- Access Manager tokens must enforce least-privilege and use short TTLs (15-60 minutes) that match consultation session durations
- Message history retention must comply with organizational and jurisdictional record-keeping requirements (typically 6-10 years for medical records)
- Audit logs must capture all message events, access grants, and consent actions for HIPAA compliance verification
- Never log PHI to console, application logs, or third-party monitoring services — audit logs must store references, not raw patient data

## MCP Tools

- **`get_chat_sdk_documentation`** — pull Chat SDK reference for the patient-provider conversation surface (route via [intent-to-tool](../pubnub-choose-docs-path/references/intent-to-tool.md))
- **`get_sdk_documentation`** — pull SDK-specific publish/subscribe APIs
- **`grant_token`** — issue scoped grants per encounter (patient + provider only, short TTL)
- **`create_pubnub_function`** — scaffold the Before-Publish consent / PHI redaction validator
- **`manage_apps`** — verify Message Persistence and add-ons against your BAA

## See Also

- **[pubnub-security](../pubnub-security/SKILL.md)** — [Access Manager](../pubnub-security/references/access-manager.md) for per-encounter grants, [AES-256 / message encryption](../pubnub-security/references/encryption.md) for PHI, [IP allowlisting](../pubnub-security/references/ip-whitelisting.md) for clinical backends, [compliance / HIPAA / SOC 2](../pubnub-security/references/compliance-reports.md) (start here for BAA flow)
- **[pubnub-functions](../pubnub-functions/SKILL.md)** — [Before Publish for consent verification and PHI redaction](../pubnub-functions/references/functions-basics.md), [`require('vault')` for keys](../pubnub-functions/references/functions-modules.md), [DB-trigger to audit log](../pubnub-functions/references/db-triggers-and-runtime-quirks.md)
- **[pubnub-presence](../pubnub-presence/SKILL.md)** — [provider availability and patient connection status](../pubnub-presence/references/presence-events.md), [dropped-connection recovery during a visit](../pubnub-presence/references/dropped-connections.md), [multi-device sync (provider tablet + workstation)](../pubnub-presence/references/multi-device-sync.md)
- **[pubnub-chat](../pubnub-chat/SKILL.md)** — [Chat SDK](../pubnub-chat/references/chat-setup.md) for patient-provider messaging, [file sharing for documents and images](../pubnub-chat/references/file-sharing.md), [threading for asynchronous follow-up](../pubnub-chat/references/threading.md)
- **[pubnub-reliability](../pubnub-reliability/SKILL.md)** — [idempotent publish](../pubnub-reliability/references/idempotent-publish.md) so retries don't duplicate clinical events; [queue-and-retry](../pubnub-reliability/references/queue-and-retry.md) for low-bandwidth patient apps
- **[pubnub-history](../pubnub-history/SKILL.md)** — [Message Persistence](../pubnub-history/references/pagination-and-ordering.md) for required clinical audit trails (configure [retention](../pubnub-history/references/retention-and-storage.md) per your retention policy)
- **[pubnub-app-context](../pubnub-app-context/SKILL.md)** — [provider directory, patient roster (PHI-safe portion only)](../pubnub-app-context/references/users.md)
- **[pubnub-events-and-actions](../pubnub-events-and-actions/SKILL.md)** — route consult-completed events to EHR / billing / BI via [action targets](../pubnub-events-and-actions/references/action-targets.md)
- **[pubnub-observability](../pubnub-observability/SKILL.md)** — [logging correlation](../pubnub-observability/references/logging-correlation.md) (audit-grade) and [incident runbook](../pubnub-observability/references/incident-runbook.md)
- **[pubnub-choose-docs-path](../pubnub-choose-docs-path/SKILL.md)** — for routing other PubNub questions

## Output Format

When providing implementations:

1. Always include the HIPAA-compliant PubNub initialization with encryption and Access Manager configuration
2. Provide complete, runnable code examples with proper error handling, audit logging, and consent verification
3. Include channel naming conventions that follow healthcare-specific patterns (e.g., `consultation.{providerId}.{patientId}`)
4. Document all compliance considerations inline with code comments explaining why specific security measures are required
5. Provide both client-side (patient/provider app) and server-side (token grants, audit logging) code where the feature requires it

# PubNub Telemedicine Setup

HIPAA/BAA legal primers and generic compliance checklists belong in counsel and [compliance-reports.md](../../pubnub-security/references/compliance-reports.md). This file is the PubNub wiring: encryption, Access Manager grants, channel names, and audit persist.

## AES-256 Encryption Setup

PHI channels must use CryptoModule. Keep one JS shape here; retrieve Python/Swift/Kotlin constructors via **`get_sdk_documentation`** (`feature: encryption`). Details: [encryption.md](../../pubnub-security/references/encryption.md).

```javascript
import PubNub from 'pubnub';

const pubnub = new PubNub({
  publishKey: process.env.PUBNUB_PUBLISH_KEY,
  subscribeKey: process.env.PUBNUB_SUBSCRIBE_KEY,
  userId: authenticatedUser.id,
  cryptoModule: PubNub.CryptoModule.aesCbcCryptoModule({
    cipherKey: process.env.PUBNUB_CIPHER_KEY
  }),
  ssl: true,
  logVerbosity: false,
  heartbeatInterval: 30,
  presenceTimeout: 120
});
```

Never hardcode cipher keys. Rotate keys with a dual-read window as described in the encryption skill — do not duplicate a KMS tutorial here.

## Access Manager Configuration

Access Manager v3 enforces authorization at the channel level. Every user must present a valid token to read or write on any channel.

### Server-Side Token Generation

The token server runs on your backend and issues scoped, time-limited tokens after verifying user identity and role.

```javascript
import PubNub from 'pubnub';

const pubnubAdmin = new PubNub({
  publishKey: process.env.PUBNUB_PUBLISH_KEY,
  subscribeKey: process.env.PUBNUB_SUBSCRIBE_KEY,
  secretKey: process.env.PUBNUB_SECRET_KEY,
  userId: 'telemedicine-server'
});

async function issueHealthcareToken(userId, role, sessionContext) {
  const permissions = buildPermissions(role, sessionContext);

  const token = await pubnubAdmin.grantToken({
    ttl: permissions.ttlMinutes,
    authorizedUUID: userId,
    resources: permissions.resources,
    patterns: permissions.patterns
  });

  await logTokenGrant(userId, role, permissions.ttlMinutes);
  return token;
}

function buildPermissions(role, context) {
  switch (role) {
    case 'provider':
      return {
        ttlMinutes: 60,
        resources: {
          channels: {
            [context.consultationChannel]: { read: true, write: true, get: true, update: true },
            [context.queueChannel]: { read: true, write: true },
            [`${context.consultationChannel}.files`]: { read: true, write: true }
          }
        },
        patterns: {
          channels: {
            [`consultation.${context.providerId}.*`]: { read: true, write: true },
            [`notification.${context.providerId}`]: { read: true }
          }
        }
      };

    case 'patient':
      return {
        ttlMinutes: 30,
        resources: {
          channels: {
            [context.consultationChannel]: { read: true, write: true },
            [`waiting-room.${context.providerId}`]: { read: true, write: true }
          }
        },
        patterns: {}
      };

    case 'nurse':
      return {
        ttlMinutes: 60,
        resources: {
          channels: {
            [context.consultationChannel]: { read: true, write: true },
            [context.queueChannel]: { read: true, write: true, get: true, update: true }
          }
        },
        patterns: {
          channels: {
            [`queue.${context.departmentId}.*`]: { read: true, write: true }
          }
        }
      };

    default:
      throw new Error(`Unknown healthcare role: ${role}`);
  }
}
```

### Client-Side Token Application

```javascript
async function initializePatientSession(patientId, appointmentId) {
  const response = await fetch('/api/telemedicine/token', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${authToken}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ patientId, appointmentId, role: 'patient' })
  });

  const { token, channelConfig } = await response.json();

  pubnub.setToken(token);

  pubnub.subscribe({
    channels: [channelConfig.consultationChannel, channelConfig.waitingRoom]
  });

  return channelConfig;
}
```

Execute a BAA before production PHI. Start at [compliance-reports.md](../../pubnub-security/references/compliance-reports.md) — do not copy a BAA clause table into this skill.

## Channel Naming Conventions

Consistent channel naming is critical for Access Manager patterns and operational clarity.

| Channel Pattern | Purpose | Example |
|----------------|---------|---------|
| `consultation.{providerId}.{patientId}` | One-on-one consultation messaging | `consultation.dr-smith.patient-abc123` |
| `consultation.{providerId}.{patientId}.video` | WebRTC video signaling | `consultation.dr-smith.patient-abc123.video` |
| `consultation.{providerId}.{patientId}.files` | Secure file sharing | `consultation.dr-smith.patient-abc123.files` |
| `waiting-room.{providerId}` | Patient queue / waiting room | `waiting-room.dr-smith` |
| `queue.{departmentId}` | Department-level patient queue | `queue.cardiology` |
| `notification.{userId}` | Personal notification delivery | `notification.dr-smith` |
| `presence.providers.{departmentId}` | Provider availability tracking | `presence.providers.cardiology` |
| `audit.{organizationId}` | Audit event log channel | `audit.clinic-north` |
| `emergency.{departmentId}` | Emergency escalation channel | `emergency.oncology` |

## Audit Logging Configuration

Publish audit events to `audit.{organizationId}` with references (IDs), not raw PHI.

```javascript
class TelemedicineAuditLogger {
  constructor(pubnub, organizationId) {
    this.pubnub = pubnub;
    this.organizationId = organizationId;
    this.auditChannel = `audit.${organizationId}`;
  }

  async logEvent(eventType, details) {
    const auditEntry = {
      id: crypto.randomUUID(),
      timestamp: new Date().toISOString(),
      eventType: eventType,
      organizationId: this.organizationId,
      actor: {
        userId: details.userId,
        role: details.userRole,
        ipAddress: details.ipAddress || 'unknown'
      },
      resource: {
        type: details.resourceType,
        id: details.resourceId,
        channel: details.channel
      },
      action: details.action,
      outcome: details.outcome || 'success',
      metadata: {
        sessionId: details.sessionId,
        consentRef: details.consentRef
      }
    };

    await this.pubnub.publish({
      channel: this.auditChannel,
      message: auditEntry,
      storeInHistory: true
    });

    return auditEntry.id;
  }
}

const AUDIT_EVENTS = {
  SESSION_START: 'CONSULTATION_SESSION_START',
  SESSION_END: 'CONSULTATION_SESSION_END',
  MESSAGE_SENT: 'PHI_MESSAGE_SENT',
  MESSAGE_READ: 'PHI_MESSAGE_READ',
  FILE_SHARED: 'PHI_FILE_SHARED',
  FILE_ACCESSED: 'PHI_FILE_ACCESSED',
  TOKEN_GRANTED: 'ACCESS_TOKEN_GRANTED',
  TOKEN_REVOKED: 'ACCESS_TOKEN_REVOKED',
  CONSENT_GIVEN: 'PATIENT_CONSENT_GIVEN',
  CONSENT_REVOKED: 'PATIENT_CONSENT_REVOKED',
  VIDEO_STARTED: 'VIDEO_SESSION_STARTED',
  VIDEO_ENDED: 'VIDEO_SESSION_ENDED',
  EMERGENCY_ESCALATION: 'EMERGENCY_ESCALATION_TRIGGERED'
};
```

### Audit Log Persistence

Persist audit logs beyond PubNub history with an After-Publish Function on `audit.*`.

```javascript
export default async (event) => {
  const message = event.message;
  const xhr = require('xhr');

  const response = await xhr.fetch('https://your-audit-api.example.com/audit-events', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${vault.get('AUDIT_API_KEY')}`
    },
    body: JSON.stringify(message)
  });
  if (response.status !== 201) {
    console.log('Audit persistence failed:', response.status);
  }
  return event;
};
```

## Best Practices

### Security

- Rotate cipher keys on a regular schedule and support decryption with previous keys during transition periods ([encryption.md](../../pubnub-security/references/encryption.md))
- Use short TTLs for Access Manager tokens — 15 minutes for patients, 60 minutes for providers
- Implement token refresh that obtains new tokens before expiry without interrupting the session
- Never expose the PubNub secret key in client-side code; all token grants must originate from your backend

### Architecture

- Deploy a dedicated token server that validates identity before issuing PubNub tokens
- Use separate PubNub keysets for production and non-production
- Design message payloads to carry the minimum PHI necessary — use references (patient ID) instead of inline demographics

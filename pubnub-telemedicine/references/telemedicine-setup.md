# PubNub Telemedicine Setup

This reference covers the foundational configuration required to build HIPAA-compliant telemedicine applications with PubNub. Every setting described here is mandatory for production healthcare deployments that handle protected health information (PHI).

## HIPAA Configuration Requirements

HIPAA mandates specific technical safeguards for electronic PHI (ePHI). PubNub provides the infrastructure to meet these requirements, but correct configuration is the developer's responsibility.

| HIPAA Safeguard | Requirement | PubNub Feature |
|----------------|-------------|----------------|
| Encryption in Transit | All ePHI must be encrypted during transmission | TLS/SSL (enabled by default) |
| Encryption at Rest | Stored messages must be encrypted | AES-256 CryptoModule |
| Access Controls | Role-based access to PHI | Access Manager v3 with token grants |
| Audit Controls | Log all access to ePHI | PubNub Functions + custom audit logging |
| Integrity Controls | Protect ePHI from improper alteration | Message immutability + signed payloads |
| Transmission Security | Guard against unauthorized access during transmission | Channel-level encryption + token auth |
| Automatic Logoff | Terminate sessions after inactivity | Token TTL + Presence heartbeat timeout |

### Pre-Deployment Checklist

Before deploying a telemedicine application to production:

1. Execute a Business Associate Agreement (BAA) with PubNub
2. Enable Access Manager on your PubNub keyset
3. Configure AES-256 encryption for all PHI channels
4. Set up audit logging infrastructure
5. Implement token-based authentication with short TTLs
6. Configure message retention policies
7. Disable verbose logging in production builds
8. Complete a HIPAA security risk assessment

## AES-256 Encryption Setup

PubNub's CryptoModule provides AES-256-CBC encryption for all message payloads. This is mandatory for any channel that carries PHI.

### JavaScript SDK Encryption

```javascript
import PubNub from 'pubnub';

// Production HIPAA-compliant initialization
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

### Python SDK Encryption

```python
from pubnub.pnconfiguration import PNConfiguration
from pubnub.pubnub import PubNub
from pubnub.crypto import AesCbcCryptoModule
import os

pnconfig = PNConfiguration()
pnconfig.publish_key = os.environ['PUBNUB_PUBLISH_KEY']
pnconfig.subscribe_key = os.environ['PUBNUB_SUBSCRIBE_KEY']
pnconfig.user_id = authenticated_user.id
pnconfig.crypto_module = AesCbcCryptoModule(
    cipher_key=os.environ['PUBNUB_CIPHER_KEY']
)
pnconfig.ssl = True
pnconfig.log_verbosity = False

pubnub = PubNub(pnconfig)
```

### Swift SDK Encryption (iOS)

```swift
import PubNub

let config = PubNubConfiguration(
    publishKey: ProcessInfo.processInfo.environment["PUBNUB_PUBLISH_KEY"]!,
    subscribeKey: ProcessInfo.processInfo.environment["PUBNUB_SUBSCRIBE_KEY"]!,
    userId: authenticatedUser.id,
    cryptoModule: CryptoModule.aesCbcCryptoModule(with: cipherKey),
    useSecureConnections: true
)
let pubnub = PubNub(configuration: config)
```

### Kotlin SDK Encryption (Android)

```kotlin
import com.pubnub.api.PNConfiguration
import com.pubnub.api.PubNub
import com.pubnub.api.crypto.CryptoModule

val pnConfiguration = PNConfiguration(userId = UserId(authenticatedUser.id)).apply {
    publishKey = BuildConfig.PUBNUB_PUBLISH_KEY
    subscribeKey = BuildConfig.PUBNUB_SUBSCRIBE_KEY
    cryptoModule = CryptoModule.createAesCbcCryptoModule(
        cipherKey = BuildConfig.PUBNUB_CIPHER_KEY
    )
    secure = true
    logVerbosity = PNLogVerbosity.NONE
}
val pubnub = PubNub.create(pnConfiguration)
```

### Cipher Key Management

Never hardcode cipher keys. Use a key management service (KMS) for production deployments.

```javascript
// Key rotation support
class HealthcareCryptoManager {
  constructor(kmsClient) {
    this.kmsClient = kmsClient;
    this.keyCache = new Map();
  }

  async getCurrentCipherKey() {
    const keyId = await this.kmsClient.getCurrentKeyId('pubnub-phi-encryption');
    if (this.keyCache.has(keyId)) {
      return this.keyCache.get(keyId);
    }
    const key = await this.kmsClient.decrypt(keyId);
    this.keyCache.set(keyId, key);
    return key;
  }

  async rotateCipherKey() {
    const newKeyId = await this.kmsClient.generateKey('pubnub-phi-encryption');
    this.keyCache.clear();
    return newKeyId;
  }
}
```

## Access Manager Configuration

Access Manager v3 enforces authorization at the channel level. Every user must present a valid token to read or write on any channel.

### Server-Side Token Generation

The token server runs on your backend and issues scoped, time-limited tokens after verifying user identity and role.

```javascript
import PubNub from 'pubnub';

// Server-side PubNub instance with secret key
const pubnubAdmin = new PubNub({
  publishKey: process.env.PUBNUB_PUBLISH_KEY,
  subscribeKey: process.env.PUBNUB_SUBSCRIBE_KEY,
  secretKey: process.env.PUBNUB_SECRET_KEY,
  userId: 'telemedicine-server'
});

// Grant token based on healthcare role
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
// After authenticating, the client requests a token from your backend
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

  // Subscribe to assigned channels
  pubnub.subscribe({
    channels: [channelConfig.consultationChannel, channelConfig.waitingRoom]
  });

  return channelConfig;
}
```

## Business Associate Agreement (BAA) Requirements

A BAA is a legal contract required by HIPAA whenever a covered entity shares PHI with a business associate. PubNub acts as a business associate when your application transmits PHI through their infrastructure.

| BAA Requirement | Developer Responsibility |
|----------------|--------------------------|
| Execute BAA with PubNub | Contact PubNub sales for enterprise healthcare plan |
| Data encryption | Configure AES-256 CryptoModule on all PHI channels |
| Access logging | Implement audit trail for all PHI access events |
| Breach notification | Set up monitoring and alerting for unauthorized access |
| Data disposal | Configure message retention and implement deletion APIs |
| Minimum necessary | Design payloads that include only required PHI fields |

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

HIPAA requires audit controls that record and examine activity in systems containing ePHI.

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

// Audit event types
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

Audit logs should be persisted to durable storage beyond PubNub's message retention window.

```javascript
// PubNub Function to persist audit logs to external storage
// Deploy as an After-Publish event handler on audit.* channels
export default (event) => {
  const message = event.message;
  const xhr = require('xhr');

  return xhr.fetch('https://your-audit-api.example.com/audit-events', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${vault.get('AUDIT_API_KEY')}`
    },
    body: JSON.stringify(message)
  }).then((response) => {
    if (response.status !== 201) {
      console.log('Audit persistence failed:', response.status);
    }
    return event;
  });
};
```

## Best Practices

### Security

- Rotate cipher keys on a regular schedule (quarterly minimum) and support decryption with previous keys during transition periods
- Use the shortest practical TTL for Access Manager tokens — 15 minutes for patients, 60 minutes for providers
- Implement token refresh logic that obtains new tokens before expiry without interrupting the user session
- Never expose the PubNub secret key in client-side code; all token grants must originate from your backend

### Architecture

- Deploy a dedicated token server that validates user identity against your auth provider before issuing PubNub tokens
- Use separate PubNub keysets for production and non-production environments — never use production keys in development
- Implement a channel registry that tracks active consultation channels and their participants
- Design message payloads to carry the minimum PHI necessary — use references (patient ID) instead of inline data (patient name, DOB) where possible

### Monitoring

- Set up alerts for token grant failures, encryption errors, and unusual access patterns
- Monitor Presence events to detect abandoned sessions and trigger automatic cleanup
- Track message delivery latency to ensure real-time communication meets clinical workflow requirements
- Log all Access Manager operations to your audit trail, not just message events

### Compliance

- Document your PubNub configuration as part of your HIPAA security risk assessment
- Maintain records of BAA execution, encryption key rotation, and access control policy changes
- Conduct periodic access reviews to ensure token grant policies match current staff roles
- Test your breach notification workflow with simulated unauthorized access scenarios

# PubNub Telemedicine Patterns

Architectural patterns for consultation workflows, video **signaling** on PubNub, audit persist, multi-provider sessions, and emergency escalation.

## Consultation Workflow

### Consultation States

| State | Active Channels |
|-------|----------------|
| `scheduled` | `notification.{patientId}` |
| `checked-in` / `waiting` | `waiting-room.{providerId}` |
| `connecting` / `in-progress` / `wrapping-up` | `consultation.{providerId}.{patientId}` plus `.video` and `.files` while in-progress |
| `completed` | None (unsubscribe; revoke encounter tokens) |

### Consultation Lifecycle Manager

```javascript
class ConsultationLifecycle {
  constructor(pubnub, auditLogger, tokenService) {
    this.pubnub = pubnub;
    this.auditLogger = auditLogger;
    this.tokenService = tokenService;
  }

  async startCheckIn(appointmentId, patientId, providerId) {
    const consultationId = `consultation.${providerId}.${patientId}`;

    const hasConsent = await this.verifyConsent(patientId, 'telehealth');
    if (!hasConsent) {
      throw new Error('Patient telehealth consent required before check-in');
    }

    const token = await this.tokenService.issueHealthcareToken(patientId, 'patient', {
      consultationChannel: consultationId,
      providerId
    });

    await this.pubnub.publish({
      channel: `waiting-room.${providerId}`,
      message: {
        action: 'PATIENT_CHECKED_IN',
        appointmentId, patientId, consultationId,
        timestamp: new Date().toISOString()
      }
    });

    await this.pubnub.publish({
      channel: `notification.${providerId}`,
      message: {
        type: 'PATIENT_READY', title: 'Patient Checked In',
        body: 'Patient is ready for consultation',
        appointmentId, consultationId, priority: 'high'
      }
    });

    await this.auditLogger.logEvent('CONSULTATION_SESSION_START', {
      userId: patientId, userRole: 'patient',
      resourceType: 'consultation', resourceId: consultationId,
      channel: `waiting-room.${providerId}`, action: 'CHECK_IN'
    });

    return { consultationId, token };
  }

  async providerJoin(consultationId, providerId) {
    const token = await this.tokenService.issueHealthcareToken(providerId, 'provider', {
      consultationChannel: consultationId, providerId
    });

    await this.pubnub.publish({
      channel: consultationId,
      message: {
        action: 'PROVIDER_JOINED', providerId,
        state: 'connecting', timestamp: new Date().toISOString()
      }
    });

    const patientId = consultationId.split('.').pop();
    await this.pubnub.publish({
      channel: `notification.${patientId}`,
      message: {
        type: 'PROVIDER_READY', title: 'Your Provider is Ready',
        body: 'Your consultation is about to begin.',
        consultationId, priority: 'high'
      }
    });

    return { token };
  }

  async endConsultation(consultationId, providerId, summary) {
    await this.pubnub.publish({
      channel: consultationId,
      message: {
        action: 'CONSULTATION_ENDED', providerId, state: 'completed',
        summary: {
          duration: summary.durationMinutes,
          followUpRequired: summary.followUpRequired
        },
        timestamp: new Date().toISOString()
      }
    });

    await this.auditLogger.logEvent('CONSULTATION_SESSION_END', {
      userId: providerId, userRole: 'provider',
      resourceType: 'consultation', resourceId: consultationId,
      action: 'END_CONSULTATION'
    });
  }

  async verifyConsent(patientId, consentType) {
    try {
      const response = await this.pubnub.getUUIDMetadata({ uuid: patientId });
      const data = response.data.custom?.[`consent_${consentType}`];
      if (!data) return false;
      const consent = JSON.parse(data);
      return consent.granted && (!consent.expiresAt || new Date(consent.expiresAt) > new Date());
    } catch {
      return false;
    }
  }
}
```

## WebRTC Video Signaling via PubNub

PubNub carries **signaling only** on `consultation.{providerId}.{patientId}.video`. Do not paste offer/answer/ICE tutorial classes here.

- Publish SDP/ICE JSON as opaque signaling messages; media never goes through PubNub.
- Set **`storeInHistory: false`** on the video signaling channel — signaling has no clinical retention value.
- Grant the `.video` channel in the same encounter token as the consultation channel.

```javascript
async function sendVideoSignal(pubnub, consultationId, signal, localUserId) {
  await pubnub.publish({
    channel: `${consultationId}.video`,
    message: { ...signal, senderId: localUserId, timestamp: new Date().toISOString() },
    storeInHistory: false
  });
}
```

## Audit Logging for Compliance

Every action involving PHI must be logged. Prefer the After-Publish persist on `audit.*` from [telemedicine-setup.md](telemedicine-setup.md).

```javascript
class ComplianceAuditTrail {
  constructor(pubnub, config) {
    this.pubnub = pubnub;
    this.auditChannel = `audit.${config.organizationId}`;
    this.persistenceEndpoint = config.persistenceEndpoint;
  }

  async log(eventType, severity, actor, resource, action, outcome, details = {}) {
    const event = {
      id: crypto.randomUUID(),
      timestamp: new Date().toISOString(),
      eventType, severity, actor, resource, action, outcome, details
    };

    await this.pubnub.publish({
      channel: this.auditChannel,
      message: event,
      storeInHistory: true
    });

    await this.persistEvent(event);
    return event.id;
  }

  async persistEvent(event) {
    try {
      const response = await fetch(this.persistenceEndpoint, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(event)
      });
      if (!response.ok) console.error(`Audit persistence failed: ${response.status}`);
    } catch (error) {
      console.error('CRITICAL: Audit persistence unavailable:', error.message);
    }
  }
}
```

## Multi-Provider Consultations

```javascript
class MultiProviderConsultation {
  constructor(pubnub, tokenService, auditTrail) {
    this.pubnub = pubnub;
    this.tokenService = tokenService;
    this.auditTrail = auditTrail;
  }

  async inviteProvider(consultationId, invitedProviderId, invitedByProviderId, role) {
    const token = await this.tokenService.issueHealthcareToken(invitedProviderId, 'provider', {
      consultationChannel: consultationId, providerId: invitedProviderId
    });

    await this.pubnub.publish({
      channel: `notification.${invitedProviderId}`,
      message: {
        type: 'CONSULTATION_INVITE', title: 'Consultation Invitation',
        body: `You have been invited to join a consultation as ${role}.`,
        consultationId, invitedBy: invitedByProviderId,
        role,
        priority: 'high'
      }
    });

    await this.pubnub.publish({
      channel: consultationId,
      message: {
        action: 'PROVIDER_INVITED', invitedProviderId,
        invitedBy: invitedByProviderId, role,
        timestamp: new Date().toISOString()
      }
    });

    await this.auditTrail.log(
      'PROVIDER_ADDED_TO_CONSULTATION', 'info',
      { userId: invitedByProviderId, role: 'provider' },
      { type: 'consultation', id: consultationId },
      `INVITE_PROVIDER_${role.toUpperCase()}`, 'success'
    );

    return { token };
  }

  async removeProvider(consultationId, providerId, removedByProviderId) {
    await this.pubnub.publish({
      channel: consultationId,
      message: {
        action: 'PROVIDER_REMOVED', providerId,
        removedBy: removedByProviderId,
        timestamp: new Date().toISOString()
      }
    });
    await this.tokenService.revokeAccess(providerId, consultationId);
  }
}
```

## Emergency Escalation Patterns

```javascript
class EmergencyEscalation {
  constructor(pubnub, auditTrail) {
    this.pubnub = pubnub;
    this.auditTrail = auditTrail;
  }

  async escalate(consultationId, escalatedBy, severity, reason) {
    const departmentId = consultationId.split('.')[1];
    const escalationId = crypto.randomUUID();

    await this.pubnub.publish({
      channel: `emergency.${departmentId}`,
      message: {
        id: escalationId, action: 'EMERGENCY_ESCALATION',
        consultationId, escalatedBy,
        severity,
        reason, timestamp: new Date().toISOString(), status: 'active'
      }
    });

    await this.notifyOnCallProviders(departmentId, escalationId, reason, severity);

    await this.auditTrail.log(
      'EMERGENCY_ESCALATION_TRIGGERED', 'critical',
      { userId: escalatedBy, role: 'provider' },
      { type: 'consultation', id: consultationId },
      `ESCALATE_${severity.toUpperCase()}`, 'success',
      { reason, escalationId }
    );

    return escalationId;
  }

  async notifyOnCallProviders(departmentId, escalationId, reason, severity) {
    const response = await fetch(`/api/on-call/${departmentId}`);
    const { providers } = await response.json();

    for (const provider of providers) {
      await this.pubnub.publish({
        channel: `notification.${provider.id}`,
        message: {
          type: 'EMERGENCY_ESCALATION',
          title: `URGENT: ${severity.toUpperCase()} Escalation`,
          body: reason, escalationId, priority: 'critical'
        },
        meta: {
          pn_apns: { aps: { alert: { title: 'EMERGENCY', body: reason }, sound: 'emergency.aiff' } },
          pn_gcm: { notification: { title: 'EMERGENCY', body: reason, priority: 'high' } }
        }
      });
    }
  }
}
```

## Message Retention

Do not copy jurisdictional retention-year tables here. Configure Message Persistence per your policy ([retention-and-storage.md](../../pubnub-history/references/retention-and-storage.md)).

PubNub-specific deltas:

- Video signaling: **`storeInHistory: false`**
- Audit channel: **`storeInHistory: true`** plus After-Publish persist
- Encounter teardown may `deleteMessages` on consultation / `.files` / `.video` when your policy requires it

```javascript
async function deleteConsultationMessages(pubnubAdmin, consultationId) {
  const channels = [consultationId, `${consultationId}.files`, `${consultationId}.video`];
  const results = [];
  for (const channel of channels) {
    try {
      await pubnubAdmin.deleteMessages({ channel });
      results.push({ channel, status: 'deleted' });
    } catch (error) {
      results.push({ channel, status: 'error', message: error.message });
    }
  }
  return results;
}
```

## Best Practices

- Refresh encounter tokens before TTL expiry rather than relying on long-lived grants
- Disable signaling message history (`storeInHistory: false`)
- Persist audit logs to durable storage; never include raw PHI in audit entries
- Grant minimum necessary permissions when additional providers join; log invite/remove

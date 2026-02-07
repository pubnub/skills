# PubNub Telemedicine Patterns

This reference covers architectural patterns for consultation workflows, WebRTC video signaling, audit logging, multi-provider consultations, emergency escalation, and message retention policies.

## Consultation Workflow

A telemedicine consultation follows a defined lifecycle from patient check-in through follow-up.

### Consultation States

| State | Description | Active Channels | Duration |
|-------|-------------|----------------|----------|
| `scheduled` | Appointment confirmed | `notification.{patientId}` | Until check-in |
| `checked-in` | Patient confirmed attendance | `waiting-room.{providerId}` | 0-5 minutes |
| `waiting` | Patient in virtual waiting room | `waiting-room.{providerId}` | Variable |
| `connecting` | Establishing session | `consultation.{providerId}.{patientId}` | 10-30 seconds |
| `in-progress` | Active consultation | `consultation.*`, `.video`, `.files` | 15-60 minutes |
| `wrapping-up` | Provider completing notes | `consultation.{providerId}.{patientId}` | 2-10 minutes |
| `completed` | Consultation finished | None (channels cleaned up) | Terminal |

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

PubNub serves as the signaling layer for WebRTC video consultations. All offer/answer exchanges and ICE candidate transfers flow through encrypted PubNub channels.

### Signaling Channel Architecture

The video signaling channel follows the pattern `consultation.{providerId}.{patientId}.video`. This channel carries only signaling metadata, never media streams.

### Video Signaling Implementation

```javascript
class TelemedicineVideoSignaling {
  constructor(pubnub, consultationId, localUserId) {
    this.pubnub = pubnub;
    this.signalingChannel = `${consultationId}.video`;
    this.localUserId = localUserId;
    this.peerConnection = null;
  }

  async initialize(onRemoteStream) {
    this.peerConnection = new RTCPeerConnection({
      iceServers: [
        { urls: 'stun:stun.l.google.com:19302' },
        {
          urls: 'turn:your-turn-server.example.com:3478',
          username: 'telemedicine',
          credential: await this.getTurnCredential()
        }
      ],
      iceTransportPolicy: 'relay' // Force TURN relay for HIPAA compliance
    });

    this.peerConnection.onicecandidate = (event) => {
      if (event.candidate) {
        this.sendSignal({ type: 'ice-candidate', candidate: event.candidate.toJSON() });
      }
    };

    this.peerConnection.ontrack = (event) => onRemoteStream(event.streams[0]);

    this.pubnub.addListener({
      message: (event) => {
        if (event.channel === this.signalingChannel && event.publisher !== this.localUserId) {
          this.handleSignal(event.message);
        }
      }
    });

    this.pubnub.subscribe({ channels: [this.signalingChannel] });
  }

  async startCall(localStream) {
    localStream.getTracks().forEach(track => this.peerConnection.addTrack(track, localStream));
    const offer = await this.peerConnection.createOffer({
      offerToReceiveAudio: true, offerToReceiveVideo: true
    });
    await this.peerConnection.setLocalDescription(offer);
    await this.sendSignal({ type: 'offer', sdp: offer.sdp });
  }

  async handleSignal(signal) {
    switch (signal.type) {
      case 'offer':
        await this.peerConnection.setRemoteDescription(
          new RTCSessionDescription({ type: 'offer', sdp: signal.sdp })
        );
        const answer = await this.peerConnection.createAnswer();
        await this.peerConnection.setLocalDescription(answer);
        await this.sendSignal({ type: 'answer', sdp: answer.sdp });
        break;
      case 'answer':
        await this.peerConnection.setRemoteDescription(
          new RTCSessionDescription({ type: 'answer', sdp: signal.sdp })
        );
        break;
      case 'ice-candidate':
        if (signal.candidate) {
          await this.peerConnection.addIceCandidate(new RTCIceCandidate(signal.candidate));
        }
        break;
      case 'end-call':
        this.endCall();
        break;
    }
  }

  async sendSignal(signal) {
    await this.pubnub.publish({
      channel: this.signalingChannel,
      message: { ...signal, senderId: this.localUserId, timestamp: new Date().toISOString() },
      storeInHistory: false // Signaling messages do not need persistence
    });
  }

  async endCall() {
    await this.sendSignal({ type: 'end-call' });
    if (this.peerConnection) { this.peerConnection.close(); this.peerConnection = null; }
    this.pubnub.unsubscribe({ channels: [this.signalingChannel] });
  }

  async getTurnCredential() {
    const response = await fetch('/api/telemedicine/turn-credentials', {
      method: 'POST', headers: { 'Authorization': `Bearer ${authToken}` }
    });
    return (await response.json()).credential;
  }
}
```

### Video Signaling Flow

| Step | Initiator | Signal Type | Direction |
|------|-----------|-------------|-----------|
| 1 | Provider | `offer` | Provider to Patient |
| 2 | Patient | `answer` | Patient to Provider |
| 3 | Both | `ice-candidate` | Bidirectional (multiple) |
| 4 | Either | `end-call` | Bidirectional |

## Audit Logging for Compliance

Every action involving PHI must be logged for compliance audits and breach investigations.

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

Some cases require multiple providers (e.g., primary care consulting with a specialist).

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
        role, // 'specialist' | 'second-opinion' | 'observer'
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

Emergency escalation allows any participant to flag an urgent situation requiring immediate attention.

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
        severity, // 'urgent' | 'critical' | 'life-threatening'
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

## Message Retention and Deletion Policies

HIPAA and organizational policies dictate message retention and deletion schedules.

### Retention Policy Configuration

| Channel Type | Retention Period | Rationale |
|-------------|-----------------|-----------|
| Consultation messages | 7 years | Medical record retention requirement |
| Video signaling | 0 (no retention) | No clinical value |
| Notifications | 90 days | Operational reference |
| Audit logs | 7 years minimum | Compliance audit trail |
| Emergency escalations | 7 years | Incident documentation |

### Implementing Retention Policies

```javascript
class MessageRetentionManager {
  constructor(pubnubAdmin) {
    this.pubnubAdmin = pubnubAdmin;
  }

  async enforceRetention(channelId, retentionDays) {
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - retentionDays);
    const cutoffTimetoken = (cutoffDate.getTime() * 10000).toString();

    try {
      await this.pubnubAdmin.deleteMessages({
        channel: channelId, start: '1', end: cutoffTimetoken
      });
      return { channel: channelId, deletedBefore: cutoffDate.toISOString() };
    } catch (error) {
      console.error(`Retention enforcement failed for ${channelId}:`, error.message);
      throw error;
    }
  }

  async deleteConsultationMessages(consultationId) {
    const channels = [consultationId, `${consultationId}.files`, `${consultationId}.video`];
    const results = [];
    for (const channel of channels) {
      try {
        await this.pubnubAdmin.deleteMessages({ channel });
        results.push({ channel, status: 'deleted' });
      } catch (error) {
        results.push({ channel, status: 'error', message: error.message });
      }
    }
    return results;
  }
}
```

## Best Practices

### Consultation Workflow

- Implement a pre-consultation device check (camera, microphone, internet speed) during check-in to avoid technical issues
- Set automatic session timeouts that warn participants 5 minutes before token expiry and refresh tokens seamlessly
- Design the waiting room with estimated wait times and queue position updates to reduce patient anxiety
- Always end consultations gracefully with a summary event capturing duration and follow-up requirements

### Video Signaling

- Force TURN relay mode (`iceTransportPolicy: 'relay'`) to ensure media flows through controlled infrastructure
- Use TLS-enabled TURN servers on port 443 to avoid firewall issues on hospital and home networks
- Implement connection quality monitoring with visible indicators for both patient and provider
- Disable signaling message history (`storeInHistory: false`) since signaling data has no clinical retention value

### Audit Logging

- Persist audit logs to immutable storage (append-only database or write-once object store)
- Include enough context in each event to reconstruct the sequence of actions without querying other systems
- Set up real-time alerts on critical audit events (escalations, access failures, unusual patterns)
- Never include raw PHI in audit entries -- use references (patient ID, consultation ID) resolvable through authorized queries

### Multi-Provider Sessions

- Clearly communicate roles when additional providers join so the patient understands who is present
- Grant minimum necessary permissions for each provider role
- Log all provider additions and removals as audit events

### Emergency Escalation

- Test escalation workflows regularly with simulated scenarios to ensure notifications reach on-call staff promptly
- Escalation notifications must bypass do-not-disturb settings on mobile devices
- Always require resolution documentation for every escalation to maintain the incident record
- Design escalation as a one-tap action for the provider to minimize delay in genuine emergencies

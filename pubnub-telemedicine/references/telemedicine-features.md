# PubNub Telemedicine Features

This reference covers core telemedicine features built on PubNub: patient queue management, real-time notifications, provider availability tracking, consent management, and secure file sharing.

## Patient Queue Management

The patient queue (virtual waiting room) coordinates telemedicine appointments, tracking patients from check-in through consultation completion.

### Queue Data Model

```javascript
const queueEntry = {
  id: crypto.randomUUID(),
  patientId: 'patient-abc123',
  appointmentId: 'appt-789',
  providerId: 'dr-smith',
  departmentId: 'cardiology',
  status: 'checked-in', // checked-in | waiting | in-consultation | completed | no-show
  priority: 'normal',   // emergency | urgent | normal | follow-up
  checkInTime: new Date().toISOString(),
  estimatedWaitMinutes: 15,
  metadata: {
    consentGiven: true,
    consentTimestamp: new Date().toISOString(),
    deviceReady: true,
    interpreterNeeded: false
  }
};
```

### Queue Manager Implementation

```javascript
class PatientQueueManager {
  constructor(pubnub, departmentId) {
    this.pubnub = pubnub;
    this.departmentId = departmentId;
    this.queueChannel = `queue.${departmentId}`;
  }

  async addPatient(queueEntry) {
    await this.pubnub.publish({
      channel: this.queueChannel,
      message: {
        action: 'PATIENT_ADDED',
        entry: queueEntry,
        timestamp: new Date().toISOString()
      },
      storeInHistory: true
    });

    const currentQueue = await this.getQueue();
    currentQueue.push(queueEntry);
    await this.updateQueueState(currentQueue);
    return queueEntry.id;
  }

  async updatePatientStatus(queueEntryId, newStatus, updatedBy) {
    await this.pubnub.publish({
      channel: this.queueChannel,
      message: {
        action: 'STATUS_UPDATED',
        queueEntryId,
        newStatus,
        updatedBy,
        timestamp: new Date().toISOString()
      },
      storeInHistory: true
    });
  }

  async removePatient(queueEntryId, reason) {
    await this.pubnub.publish({
      channel: this.queueChannel,
      message: {
        action: 'PATIENT_REMOVED',
        queueEntryId,
        reason, // 'completed' | 'no-show' | 'cancelled' | 'rescheduled'
        timestamp: new Date().toISOString()
      },
      storeInHistory: true
    });
  }

  async getQueue() {
    try {
      const response = await this.pubnub.getChannelMetadata({ channel: this.queueChannel });
      return JSON.parse(response.data.custom?.queueState || '[]');
    } catch (error) {
      return [];
    }
  }

  async updateQueueState(queue) {
    await this.pubnub.setChannelMetadata({
      channel: this.queueChannel,
      data: {
        name: `Queue - ${this.departmentId}`,
        custom: {
          queueState: JSON.stringify(queue),
          lastUpdated: new Date().toISOString()
        }
      }
    });
  }
}
```

### Queue Status Transitions

| Current Status | Valid Transitions | Triggered By |
|---------------|-------------------|--------------|
| `checked-in` | `waiting`, `cancelled` | Patient completes device check / Patient cancels |
| `waiting` | `in-consultation`, `no-show`, `cancelled` | Provider starts session / Timeout / Patient cancels |
| `in-consultation` | `completed`, `waiting` | Provider ends session / Provider returns to queue |
| `completed` | (terminal) | Consultation finished |
| `no-show` | `waiting` | Patient returns within grace period |
| `cancelled` | `checked-in` | Patient reschedules same day |

## Real-Time Notifications

Telemedicine applications require multiple notification types delivered instantly to the right users.

### Notification Types

| Notification Type | Recipient | Channel Pattern | Priority |
|------------------|-----------|----------------|----------|
| Appointment Reminder | Patient | `notification.{patientId}` | Normal |
| Provider Ready | Patient | `notification.{patientId}` | High |
| Patient Waiting | Provider | `notification.{providerId}` | Normal |
| Lab Results Available | Patient | `notification.{patientId}` | Normal |
| Emergency Escalation | Provider | `emergency.{departmentId}` | Critical |
| Schedule Change | Provider | `notification.{providerId}` | High |

### Notification Service

```javascript
class TelemedicineNotificationService {
  constructor(pubnub, auditLogger) {
    this.pubnub = pubnub;
    this.auditLogger = auditLogger;
  }

  async sendNotification(recipientId, notification) {
    const channel = `notification.${recipientId}`;
    const payload = {
      id: crypto.randomUUID(),
      type: notification.type,
      title: notification.title,
      body: notification.body,
      priority: notification.priority || 'normal',
      actionUrl: notification.actionUrl || null,
      timestamp: new Date().toISOString(),
      metadata: {
        appointmentId: notification.appointmentId,
        senderId: notification.senderId
      }
    };

    await this.pubnub.publish({
      channel,
      message: payload,
      storeInHistory: true,
      meta: {
        pn_apns: {
          aps: {
            alert: { title: payload.title, body: payload.body },
            sound: payload.priority === 'critical' ? 'emergency.aiff' : 'default'
          }
        },
        pn_gcm: {
          notification: {
            title: payload.title,
            body: payload.body,
            priority: payload.priority === 'critical' ? 'high' : 'normal'
          }
        }
      }
    });

    await this.auditLogger.logEvent('NOTIFICATION_SENT', {
      userId: notification.senderId,
      userRole: 'system',
      resourceType: 'notification',
      resourceId: payload.id,
      channel,
      action: `NOTIFY_${notification.type}`
    });

    return payload.id;
  }

  async scheduleReminder(appointmentId, patientId, appointmentTime, minutesBefore = 15) {
    const reminderTime = new Date(appointmentTime);
    reminderTime.setMinutes(reminderTime.getMinutes() - minutesBefore);

    return {
      appointmentId,
      patientId,
      scheduledFor: reminderTime.toISOString(),
      notification: {
        type: 'APPOINTMENT_REMINDER',
        title: 'Upcoming Appointment',
        body: `Your telemedicine appointment starts in ${minutesBefore} minutes.`,
        priority: 'high',
        appointmentId,
        senderId: 'system',
        actionUrl: `/appointments/${appointmentId}/join`
      }
    };
  }
}
```

## Provider Availability Tracking

PubNub Presence tracks provider online/offline status in real time.

### Provider Status Values

| Status | Meaning | Visible to Patients |
|--------|---------|-------------------|
| `available` | Ready to see patients | Yes |
| `in-consultation` | Currently in a session | Yes (as "Busy") |
| `away` | Temporarily unavailable | Yes (as "Away") |
| `offline` | Not connected | Yes (as "Offline") |

### Presence-Based Availability

```javascript
class ProviderAvailabilityTracker {
  constructor(pubnub, departmentId) {
    this.pubnub = pubnub;
    this.presenceChannel = `presence.providers.${departmentId}`;
  }

  async setProviderStatus(providerId, status, details = {}) {
    await this.pubnub.setState({
      channels: [this.presenceChannel],
      state: {
        status,
        displayName: details.displayName,
        specialty: details.specialty,
        currentPatientCount: details.currentPatientCount || 0,
        maxPatients: details.maxPatients || 1,
        lastStatusChange: new Date().toISOString()
      }
    });
  }

  async getAvailableProviders() {
    try {
      const response = await this.pubnub.hereNow({
        channels: [this.presenceChannel],
        includeState: true,
        includeUUIDs: true
      });
      const occupants = response.channels[this.presenceChannel]?.occupants || [];
      return occupants
        .filter(o => o.state?.status === 'available')
        .map(o => ({
          providerId: o.uuid,
          displayName: o.state.displayName,
          specialty: o.state.specialty
        }));
    } catch (error) {
      console.error('Failed to fetch provider availability:', error.message);
      return [];
    }
  }

  subscribeToAvailabilityChanges(onProviderChange) {
    this.pubnub.addListener({
      presence: (event) => {
        if (event.channel !== this.presenceChannel) return;
        const type = event.action === 'state-change' ? 'statusChange'
          : (event.action === 'join' ? 'online' : 'offline');
        onProviderChange({ type, providerId: event.uuid, state: event.state });
      }
    });
    this.pubnub.subscribe({ channels: [this.presenceChannel], withPresence: true });
  }
}
```

## Patient Consent Management

HIPAA requires documented patient consent before sharing PHI.

```javascript
class ConsentManager {
  constructor(pubnub, auditLogger) {
    this.pubnub = pubnub;
    this.auditLogger = auditLogger;
  }

  async recordConsent(patientId, consentDetails) {
    const consentRecord = {
      id: crypto.randomUUID(),
      patientId,
      type: consentDetails.type, // 'telehealth' | 'data-sharing' | 'recording'
      granted: true,
      scope: consentDetails.scope,
      grantedAt: new Date().toISOString(),
      expiresAt: consentDetails.expiresAt || null,
      version: consentDetails.consentFormVersion
    };

    await this.pubnub.setUUIDMetadata({
      uuid: patientId,
      data: {
        custom: { [`consent_${consentDetails.type}`]: JSON.stringify(consentRecord) }
      }
    });

    await this.auditLogger.logEvent('PATIENT_CONSENT_GIVEN', {
      userId: patientId,
      userRole: 'patient',
      resourceType: 'consent',
      resourceId: consentRecord.id,
      action: `CONSENT_GRANTED_${consentDetails.type}`,
      consentRef: consentRecord.id
    });

    return consentRecord;
  }

  async verifyConsent(patientId, consentType) {
    try {
      const response = await this.pubnub.getUUIDMetadata({ uuid: patientId });
      const consentData = response.data.custom?.[`consent_${consentType}`];
      if (!consentData) return false;
      const consent = JSON.parse(consentData);
      if (!consent.granted) return false;
      if (consent.expiresAt && new Date(consent.expiresAt) < new Date()) return false;
      return true;
    } catch (error) {
      return false;
    }
  }

  async revokeConsent(patientId, consentType) {
    await this.pubnub.setUUIDMetadata({
      uuid: patientId,
      data: {
        custom: { [`consent_${consentType}`]: JSON.stringify({ granted: false, revokedAt: new Date().toISOString() }) }
      }
    });
    await this.auditLogger.logEvent('PATIENT_CONSENT_REVOKED', {
      userId: patientId, userRole: 'patient', resourceType: 'consent',
      resourceId: `${patientId}-${consentType}`, action: `CONSENT_REVOKED_${consentType}`
    });
  }
}
```

## Secure File Sharing

Telemedicine consultations often require sharing lab results, images, and prescriptions.

```javascript
class SecureFileSharing {
  constructor(pubnub, auditLogger) {
    this.pubnub = pubnub;
    this.auditLogger = auditLogger;
  }

  async shareFile(channelId, file, senderDetails) {
    const fileChannel = `${channelId}.files`;
    const allowedTypes = ['application/pdf', 'image/jpeg', 'image/png', 'image/dicom', 'text/plain'];
    const maxSizeBytes = 25 * 1024 * 1024;

    if (!allowedTypes.includes(file.type)) {
      throw new Error(`File type not allowed: ${file.type}`);
    }
    if (file.size > maxSizeBytes) {
      throw new Error(`File exceeds maximum size of 25 MB`);
    }

    try {
      const result = await this.pubnub.sendFile({
        channel: fileChannel,
        file: { data: file.data, name: file.name, mimeType: file.type },
        message: {
          type: 'FILE_SHARED',
          category: file.category, // 'lab-result' | 'prescription' | 'imaging'
          sender: { id: senderDetails.id, role: senderDetails.role },
          timestamp: new Date().toISOString()
        },
        storeInHistory: true
      });

      await this.auditLogger.logEvent('PHI_FILE_SHARED', {
        userId: senderDetails.id, userRole: senderDetails.role,
        resourceType: 'file', resourceId: result.id,
        channel: fileChannel, action: 'FILE_UPLOAD'
      });

      return { fileId: result.id, fileName: file.name };
    } catch (error) {
      throw new Error(`File sharing failed: ${error.message}`);
    }
  }
}
```

## Best Practices

### Queue Management

- Implement automatic no-show detection with a configurable timeout (typically 10-15 minutes past appointment time)
- Send progressive notifications as wait times change: initial estimate, updated estimate at 50% wait, and a "provider ready" alert
- Support priority override for urgent cases that need to bypass the standard queue order
- Persist queue state to your database in addition to PubNub channel metadata for recovery after outages

### Notifications

- Use PubNub Mobile Push for appointment reminders so notifications reach patients even when the app is backgrounded
- Include actionable deep links in notifications so patients can join consultations with a single tap
- Implement notification deduplication to prevent repeated alerts when patients have multiple devices

### Provider Availability

- Combine Presence events with manual status updates for accurate availability -- Presence detects connectivity, manual status reflects clinical availability
- Implement automatic status transitions: set to `in-consultation` when a session starts and back to `available` when it ends
- Track provider capacity (concurrent patients) to prevent overloading

### Consent and Compliance

- Always verify consent before establishing a consultation channel -- block channel access if consent is not recorded
- Record consent events immutably: never delete consent records, only append revocation records
- Support multiple consent types independently (telehealth, data sharing, recording)

### File Sharing

- Restrict file types to clinical document formats and common image types -- block executable files
- Scan uploaded files for malware before making them available to recipients
- Implement automatic file expiry for sensitive documents like prescriptions

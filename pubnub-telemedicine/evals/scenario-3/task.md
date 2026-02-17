# Compliance Audit Trail and Retention Policy System

## Problem/Feature Description

VitalCare Health is preparing for a HIPAA compliance audit of their PubNub-based telemedicine platform. The compliance officer has identified that the platform currently lacks a structured audit logging system and has no documented or enforced message retention policies. Every action involving patient data -- messages sent, files accessed, tokens granted, consent given or revoked, video sessions, and emergency escalations -- must be logged in a way that can be reviewed by auditors.

The engineering team needs to build an audit logging service that publishes structured audit events through PubNub and also persists them to an external durable storage endpoint for long-term retention. The audit events must not contain actual patient health data -- only identifiers and references that can be used to look up details through authorized systems. Additionally, the team needs a retention policy manager that can enforce different retention periods for different types of channels, since clinical messages, transient signaling data, and operational notifications have very different retention requirements.

## Output Specification

Create the following files in a `vitalcare/` directory:

1. `vitalcare/compliance/AuditLogger.js` -- A JavaScript class that publishes structured audit events to PubNub and persists them to an external API endpoint. The class should support logging various healthcare event types (message sent, file shared, token granted, consent actions, session events, escalations). Each audit entry should be a structured object with appropriate fields.

2. `vitalcare/compliance/RetentionManager.js` -- A JavaScript class that enforces message retention policies by deleting messages older than the configured retention period for different channel types. It should support different retention periods for different categories of channels.

3. `vitalcare/compliance/retentionConfig.json` -- A JSON configuration file that specifies the retention periods (in days or years) for each category of channel used in the telemedicine platform (e.g., consultation messages, signaling data, notifications, audit logs, emergency records).

4. `vitalcare/compliance/AUDIT_EVENTS.md` -- A document listing all audit event types the system supports, with a brief description of when each event is logged.

# Secure File Sharing and Emergency Escalation for Consultations

## Problem/Feature Description

PulseHealth is enhancing their telemedicine consultation platform with two critical features that providers have been requesting.

First, providers and patients need to exchange clinical documents during consultations -- lab results, prescriptions, and medical images. The platform needs a secure file sharing module that validates uploaded files, sends them through PubNub, and logs the sharing events for compliance. Only clinically appropriate file formats should be accepted, and there must be a reasonable file size limit to prevent abuse.

Second, the medical director wants an emergency escalation system. During any consultation, a provider should be able to flag an urgent situation that requires immediate backup from on-call staff. The escalation must fan out notifications to the appropriate department's on-call providers with high-priority mobile alerts that will break through notification silencing on phones. The escalation events must be logged for incident review.

Both features must integrate with the existing PubNub infrastructure and audit logging.

## Output Specification

Create the following files in a `pulsehealth/` directory:

1. `pulsehealth/features/SecureFileSharing.js` -- A JavaScript class for sharing clinical files during consultations through PubNub. It should validate file types and sizes, send files via PubNub, and log the event for audit purposes.

2. `pulsehealth/features/EmergencyEscalation.js` -- A JavaScript class that handles emergency escalation during consultations. It should publish the escalation event, notify on-call providers with mobile push notifications, and log the escalation for compliance.

3. `pulsehealth/features/NotificationPayloads.js` -- A JavaScript module that constructs notification payloads for different platforms (iOS and Android) used by both file sharing alerts and emergency escalation alerts. Critical alerts should use distinctive sounds to stand out.

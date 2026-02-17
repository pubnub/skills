# PubNub Authentication Server for Healthcare Messaging

## Problem/Feature Description

A healthcare startup is building a real-time messaging platform where doctors and patients communicate through private channels. The backend team needs a Node.js Express server that handles PubNub authentication -- when a user logs in and requests messaging access, the server should verify their identity and issue them the appropriate PubNub credentials to access their authorized channels.

The system has three types of users: patients (who can only read their own consultation channels), doctors (who can read and write to their assigned patient channels), and admins (who have full access to moderation channels). Patient consultations involve sensitive medical data, so those channels require tighter security controls than general notification channels.

The team has already set up their PubNub keyset in the Admin Portal with Access Manager enabled. They need you to build the authentication endpoint that issues the right credentials based on user role and returns them to the calling client application.

## Output Specification

Create a file called `server.js` containing:
- A complete Node.js/Express server with a POST `/api/pubnub/auth` endpoint
- The endpoint should accept a JSON body with `userId` and `role` (one of "patient", "doctor", "admin")
- Based on the role, determine which channels the user can access and with what permissions
- Issue PubNub credentials and return them as JSON including the token/auth info, the subscribe key, and an expiration timestamp
- Patient channels should follow the pattern `consultation-{userId}`
- Doctor channels follow `consultation-{patientId}` for their assigned patients (assume a helper function `getAssignedPatients(userId)` exists)
- Admin channels include `mod-alerts` and `system-logs`
- All users should also have access to a `notifications` channel (read-only)

Also create a `package.json` with the necessary dependencies.

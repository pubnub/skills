# Telemedicine Platform Backend Setup

## Problem/Feature Description

A healthcare startup called MedConnect is building a telemedicine platform that allows patients to have real-time text consultations with doctors and nurses. The platform must comply with HIPAA regulations since it will transmit protected health information (PHI) between patients and healthcare providers.

The engineering team needs the foundational server-side and client-side infrastructure code for secure real-time messaging. The server component handles authorization -- issuing scoped tokens so that each user (doctor, nurse, or patient) can only access channels they are authorized for. The client component initializes the messaging SDK for end users. The platform needs to support three roles: providers (doctors), nurses, and patients -- each with different levels of access to consultation channels.

The team has chosen PubNub as their real-time messaging provider but has not yet written any code. They need a production-ready setup that a compliance officer would approve.

## Output Specification

Create the following files in a `medconnect/` directory:

1. `medconnect/server/tokenService.js` -- A Node.js module that exports a function to initialize PubNub on the server and a function to issue authorization tokens for a given user based on their healthcare role and consultation context. The token service should handle at least provider, patient, and nurse roles with appropriate permissions and token lifetimes.

2. `medconnect/client/initPubnub.js` -- A JavaScript module that initializes PubNub for a client-side user, configured for a healthcare production environment where sensitive patient data is exchanged.

3. `medconnect/ARCHITECTURE.md` -- A brief document describing the channel naming strategy the platform will use for consultations, waiting rooms, department queues, notifications, and any other channel types needed for a telemedicine workflow. Include example channel names.

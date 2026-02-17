# Virtual Waiting Room and Patient Check-In System

## Problem/Feature Description

CareFirst Clinic is migrating from an in-person waiting room to a virtual waiting room system for their telemedicine appointments. Today, patients call in and a receptionist manually tracks who is waiting. The clinic wants a real-time system where patients check in online, are placed in a queue, and providers can see who is waiting and pull the next patient into a consultation.

The clinic has three departments (cardiology, dermatology, general-practice) and multiple providers per department. Patients should be able to check in for their appointment, see their position and estimated wait, and be notified when their provider is ready. Providers need to see the queue for their department and update patient statuses as they move through the workflow. The system must verify that a patient has given appropriate consent before they can enter the waiting room.

The clinic already has PubNub set up and needs the queue management and check-in logic implemented.

## Output Specification

Create the following files in a `carefirst/` directory:

1. `carefirst/queue/PatientQueueManager.js` -- A JavaScript class that manages the patient queue for a department. It should support adding patients, updating their status through the workflow, removing them, and persisting/retrieving queue state.

2. `carefirst/consultation/CheckInFlow.js` -- A JavaScript module that orchestrates the patient check-in process: verifying consent, adding the patient to the queue, notifying the assigned provider, and returning the consultation context.

3. `carefirst/models/queueEntry.js` -- A module that defines or documents the data model for a queue entry, including all the fields a queue entry should contain and their valid values.

# Video Consultation Signaling Module

## Problem/Feature Description

HealthBridge Telehealth currently offers text-based consultations between patients and providers using PubNub. They now want to add video calling capability so providers can conduct face-to-face virtual appointments. The video calls will use WebRTC for peer-to-peer media, and PubNub will serve as the signaling layer to coordinate the call setup between participants.

The engineering team needs a signaling module that handles the full video call lifecycle: one participant initiates the call with an offer, the other responds with an answer, ICE candidates are exchanged for network traversal, and either party can end the call. The implementation must meet healthcare compliance requirements -- media should not flow through arbitrary internet paths, and the signaling data itself should be handled appropriately given it has no long-term clinical value.

The module should also support notifying a patient that their provider is ready to start the video call, since patients may be in a virtual waiting room at the time.

## Output Specification

Create the following files in a `healthbridge/` directory:

1. `healthbridge/video/VideoSignaling.js` -- A JavaScript class that manages WebRTC signaling through PubNub for a telemedicine video call. It should handle initializing the peer connection, creating and handling offers/answers, exchanging ICE candidates, and ending calls. The class should receive PubNub and consultation context as constructor parameters.

2. `healthbridge/video/VideoCallManager.js` -- A JavaScript module that orchestrates the higher-level video call workflow: notifying the patient that the provider is ready, initiating the signaling, and cleaning up when the call ends. Include audit logging calls for video session start and end events.

3. `healthbridge/video/README.md` -- A brief description of the signaling architecture, including which PubNub channels are used for signaling versus messaging, and how the module fits into the overall consultation flow.

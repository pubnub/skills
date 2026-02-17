# Driver Location Broadcasting Module

## Problem/Feature Description

A food delivery startup is building the driver-side component of their mobile app. Drivers need to continuously broadcast their GPS position while on active deliveries so that customers and fleet managers can track them in real time. The startup's biggest concern is battery life -- drivers work 8-hour shifts and their phones need to last the whole day. They also need to be mindful of their PubNub message quota since they are on a limited plan.

The engineering team needs a JavaScript/Node.js module that handles location publishing from the driver app. The module should initialize the PubNub SDK for a driver, manage GPS position publishing with smart frequency control based on driving conditions, and include distance calculation utilities for determining proximity to destinations. The solution must balance tracking accuracy with resource efficiency.

## Output Specification

Produce the following files:

1. `driver-location-publisher.js` -- The main module containing the PubNub initialization for a driver, the adaptive location publishing class, and the distance calculation utility function.
2. `driver-location-publisher.test.js` -- Unit tests that verify the publishing frequency adapts correctly to different driving speeds and conditions, and that stationary drivers do not generate excessive messages.

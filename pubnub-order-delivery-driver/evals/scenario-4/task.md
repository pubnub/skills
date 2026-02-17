# Delivery Communication and Confirmation System

## Problem/Feature Description

A meal delivery platform wants to improve their delivery completion rate by adding real-time communication between drivers and customers and implementing delivery confirmation workflows. Currently, when a driver cannot find a customer's apartment, they call the customer's personal phone number, which raises privacy concerns and does not work when customers have do-not-disturb enabled. The company also has a problem with disputed deliveries where customers claim they never received their food.

The team needs to build three interconnected features: (1) an in-app chat system scoped to each delivery so drivers and customers can communicate without sharing contact information, with pre-written quick responses for common driver situations; (2) mobile push notifications that alert customers of key delivery milestones even when the app is in the background; and (3) a delivery confirmation workflow that captures proof (photo, PIN, or signature) when the driver hands off the order. Finally, all resources associated with a delivery should be properly cleaned up once it reaches a final state.

## Output Specification

Produce the following files:

1. `delivery-chat.js` -- The chat module that enables real-time messaging between driver and customer for a specific order, including message history loading and pre-built response templates for drivers.
2. `push-notifications.js` -- The module for registering devices for push notifications on order channels and publishing status updates with embedded push payloads for both iOS and Android.
3. `proof-of-delivery.js` -- Functions for capturing and publishing delivery proof (photo, PIN verification, signature) alongside the delivery completion status.
4. `delivery-cleanup.js` -- A module that handles post-delivery cleanup: unsubscribing clients, removing channels from channel groups, and revoking access tokens.

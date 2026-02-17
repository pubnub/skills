# Dispatch and Delivery Failure Management System

## Problem/Feature Description

A courier service is building out their dispatch backend that assigns incoming delivery orders to available drivers. Their current manual dispatch process is too slow -- by the time a dispatcher picks a driver, a closer one may have become available. They need an automated system that finds the nearest available driver for each new order based on live GPS positions and assigns them.

The operations team also needs the system to handle delivery failures gracefully. Drivers sometimes cannot complete a delivery (customer not home, wrong address, etc.) and the order needs to be reassigned to another driver. They have observed that some problematic addresses cause repeated failures, so the system should cap the number of reassignment attempts to prevent an order from bouncing between drivers indefinitely. Additionally, the system should use location-based triggers to automatically detect when a driver arrives at the pickup merchant or gets close to the customer's address.

## Output Specification

Produce the following files:

1. `dispatch-system.js` -- The dispatch engine that tracks available drivers via their real-time positions, finds the nearest driver for a new order, sends assignment commands, and handles the assignment lifecycle.
2. `delivery-failure-handler.js` -- Logic for handling failed deliveries, notifying the dispatch system, freeing the failed driver, and reassigning to a new driver with retry limits.
3. `geofence-trigger.js` -- A PubNub Function that monitors driver location publishes and automatically triggers status transitions when a driver enters the pickup or dropoff proximity zone.

# IoT Sensor Monitoring Platform

## Problem/Feature Description

A smart building management company operates sensors across 3 office buildings. Each building has multiple floors, and each floor has various sensor types (temperature, humidity, motion, air quality). The company needs a real-time monitoring backend built with PubNub (JavaScript SDK) that allows:

1. **Floor supervisors** to monitor all sensors on their floor
2. **Building managers** to monitor all sensors in their building
3. **The operations center** to monitor a custom selection of about 500 specific sensor channels across all buildings

The operations center team frequently changes which sensors they care about, adding and removing specific channels from their subscription set.

Sensor data is published from individual sensor devices to their specific channels following a hierarchical naming convention.

## Output Specification

Produce the following files:

- `channel-design.md`: A document describing the channel naming scheme, the subscription strategy for each user role (floor supervisor, building manager, operations center), and any PubNub features or add-ons that need to be enabled. Include any relevant limits or constraints that apply.

- `sensor-publisher.js`: A Node.js module that exports a function to publish a sensor reading, given building ID, floor number, sensor type, and the reading value. Include appropriate size validation for the messages.

- `floor-monitor.js`: A Node.js module that subscribes to all sensors on a given floor of a given building, with a listener that logs incoming sensor data.

- `ops-center.js`: A Node.js module that manages the operations center's subscriptions, with functions to add sensor channels, remove sensor channels, list current channels, and subscribe to the managed set.

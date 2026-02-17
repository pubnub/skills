# Real-Time Fleet Tracking Dashboard

## Problem/Feature Description

A logistics company tracks 2,000 delivery vehicles in real-time. Each vehicle sends location updates, speed, fuel level, engine status, driver status, and other telemetry data every 2 seconds. The data is sent via PubNub to a central dashboard. The current implementation is hitting performance issues: messages are too large, the publish rate is overwhelming certain channels, and the dashboard is sluggish because of the volume of data.

The engineering team needs a refactored PubNub message layer (JavaScript/Node.js) that dramatically reduces bandwidth while preserving all necessary data. The vehicles frequently report the same values (e.g., parked vehicles report identical location/speed for hours). Many telemetry fields are sometimes null (e.g., fuel level sensor is intermittently unavailable).

## Output Specification

Produce the following files:

- `message-optimizer.js`: A Node.js module that exports:
  - A function that takes a raw verbose vehicle telemetry object and produces an optimized compact message ready for PubNub publishing
  - A function that takes a previous state and current state and produces a delta message containing only changed fields
  - A function to decode/expand a compact message back to its verbose form
  - A function to estimate the size of a message before publishing and warn if it approaches the limit

- `fleet-publisher.js`: A Node.js module that exports:
  - A function to publish a single vehicle update to its channel
  - A function to publish a batch of updates from multiple vehicles that are in the same geographic zone, combining them into fewer messages
  - The module should include comments explaining the target publish rate and how the batching strategy helps stay within it

- `dashboard-listener.js`: A Node.js module that sets up a PubNub listener for incoming fleet messages, handling the high volume of incoming data efficiently without blocking.

## Input Files

The following files are provided as inputs. Extract them before beginning.

=============== FILE: inputs/sample-telemetry.json ===============
{
  "vehicleIdentifier": "truck-4821",
  "currentLatitude": 37.7749,
  "currentLongitude": -122.4194,
  "currentSpeedMph": 35.5,
  "currentHeadingDegrees": 180,
  "fuelLevelPercentage": 72.3,
  "engineRunningStatus": "running",
  "driverActivityStatus": "driving",
  "cargoTemperatureFahrenheit": null,
  "tirePressureFrontLeft": null,
  "tirePressureFrontRight": null,
  "tirePressureRearLeft": null,
  "tirePressureRearRight": null,
  "lastMaintenanceTimestamp": 1704067200000,
  "currentOdometerMiles": 84521.3,
  "batteryVoltage": 12.6,
  "diagnosticTroubleCodesActive": []
}

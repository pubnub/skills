# IoT Sensor Monitoring Dashboard

## Problem/Feature Description

A smart building management company operates sensors across multiple buildings. Each building has multiple floors, and each floor has sensors for temperature, humidity, and air quality. The operations team wants a monitoring dashboard where a facility manager can subscribe to all sensors in a specific building at once, or drill down to a specific floor. They also want the ability to subscribe to all temperature readings across all buildings simultaneously.

The company currently has 3 buildings, each with 5 floors, and 3 sensor types per floor. The engineering team needs a JavaScript module that structures the channel hierarchy for these sensors, provides functions for publishing sensor readings, and allows flexible subscription patterns that let managers monitor at different granularity levels (all sensors in a building, all sensors on a floor, or a specific sensor type across all buildings).

## Output Specification

Produce a single JavaScript file named `iot-sensor-channels.js` that exports:

1. A function to generate the channel name for a specific sensor given a building ID, floor number, and sensor type
2. A function to subscribe to all sensors in a given building
3. A function to subscribe to all sensors on a specific floor of a building
4. A function to publish a sensor reading, accepting building ID, floor, sensor type, and the reading value
5. Brief inline comments explaining the channel hierarchy design choices

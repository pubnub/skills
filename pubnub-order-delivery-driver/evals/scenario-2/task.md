# Customer Delivery Tracking Page

## Problem/Feature Description

An online grocery delivery service wants to build a real-time tracking page that customers see after placing an order. The page should show the driver's location on a map, display the current order status, and show an estimated time of arrival. The product team has a firm requirement around driver privacy -- they do not want customers to see the driver's precise GPS coordinates when the driver is far away (e.g., picking up multiple orders across town). Location precision should increase as the driver gets closer.

The front-end team needs the JavaScript implementation for the customer-side tracking component. This includes subscribing to the right PubNub channels, handling incoming location and status messages, rendering the driver position smoothly on the map (avoiding jarring jumps between GPS updates), showing an ETA display, and implementing the location privacy filtering logic. The tracker should also handle the case where a customer opens the tracking page mid-delivery by fetching the most recent data.

## Output Specification

Produce the following files:

1. `delivery-tracker.js` -- The customer-facing tracking class that subscribes to PubNub channels, handles location and status events, renders driver position on a map with smooth updates, and displays ETA.
2. `privacy-filter.js` -- A PubNub Function (or module) that sanitizes driver location data based on distance to the customer, controlling precision levels before the data reaches the customer.
3. `eta-display.js` -- A module for calculating and formatting ETA based on incoming driver location data, including distance formatting utilities.

---
name: pubnub-order-delivery-driver
description: Build real-time order tracking and delivery driver systems with PubNub
license: PubNub
metadata:
  author: pubnub
  version: "0.2.0"
  domain: real-time
  triggers: pubnub, delivery, order tracking, driver location, dispatch, fleet, logistics, eta
  role: specialist
  scope: implementation
  output-format: code
---




# PubNub Order & Delivery Driver Specialist

You are a specialist in building real-time order tracking and delivery driver systems using PubNub. You help developers implement end-to-end delivery experiences including GPS location streaming, order status management, dispatch coordination, ETA calculations, and fleet visibility. You produce production-ready code that handles the full delivery lifecycle from order placement through proof of delivery.

> **Precedence:** PubNub MCP tools and pubnub.com/docs are authoritative for API shapes, limits, and configuration values. This skill is authoritative for patterns, sequencing, and design tradeoffs.


## When to Use This Skill

Invoke this skill when:

- Building a real-time delivery tracking page where customers watch their driver approach on a map
- Implementing GPS location streaming from driver mobile apps with battery-efficient updates
- Designing order status pipelines that transition through placed, confirmed, preparing, dispatched, en-route, and delivered states
- Creating dispatch systems that assign the nearest available driver to incoming orders
- Building fleet management dashboards with live positions and status for all active drivers
- Implementing driver-customer communication channels, ETA updates, and delivery confirmation flows

## Core Workflow

1. **Design Channel Architecture** -- Define the channel naming conventions for order tracking, driver locations, fleet management, and dispatch coordination so each concern is isolated and scalable.
2. **Implement Location Streaming** -- Set up GPS publishing from driver devices with adaptive frequency, battery optimization, and fallback strategies for poor connectivity.
3. **Build Order Status Pipeline** -- Create the state machine that governs order transitions, validates each change, and broadcasts updates to all interested subscribers.
4. **Configure Dispatch Logic** -- Implement driver assignment using proximity calculations, availability checks, and load balancing through PubNub Functions or your backend.
5. **Add Customer-Facing Tracking** -- Build the tracking page that subscribes to order and driver channels, renders the map, displays ETA, and shows status updates in real time.
6. **Handle Edge Cases** -- Implement reconnection logic, offline queueing, failed delivery flows, driver reassignment, and proof-of-delivery capture.

## Reference Guide

| Reference | Purpose |
|-----------|---------|
| [delivery-setup.md](references/delivery-setup.md) | Channel design, GPS publishing, SDK initialization, and tracking page setup |
| [delivery-status.md](references/delivery-status.md) | Order lifecycle states, ETA calculation, geofencing, push notifications, and status validation |
| [delivery-patterns.md](references/delivery-patterns.md) | Dispatch coordination, driver-customer chat, fleet dashboards, privacy controls, and proof of delivery |

## Key Implementation Requirements

### GPS Location Publishing

Driver apps must publish location updates to a dedicated driver channel. Use adaptive frequency -- publish more often when the driver is moving and less often when stationary.

```javascript
import PubNub from 'pubnub';

const pubnub = new PubNub({
  publishKey: 'pub-key',
  subscribeKey: 'sub-key',
  userId: 'driver-1234'
});

let lastPublishedLocation = null;

function publishDriverLocation(latitude, longitude, heading, speed) {
  const location = {
    lat: latitude,
    lng: longitude,
    heading: heading,
    speed: speed,
    timestamp: Date.now(),
    driverId: 'driver-1234'
  };

  // Adaptive publishing: skip if driver hasn't moved significantly
  if (lastPublishedLocation) {
    const distance = haversineDistance(lastPublishedLocation, location);
    if (distance < 5 && speed < 1) {
      return; // Skip publish if moved less than 5 meters and nearly stationary
    }
  }

  pubnub.publish({
    channel: 'driver.driver-1234.location',
    message: location
  });

  lastPublishedLocation = location;
}
```

### Order Tracking Channels

Each order gets its own channel for status updates. Customers subscribe to their order channel and the assigned driver's location channel.

```javascript
function subscribeToOrderTracking(orderId, driverId) {
  pubnub.subscribe({
    channels: [
      `order.${orderId}.status`,
      `driver.${driverId}.location`
    ]
  });

  pubnub.addListener({
    message: (event) => {
      if (event.channel.includes('.status')) {
        updateOrderStatusUI(event.message);
      } else if (event.channel.includes('.location')) {
        updateDriverMarkerOnMap(event.message);
        recalculateETA(event.message);
      }
    }
  });
}
```

### Status Updates with Validation

Publish order status transitions with metadata. Use PubNub Functions to validate that transitions follow the allowed state machine.

```javascript
async function updateOrderStatus(orderId, newStatus, metadata = {}) {
  const statusUpdate = {
    orderId: orderId,
    status: newStatus,
    timestamp: Date.now(),
    ...metadata
  };

  await pubnub.publish({
    channel: `order.${orderId}.status`,
    message: statusUpdate
  });

  // Also update the dispatch channel so fleet managers see the change
  await pubnub.publish({
    channel: 'dispatch.status-updates',
    message: statusUpdate
  });
}

// Example transitions
await updateOrderStatus('order-5678', 'dispatched', {
  driverId: 'driver-1234',
  estimatedDelivery: Date.now() + 25 * 60 * 1000
});
```

## Constraints

- Always use separate channels for location data and status updates to avoid mixing high-frequency GPS messages with critical state changes.
- Never expose raw driver GPS coordinates to customers until the driver is within a reasonable proximity of the delivery address.
- Implement message deduplication for status updates since network retries can cause duplicate publishes.
- Cap GPS publishing frequency at no more than once per second to avoid exceeding PubNub message quotas and draining driver device batteries.
- Use PubNub presence to track driver online/offline state rather than relying on periodic heartbeat messages in the data channel.
- Store order status history using PubNub message persistence so customers can view the full timeline even after reconnecting.

## MCP Tools

- **`get_sdk_documentation`** — pull SDK-specific publish/subscribe APIs (route via [intent-to-tool](../pubnub-choose-docs-path/references/intent-to-tool.md))
- **`manage_functions`** (`resource=package`, `operation=create`) — create the After-Publish geofence trigger / dispatch logic package
- **`grant_token`** — issue scoped grants per order (driver, customer, dispatcher)
- **`manage_apps`** — verify Stream Controller for fleet dashboard fan-in

## See Also

- **[pubnub-presence](../pubnub-presence/SKILL.md)** — [driver online/offline](../pubnub-presence/SKILL.md), [dropped-connection recovery](../pubnub-presence/references/dropped-connections.md), [multi-device sync (driver app + tablet)](../pubnub-presence/references/multi-device-sync.md)
- **[pubnub-functions](../pubnub-functions/SKILL.md)** — [After Publish for geofence + dispatch](../pubnub-functions/references/functions-basics.md), [`require('kvstore')`](../pubnub-functions/references/functions-modules.md) for last-known-location, [DB-trigger pattern to mirror to your warehouse](../pubnub-functions/references/db-triggers-and-runtime-quirks.md)
- **[pubnub-security](../pubnub-security/SKILL.md)** — [Access Manager grants](../pubnub-security/references/access-manager.md) to isolate order channels (driver vs customer vs dispatcher); [encryption](../pubnub-security/references/encryption.md) for location data; [compliance](../pubnub-security/references/compliance-reports.md) per region
- **[pubnub-reliability](../pubnub-reliability/SKILL.md)** — [queue-and-retry](../pubnub-reliability/references/queue-and-retry.md) for offline driver phones; [idempotent publish](../pubnub-reliability/references/idempotent-publish.md) for status updates; [exponential backoff](../pubnub-reliability/references/backoff-and-jitter.md) on cellular hiccups; use `signal` for high-frequency GPS via [payload hygiene](../pubnub-observability/references/cost-and-payload-hygiene.md)
- **[pubnub-history](../pubnub-history/SKILL.md)** — [Message Persistence](../pubnub-history/references/pagination-and-ordering.md) for delivery audit and replay; [offline catch-up](../pubnub-history/references/offline-catch-up.md) on customer-app reopen
- **[pubnub-scale](../pubnub-scale/SKILL.md)** — [channel groups for fleet dashboards](../pubnub-scale/references/scaling-patterns.md), [performance tuning](../pubnub-observability/references/cost-and-payload-hygiene.md)
- **[pubnub-app-context](../pubnub-app-context/SKILL.md)** — [driver profiles, vehicle metadata, customer addresses](../pubnub-app-context/references/users.md)
- **[pubnub-events-and-actions](../pubnub-events-and-actions/SKILL.md)** — route order-state-change events to ETA SMS, push, BI sinks via [action targets](../pubnub-events-and-actions/references/action-targets.md)
- **[pubnub-illuminate](../pubnub-illuminate/SKILL.md)** — [real-time fleet KPIs and exception detection via Decisions](../pubnub-illuminate/references/decisions-4-step-workflow.md)
- **[pubnub-observability](../pubnub-observability/SKILL.md)** — [logging correlation](../pubnub-observability/references/logging-correlation.md) per order, [usage metrics](../pubnub-observability/references/usage-metrics.md), [incident runbook](../pubnub-observability/references/incident-runbook.md)
- **[pubnub-choose-docs-path](../pubnub-choose-docs-path/SKILL.md)** — for routing other PubNub questions

## Output Format

When providing implementations:

1. Start with the channel naming convention and architecture diagram showing how channels relate to orders, drivers, and customers.
2. Provide complete JavaScript/TypeScript code for both the driver app (publishing) and customer app (subscribing) sides.
3. Include PubNub Functions code for any server-side validation, dispatch logic, or geofence triggers.
4. Add error handling for network failures, reconnection, and offline scenarios with code examples.
5. Finish with a testing checklist covering location accuracy, status transitions, ETA updates, and edge cases like driver reassignment.

# PubNub Delivery Tracking Setup

This reference covers channel architecture, GPS location publishing, Access Manager, and tracking-page subscriptions for real-time delivery tracking with PubNub.

## Channel Architecture for Delivery

A well-designed channel structure separates concerns and keeps subscriptions efficient. Delivery systems involve three primary actors -- customers, drivers, and dispatchers -- each needing different slices of data.

### Channel Naming Conventions

| Channel Pattern | Purpose | Subscribers |
|----------------|---------|-------------|
| `order.<orderId>.status` | Order lifecycle updates | Customer, driver, dispatch dashboard |
| `driver.<driverId>.location` | Real-time GPS coordinates | Customer (when assigned), fleet dashboard |
| `driver.<driverId>.commands` | Instructions to the driver | Driver app only |
| `dispatch.new-orders` | Incoming order broadcast | Available drivers, dispatch system |
| `dispatch.assignments` | Driver assignment confirmations | Dispatch dashboard, assigned driver |
| `fleet.<fleetId>.positions` | Aggregated fleet positions | Fleet management dashboard |
| `chat.order.<orderId>` | Driver-customer messaging | Customer, assigned driver |

### Channel Groups for Fleet Management

Use channel groups to let fleet dashboards subscribe to all active driver location channels without managing individual subscriptions.

```javascript
async function addDriverToFleet(pubnub, driverId, fleetId) {
  await pubnub.channelGroups.addChannels({
    channelGroup: `fleet-${fleetId}-locations`,
    channels: [`driver.${driverId}.location`]
  });
}

async function removeDriverFromFleet(pubnub, driverId, fleetId) {
  await pubnub.channelGroups.removeChannels({
    channelGroup: `fleet-${fleetId}-locations`,
    channels: [`driver.${driverId}.location`]
  });
}

pubnub.subscribe({
  channelGroups: ['fleet-main-locations']
});
```

## SDK Initialization

### Driver App Initialization

The driver app needs publish and subscribe permissions. Set a unique `userId` tied to the driver's identity.

```javascript
import PubNub from 'pubnub';

const pubnub = new PubNub({
  publishKey: process.env.PUBNUB_PUBLISH_KEY,
  subscribeKey: process.env.PUBNUB_SUBSCRIBE_KEY,
  userId: `driver-${driverProfile.id}`,
  restore: true,
  autoNetworkDetection: true,
  heartbeatInterval: 30
});

pubnub.setState({
  channels: [`driver.${driverProfile.id}.location`],
  state: {
    status: 'available',
    vehicleType: driverProfile.vehicleType,
    currentOrderId: null
  }
});
```

### Customer App Initialization

Customers only need subscribe access. They subscribe to their specific order channel and the assigned driver's location.

```javascript
const pubnub = new PubNub({
  subscribeKey: process.env.PUBNUB_SUBSCRIBE_KEY,
  userId: `customer-${customerId}`,
  restore: true,
  autoNetworkDetection: true
});
```

### Access Manager Configuration

Use Access Manager to control who can publish and subscribe to which channels.

```javascript
async function grantCustomerAccess(orderId, driverId, customerToken) {
  const token = await pubnub.grantToken({
    ttl: 120,  // 2 hours
    authorizedUuid: `customer-${customerToken}`,
    resources: {
      channels: {
        [`order.${orderId}.status`]: { read: true },
        [`driver.${driverId}.location`]: { read: true },
        [`chat.order.${orderId}`]: { read: true, write: true }
      }
    }
  });
  return token;
}

async function grantDriverAccess(driverId, orderId) {
  const token = await pubnub.grantToken({
    ttl: 480,  // 8 hours (full shift)
    authorizedUuid: `driver-${driverId}`,
    resources: {
      channels: {
        [`driver.${driverId}.location`]: { read: true, write: true },
        [`order.${orderId}.status`]: { read: true, write: true },
        [`driver.${driverId}.commands`]: { read: true },
        [`chat.order.${orderId}`]: { read: true, write: true }
      }
    }
  });
  return token;
}
```

When using a privacy Function, grant customers read on `order.{orderId}.driver-location` instead of raw `driver.{driverId}.location`. See [delivery-patterns.md](delivery-patterns.md).

## GPS Location Publishing

### Adaptive Frequency Publishing

Throttle publishes to `driver.{driverId}.location` so GPS ticks do not flood the keyset. Tune intervals to PubNub message volume, not to a mapping SDK.

| Driver State | Speed | Publish Interval | Rationale |
|-------------|-------|-----------------|-----------|
| Stationary | 0 km/h | Every 30 seconds | Heartbeat only |
| Slow / congested | 1-15 km/h | Every 5 seconds | Frequent updates in urban areas |
| Normal driving | 16-60 km/h | Every 3 seconds | Smooth map tracking |
| Highway | 61+ km/h | Every 2 seconds | Fast movement needs more points |
| Near destination | Any | Every 1 second | Maximum accuracy for arrival |

```javascript
class AdaptiveLocationPublisher {
  constructor(pubnub, driverId) {
    this.pubnub = pubnub;
    this.driverId = driverId;
    this.channel = `driver.${driverId}.location`;
    this.lastPublishTime = 0;
  }

  getPublishInterval(speed, nearDestination) {
    if (nearDestination) return 1000;
    if (speed < 1) return 30000;
    if (speed < 15) return 5000;
    if (speed < 60) return 3000;
    return 2000;
  }

  publish(position, nearDestination) {
    const interval = this.getPublishInterval(position.speed || 0, nearDestination);
    const now = Date.now();
    if (now - this.lastPublishTime < interval) return;

    this.pubnub.publish({
      channel: this.channel,
      message: {
        lat: position.latitude,
        lng: position.longitude,
        heading: position.heading || 0,
        speed: position.speed || 0,
        accuracy: position.accuracy || null,
        timestamp: now,
        driverId: this.driverId
      },
      storeInHistory: false
    });

    this.lastPublishTime = now;
  }
}
```

## Tracking Page Implementation

### Customer Tracking Page

Subscribe to `order.{orderId}.status` and either the privacy-filtered location channel or (if AM allows) `driver.{driverId}.location`. Catch up the last GPS point with `fetchMessages` (`count: 1`) because location publishes use `storeInHistory: false` — persist a last-known waypoint separately if you need reload after the ephemeral window.

```javascript
class DeliveryTracker {
  constructor(pubnub, orderId, driverId) {
    this.pubnub = pubnub;
    this.orderId = orderId;
    this.driverId = driverId;
  }

  start() {
    this.pubnub.subscribe({
      channels: [
        `order.${this.orderId}.status`,
        `driver.${this.driverId}.location`
      ]
    });

    this.pubnub.addListener({
      message: (event) => this.handleMessage(event)
    });

    this.fetchLastLocation();
  }

  async fetchLastLocation() {
    const channel = `driver.${this.driverId}.location`;
    const response = await this.pubnub.fetchMessages({
      channels: [channel],
      count: 1
    });
    const messages = response.channels[channel] ?? [];
    if (messages.length > 0) {
      this.onLocation(messages[0].message);
    }
  }

  handleMessage(event) {
    if (event.channel === `driver.${this.driverId}.location`) {
      this.onLocation(event.message);
    } else if (event.channel === `order.${this.orderId}.status`) {
      this.onStatus(event.message);
    }
  }

  stop() {
    this.pubnub.unsubscribe({
      channels: [
        `order.${this.orderId}.status`,
        `driver.${this.driverId}.location`
      ]
    });
  }
}
```

## Best Practices

1. **Store location history selectively.** Disable `storeInHistory` for GPS publishes to avoid filling message persistence storage with ephemeral data. Only store significant waypoints or events.

2. **Use `storeInHistory: true` for status updates.** Order status messages should be persisted so customers can reload the tracking page and see the full timeline.

3. **Set up presence for driver availability.** Use PubNub presence events (`join`, `leave`, `timeout`) on a shared `drivers.available` channel to know which drivers are online.

4. **Implement reconnection handling.** Driver apps in tunnels or dead zones must queue location updates locally and publish the latest position once reconnected, discarding stale queued points.

5. **Secure channels with Access Manager.** Never let customers publish to driver channels or other customers' order channels. Issue time-limited tokens scoped to the exact channels needed for one delivery.

6. **Plan for scale.** A fleet of 1,000 drivers publishing every 3 seconds generates about 20,000 messages per minute. Factor this into your PubNub plan and consider aggregating fleet positions server-side if the dashboard does not need per-driver granularity.

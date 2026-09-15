# PubNub Delivery Patterns

This reference covers dispatch coordination, driver-customer chat, fleet dashboards, and location privacy on PubNub.

## Dispatch Coordination

Match incoming orders with available drivers using presence on the fleet channel group plus publishes to `driver.{id}.commands` and `order.{id}.status`.

### Broadcast and Claim Dispatch

```javascript
async function broadcastOrder(pubnub, orderId, pickupLocation, orderDetails) {
  await pubnub.publish({
    channel: 'dispatch.available-orders',
    message: {
      type: 'new-order-available',
      orderId,
      pickup: pickupLocation,
      estimatedPayout: orderDetails.driverPayout,
      estimatedDistance: orderDetails.distance,
      expiresAt: Date.now() + 60000,
      timestamp: Date.now()
    }
  });
}

async function claimOrder(pubnub, orderId, driverId) {
  await pubnub.publish({
    channel: 'dispatch.order-claims',
    message: {
      type: 'claim',
      orderId,
      driverId,
      timestamp: Date.now()
    }
  });
}

// PubNub Function: Resolve conflicting claims (first-write-wins)
// Deployed as After Publish on dispatch.order-claims
// export default async (request) => {
//   const kvstore = require('kvstore');
//   const pubnub = require('pubnub');
//   const msg = request.message;
//   const existing = await kvstore.get(`order-claim-${msg.orderId}`);
//   if (existing) {
//     return pubnub.publish({
//       channel: `driver.${msg.driverId}.commands`,
//       message: { type: 'claim-rejected', orderId: msg.orderId }
//     });
//   }
//   await kvstore.set(`order-claim-${msg.orderId}`, msg.driverId);
//   return pubnub.publish({
//     channel: `driver.${msg.driverId}.commands`,
//     message: { type: 'claim-accepted', orderId: msg.orderId }
//   });
// };
```

Assignment after a successful claim: publish `dispatched` on `order.{orderId}.status` (`storeInHistory: true`) and `new-assignment` on `driver.{driverId}.commands`. Track availability via presence `join` / `leave` / `timeout` and `setState({ status: 'available' })`. If you pick a driver from last-known coordinates, do that in your dispatch service — PubNub only carries the location messages and the assignment publishes.

## Driver-Customer Real-Time Chat

Allow drivers and customers to communicate through an order-scoped chat channel, avoiding the need to share phone numbers.

### Chat Channel Setup

```javascript
class DeliveryChat {
  constructor(pubnub, orderId, userId, userRole) {
    this.pubnub = pubnub;
    this.orderId = orderId;
    this.userId = userId;
    this.userRole = userRole; // 'customer' or 'driver'
    this.channel = `chat.order.${orderId}`;
  }

  start(onMessageReceived) {
    this.pubnub.subscribe({ channels: [this.channel] });

    this.pubnub.addListener({
      message: (event) => {
        if (event.channel === this.channel) {
          onMessageReceived({
            text: event.message.text,
            sender: event.message.senderRole,
            timestamp: event.message.timestamp
          });
        }
      }
    });

    this.loadHistory(onMessageReceived);
  }

  async loadHistory(onMessageReceived) {
    const response = await this.pubnub.fetchMessages({
      channels: [this.channel],
      count: 50
    });
    const messages = response.channels[this.channel] ?? [];
    messages.forEach((msg) => {
      onMessageReceived({
        text: msg.message.text,
        sender: msg.message.senderRole,
        timestamp: msg.message.timestamp,
        isHistory: true
      });
    });
  }

  async sendMessage(text) {
    await this.pubnub.publish({
      channel: this.channel,
      message: {
        text,
        senderId: this.userId,
        senderRole: this.userRole,
        timestamp: Date.now()
      },
      storeInHistory: true
    });
  }

  stop() {
    this.pubnub.unsubscribe({ channels: [this.channel] });
  }
}
```

## Fleet Management Dashboard

A fleet dashboard provides dispatchers with a real-time overview of all active drivers and orders.

### Dashboard Data Aggregator

```javascript
class FleetDashboard {
  constructor(pubnub, fleetId) {
    this.pubnub = pubnub;
    this.fleetId = fleetId;
    this.drivers = new Map();
    this.activeOrders = new Map();
  }

  start(onUpdate) {
    this.onUpdate = onUpdate;

    this.pubnub.subscribe({
      channelGroups: [`fleet-${this.fleetId}-locations`],
      channels: ['dispatch.status-updates'],
      withPresence: true
    });

    this.pubnub.addListener({
      message: (event) => {
        if (event.channel.includes('.location')) {
          this.updateDriverPosition(event.message);
        } else if (event.channel === 'dispatch.status-updates') {
          this.updateOrderStatus(event.message);
        }
        this.onUpdate(this.getSnapshot());
      },
      presence: (event) => {
        this.updateDriverPresence(event);
        this.onUpdate(this.getSnapshot());
      }
    });
  }

  updateDriverPosition(location) {
    const existing = this.drivers.get(location.driverId) || {};
    this.drivers.set(location.driverId, {
      ...existing,
      lat: location.lat,
      lng: location.lng,
      speed: location.speed,
      heading: location.heading,
      lastUpdate: location.timestamp
    });
  }

  updateDriverPresence(event) {
    const driverId = event.uuid;
    if (event.action === 'leave' || event.action === 'timeout') {
      const driver = this.drivers.get(driverId);
      if (driver) {
        driver.online = false;
      }
    } else if (event.action === 'join') {
      const driver = this.drivers.get(driverId) || {};
      driver.online = true;
      this.drivers.set(driverId, driver);
    }
  }

  updateOrderStatus(statusMessage) {
    this.activeOrders.set(statusMessage.orderId, {
      status: statusMessage.status,
      driverId: statusMessage.driverId,
      timestamp: statusMessage.timestamp
    });

    if (statusMessage.status === 'delivered' || statusMessage.status === 'cancelled') {
      setTimeout(() => {
        this.activeOrders.delete(statusMessage.orderId);
        this.onUpdate(this.getSnapshot());
      }, 60000);
    }
  }

  getSnapshot() {
    return {
      totalDrivers: this.drivers.size,
      onlineDrivers: [...this.drivers.values()].filter((d) => d.online).length,
      activeOrders: this.activeOrders.size,
      drivers: Object.fromEntries(this.drivers),
      orders: Object.fromEntries(this.activeOrders)
    };
  }

  stop() {
    this.pubnub.unsubscribeAll();
  }
}
```

For more than ~500 concurrent location streams, aggregate server-side and publish a snapshot to `fleet.{fleetId}.positions` instead of subscribing the dashboard to every driver channel.

## Privacy Controls

Customers should not see the driver's exact location at all times. Implement privacy zones that reveal location progressively.

### Privacy Rules

| Order Status | Location Visibility | Detail Level |
|-------------|--------------------:|-------------|
| `dispatched` | General area only | City quadrant or neighborhood |
| `picked-up` | Approximate | Within 500m radius |
| `en-route` | Approximate until close | Exact when within 1km |
| `driver-nearby` | Exact | Full precision GPS |

### Privacy Filter via PubNub Function

```javascript
// PubNub Function: After Publish on driver.*.location
// Republishes a sanitized location to a customer-facing channel
export default async (request) => {
  const kvstore = require('kvstore');
  const pubnub = require('pubnub');
  const message = request.message;
  const delivery = await kvstore.get(`driver-delivery-${message.driverId}`);
  if (!delivery) return request.ok();

  const sanitizedLocation = sanitizeForCustomer(message, delivery);

  return pubnub.publish({
    channel: `order.${delivery.orderId}.driver-location`,
    message: {
      ...sanitizedLocation,
      heading: message.heading,
      timestamp: message.timestamp,
      driverId: message.driverId
    }
  });
};
```

Round coordinates by distance-to-dropoff in `sanitizeForCustomer` (exact / ~100m / ~1km). Grant customers AM read on `order.{orderId}.driver-location`, not raw `driver.{id}.location`.

### Customer Subscribes to Sanitized Channel

```javascript
function subscribeToSafeDriverLocation(pubnub, orderId) {
  pubnub.subscribe({
    channels: [`order.${orderId}.driver-location`]
  });
}
```

## Multi-Stop Delivery

Keep each stop on its own `order.{orderId}.status` channel. When a stop completes, publish `delivered` with `remainingStops` and a `multi-stop-progress` event on `dispatch.status-updates`. Do not put route-optimization logic in PubNub.

## Proof of Delivery

Publish proof **on the order status channel** with `status: 'delivered'` and `storeInHistory: true`. Put a storage URL or verification flag in `proof` — do not send photo/signature bytes through PubNub.

```javascript
await pubnub.publish({
  channel: `order.${orderId}.status`,
  message: {
    orderId,
    status: 'delivered',
    driverId,
    proof: {
      type: 'photo', // or 'pin' | 'signature'
      url: photoUrl,
      capturedAt: Date.now()
    },
    timestamp: Date.now()
  },
  storeInHistory: true
});
```

Leave-at-door can also publish a note on `chat.order.{orderId}`.

## Best Practices

1. **Set timeouts on driver assignments.** If a driver does not accept or acknowledge an assignment within 60 seconds, automatically reassign. Publish a timeout event to the dispatch channel.

2. **Scope chat channels to orders.** Always use `chat.order.<orderId>`. Isolate per delivery and clean up after the order completes.

3. **Never share phone numbers.** The PubNub chat channel eliminates the need for customers and drivers to exchange personal contact information.

4. **Aggregate fleet data server-side for large fleets.** If you have more than 500 active drivers, do not subscribe a dashboard client to every location channel.

5. **Implement progressive location disclosure.** Use the privacy filter pattern so customers far from delivery do not receive exact GPS.

6. **Clean up channels after delivery.** Once an order reaches a terminal state (`delivered` or `cancelled`), remove channels from channel groups, revoke access tokens, and unsubscribe clients.

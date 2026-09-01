# PubNub Delivery Patterns

This reference covers advanced delivery patterns including dispatch coordination, driver-customer chat, fleet management dashboards, privacy controls, multi-stop optimization, and proof of delivery workflows built on PubNub.

## Dispatch Coordination

Dispatch is the process of matching incoming orders with available drivers. The strategy you choose affects delivery speed, driver utilization, and customer satisfaction.

### Dispatch Strategies

| Strategy | How It Works | Best For | Tradeoffs |
|----------|-------------|----------|-----------|
| Nearest driver | Assign the closest available driver by straight-line distance | Low-volume, small coverage area | Does not account for traffic or route distance |
| Broadcast and claim | Broadcast the order to nearby drivers; first to accept wins | High driver density, gig workforce | May cause contention; requires timeout handling |
| Round robin | Rotate assignments evenly across available drivers | Fairness-focused fleets | Ignores proximity; may increase delivery times |
| Score-based | Rank drivers by composite score (distance, rating, load) | High-volume operations | More complex to implement and tune |
| Zone-based | Pre-assign drivers to geographic zones; route orders to zone driver | Predictable coverage areas | Less flexible for cross-zone deliveries |

### Nearest Driver Dispatch

```javascript
class DispatchSystem {
  constructor(pubnub) {
    this.pubnub = pubnub;
    this.availableDrivers = new Map();

    // Listen for driver presence and location
    this.pubnub.addListener({
      presence: (event) => this.handlePresence(event),
      message: (event) => this.handleDriverLocation(event)
    });

    this.pubnub.subscribe({
      channels: ['dispatch.new-orders'],
      channelGroups: ['fleet-main-locations'],
      withPresence: true
    });
  }

  handlePresence(event) {
    if (event.action === 'join' || event.action === 'state-change') {
      if (event.state && event.state.status === 'available') {
        this.availableDrivers.set(event.uuid, {
          status: 'available',
          ...event.state
        });
      }
    } else if (event.action === 'leave' || event.action === 'timeout') {
      this.availableDrivers.delete(event.uuid);
    }
  }

  handleDriverLocation(event) {
    if (event.channel.startsWith('driver.') && event.channel.endsWith('.location')) {
      const driver = this.availableDrivers.get(event.message.driverId);
      if (driver) {
        driver.lat = event.message.lat;
        driver.lng = event.message.lng;
        driver.lastUpdate = event.message.timestamp;
      }
    }
  }

  findNearestDriver(pickupLat, pickupLng, maxDistanceKm = 10) {
    let nearest = null;
    let minDistance = Infinity;

    for (const [driverId, driver] of this.availableDrivers) {
      if (driver.status !== 'available' || !driver.lat) continue;

      const distance = haversineDistance(
        pickupLat, pickupLng, driver.lat, driver.lng
      );

      if (distance < minDistance && distance <= maxDistanceKm * 1000) {
        minDistance = distance;
        nearest = { driverId, distance, ...driver };
      }
    }

    return nearest;
  }

  async assignOrder(orderId, pickupLat, pickupLng, dropoffLat, dropoffLng) {
    const driver = this.findNearestDriver(pickupLat, pickupLng);

    if (!driver) {
      console.log(`No available drivers for order ${orderId}`);
      await this.scheduleRetry(orderId, pickupLat, pickupLng, dropoffLat, dropoffLng);
      return null;
    }

    // Mark driver as busy
    driver.status = 'assigned';
    this.availableDrivers.delete(driver.driverId);

    // Send assignment to the driver
    await this.pubnub.publish({
      channel: `driver.${driver.driverId}.commands`,
      message: {
        type: 'new-assignment',
        orderId,
        pickup: { lat: pickupLat, lng: pickupLng },
        dropoff: { lat: dropoffLat, lng: dropoffLng },
        estimatedPickupDistance: Math.round(driver.distance),
        timestamp: Date.now()
      }
    });

    // Publish assignment confirmation
    await this.pubnub.publish({
      channel: `order.${orderId}.status`,
      message: {
        orderId,
        status: 'dispatched',
        driverId: driver.driverId,
        estimatedPickupTime: Math.round(driver.distance / 8.3) * 1000, // ~30 km/h avg
        timestamp: Date.now()
      },
      storeInHistory: true
    });

    return driver;
  }

  async scheduleRetry(orderId, pickupLat, pickupLng, dropoffLat, dropoffLng) {
    // Retry after 30 seconds
    setTimeout(() => {
      this.assignOrder(orderId, pickupLat, pickupLng, dropoffLat, dropoffLng);
    }, 30000);
  }
}
```

### Broadcast and Claim Dispatch

```javascript
async function broadcastOrder(pubnub, orderId, pickupLocation, orderDetails) {
  // Broadcast to all available drivers in the area
  await pubnub.publish({
    channel: 'dispatch.available-orders',
    message: {
      type: 'new-order-available',
      orderId,
      pickup: pickupLocation,
      estimatedPayout: orderDetails.driverPayout,
      estimatedDistance: orderDetails.distance,
      expiresAt: Date.now() + 60000, // 60 second window to claim
      timestamp: Date.now()
    }
  });
}

// Driver claims an order
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
// export default (request) => {
//   const kvstore = require('kvstore');
//   const pubnub = require('pubnub');
//   const msg = request.message;
//
//   return kvstore.get(`order-claim-${msg.orderId}`).then((existing) => {
//     if (existing) {
//       // Already claimed, notify this driver they lost
//       return pubnub.publish({
//         channel: `driver.${msg.driverId}.commands`,
//         message: { type: 'claim-rejected', orderId: msg.orderId }
//       });
//     }
//     // First claim wins
//     return kvstore.set(`order-claim-${msg.orderId}`, msg.driverId).then(() => {
//       return pubnub.publish({
//         channel: `driver.${msg.driverId}.commands`,
//         message: { type: 'claim-accepted', orderId: msg.orderId }
//       });
//     });
//   });
// };
```

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

    // Load recent message history
    this.loadHistory(onMessageReceived);
  }

  async loadHistory(onMessageReceived) {
    const response = await this.pubnub.history({
      channel: this.channel,
      count: 50
    });

    response.messages.forEach((msg) => {
      onMessageReceived({
        text: msg.entry.text,
        sender: msg.entry.senderRole,
        timestamp: msg.entry.timestamp,
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

  async sendQuickReply(templateKey) {
    const templates = {
      'running-late': "I'm running a few minutes late. Sorry for the delay!",
      'at-door': "I'm at your door with your order.",
      'cant-find': "I'm having trouble finding your location. Can you share more details?",
      'left-at-door': "I've left your order at the door. Enjoy!",
      'on-my-way': "On my way to pick up your order!"
    };

    const text = templates[templateKey];
    if (text) {
      await this.sendMessage(text);
    }
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

    // Clean up delivered or cancelled orders after a delay
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

  getDriverMetrics() {
    const metrics = {
      available: 0,
      assigned: 0,
      enRoute: 0,
      offline: 0
    };

    for (const driver of this.drivers.values()) {
      if (!driver.online) metrics.offline++;
      else if (!driver.currentOrderId) metrics.available++;
      else metrics.assigned++;
    }

    return metrics;
  }

  stop() {
    this.pubnub.unsubscribeAll();
  }
}
```

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
export default (request) => {
  const kvstore = require('kvstore');
  const pubnub = require('pubnub');
  const message = request.message;
  const driverId = message.driverId;

  return kvstore.get(`driver-delivery-${driverId}`).then((delivery) => {
    if (!delivery) return request.ok();

    const distToCustomer = haversine(
      message.lat, message.lng,
      delivery.dropoffLat, delivery.dropoffLng
    );

    let sanitizedLocation;

    if (distToCustomer < 1000) {
      // Within 1km: show exact location
      sanitizedLocation = {
        lat: message.lat,
        lng: message.lng,
        precision: 'exact'
      };
    } else if (distToCustomer < 3000) {
      // Within 3km: round to ~100m
      sanitizedLocation = {
        lat: Math.round(message.lat * 1000) / 1000,
        lng: Math.round(message.lng * 1000) / 1000,
        precision: 'approximate'
      };
    } else {
      // Far away: round to ~1km
      sanitizedLocation = {
        lat: Math.round(message.lat * 100) / 100,
        lng: Math.round(message.lng * 100) / 100,
        precision: 'general'
      };
    }

    return pubnub.publish({
      channel: `order.${delivery.orderId}.driver-location`,
      message: {
        ...sanitizedLocation,
        heading: message.heading,
        timestamp: message.timestamp,
        driverId: driverId
      }
    });
  });

  function haversine(lat1, lng1, lat2, lng2) {
    const R = 6371e3;
    const toRad = (d) => (d * Math.PI) / 180;
    const dLat = toRad(lat2 - lat1);
    const dLng = toRad(lng2 - lng1);
    const a = Math.sin(dLat / 2) ** 2 +
              Math.cos(toRad(lat1)) * Math.cos(toRad(lat2)) *
              Math.sin(dLng / 2) ** 2;
    return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
  }
};
```

### Customer Subscribes to Sanitized Channel

```javascript
// Instead of subscribing to the raw driver location channel,
// subscribe to the privacy-filtered channel
function subscribeToSafeDriverLocation(pubnub, orderId) {
  pubnub.subscribe({
    channels: [`order.${orderId}.driver-location`]
  });

  pubnub.addListener({
    message: (event) => {
      if (event.channel === `order.${orderId}.driver-location`) {
        const loc = event.message;

        if (loc.precision === 'exact') {
          updateDriverMarker(loc.lat, loc.lng);
        } else if (loc.precision === 'approximate') {
          updateDriverArea(loc.lat, loc.lng, 200); // Show a 200m radius circle
        } else {
          updateDriverArea(loc.lat, loc.lng, 1000); // Show a 1km radius circle
        }
      }
    }
  });
}
```

## Multi-Stop Delivery Optimization

For drivers handling multiple deliveries in a single trip, manage the route and status for each stop independently.

### Multi-Stop Order Management

```javascript
class MultiStopDelivery {
  constructor(pubnub, driverId) {
    this.pubnub = pubnub;
    this.driverId = driverId;
    this.stops = []; // Ordered list of deliveries
  }

  addStop(orderId, address, lat, lng, priority) {
    this.stops.push({
      orderId,
      address,
      lat,
      lng,
      priority,
      status: 'pending'
    });
    this.optimizeRoute();
  }

  optimizeRoute() {
    // Simple nearest-neighbor optimization
    if (this.stops.length < 2) return;

    const pending = this.stops.filter((s) => s.status === 'pending');
    if (pending.length < 2) return;

    // Sort by priority first, then optimize within same priority
    pending.sort((a, b) => {
      if (a.priority !== b.priority) return a.priority - b.priority;
      return 0; // Keep original order for same priority
    });
  }

  async completeStop(orderId) {
    const stop = this.stops.find((s) => s.orderId === orderId);
    if (!stop) return;

    stop.status = 'delivered';

    // Publish delivery confirmation for this specific order
    await this.pubnub.publish({
      channel: `order.${orderId}.status`,
      message: {
        orderId,
        status: 'delivered',
        driverId: this.driverId,
        timestamp: Date.now(),
        remainingStops: this.stops.filter((s) => s.status === 'pending').length
      },
      storeInHistory: true
    });

    // Notify dispatch of progress
    await this.pubnub.publish({
      channel: 'dispatch.status-updates',
      message: {
        type: 'multi-stop-progress',
        driverId: this.driverId,
        completedOrderId: orderId,
        remainingStops: this.stops.filter((s) => s.status === 'pending').length,
        timestamp: Date.now()
      }
    });
  }

  getNextStop() {
    return this.stops.find((s) => s.status === 'pending') || null;
  }

  getRemainingStops() {
    return this.stops.filter((s) => s.status === 'pending');
  }
}
```

## Proof of Delivery

Capture delivery confirmation through photos, signatures, or PIN codes, and publish the proof to the order channel.

### Photo Proof of Delivery

```javascript
async function submitPhotoProof(pubnub, orderId, driverId, photoBase64) {
  // Store the photo in your backend first, then publish the URL
  const photoUrl = await uploadPhotoToStorage(photoBase64, orderId);

  await pubnub.publish({
    channel: `order.${orderId}.status`,
    message: {
      orderId,
      status: 'delivered',
      driverId,
      proof: {
        type: 'photo',
        url: photoUrl,
        capturedAt: Date.now()
      },
      timestamp: Date.now()
    },
    storeInHistory: true
  });
}
```

### PIN Code Verification

```javascript
async function verifyDeliveryPIN(pubnub, orderId, driverId, enteredPin) {
  // Validate PIN against the stored value (via your backend or PubNub Function)
  const response = await fetch('/api/verify-delivery-pin', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ orderId, pin: enteredPin })
  });

  const result = await response.json();

  if (result.valid) {
    await pubnub.publish({
      channel: `order.${orderId}.status`,
      message: {
        orderId,
        status: 'delivered',
        driverId,
        proof: {
          type: 'pin',
          verified: true,
          verifiedAt: Date.now()
        },
        timestamp: Date.now()
      },
      storeInHistory: true
    });
    return true;
  }

  return false;
}
```

### Signature Capture

```javascript
async function submitSignatureProof(pubnub, orderId, driverId, signatureData) {
  // signatureData is a base64-encoded image from a canvas signature pad
  const signatureUrl = await uploadSignatureToStorage(signatureData, orderId);

  await pubnub.publish({
    channel: `order.${orderId}.status`,
    message: {
      orderId,
      status: 'delivered',
      driverId,
      proof: {
        type: 'signature',
        url: signatureUrl,
        signedBy: 'recipient',
        capturedAt: Date.now()
      },
      timestamp: Date.now()
    },
    storeInHistory: true
  });
}
```

### Leave-at-Door with Photo

```javascript
async function leaveAtDoor(pubnub, orderId, driverId, photoBase64, note) {
  const photoUrl = await uploadPhotoToStorage(photoBase64, orderId);

  await pubnub.publish({
    channel: `order.${orderId}.status`,
    message: {
      orderId,
      status: 'delivered',
      driverId,
      deliveryMethod: 'left-at-door',
      proof: {
        type: 'photo',
        url: photoUrl,
        note: note || 'Left at front door',
        capturedAt: Date.now()
      },
      timestamp: Date.now()
    },
    storeInHistory: true
  });

  // Notify the customer via chat
  await pubnub.publish({
    channel: `chat.order.${orderId}`,
    message: {
      text: `Your order has been left at the door. ${note || ''}`.trim(),
      senderId: driverId,
      senderRole: 'driver',
      attachmentUrl: photoUrl,
      timestamp: Date.now()
    },
    storeInHistory: true
  });
}
```

## Best Practices

1. **Choose the right dispatch strategy for your scale.** Start with nearest-driver for small fleets. Switch to broadcast-and-claim when you have many independent contractors. Use score-based dispatch only when you have enough data and volume to justify the complexity.

2. **Set timeouts on driver assignments.** If a driver does not accept or acknowledge an assignment within 60 seconds, automatically reassign to the next best driver. Publish a timeout event to the dispatch channel.

3. **Scope chat channels to orders.** Always use `chat.order.<orderId>` as the channel pattern. This ensures the chat channel is automatically isolated per delivery and can be cleaned up after the order is complete.

4. **Never share phone numbers.** The PubNub chat channel eliminates the need for customers and drivers to exchange personal contact information, improving privacy and safety.

5. **Aggregate fleet data server-side for large fleets.** If you have more than 500 active drivers, subscribing to every driver location channel from a dashboard client is expensive. Instead, use a server process to aggregate positions and publish a single fleet snapshot every few seconds to a dashboard channel.

6. **Implement progressive location disclosure.** Use the privacy filter pattern to reveal driver location gradually. Customers far from delivery do not need exact GPS coordinates. This protects driver privacy and reduces anxiety from seeing erratic GPS jumps.

7. **Require proof of delivery for high-value orders.** Implement at least one proof mechanism (photo, PIN, signature) for orders above a threshold value. Store proof references alongside order records in your database.

8. **Clean up channels after delivery.** Once an order reaches a terminal state (`delivered` or `cancelled`), remove channels from channel groups, revoke access tokens, and unsubscribe clients. This prevents channel count from growing unboundedly.

9. **Handle multi-stop delivery ETA independently.** Each customer on a multi-stop route should see their own ETA, not the total route time. Recalculate each customer's ETA based on the number of stops remaining before theirs.

10. **Test dispatch under load.** Simulate scenarios with hundreds of concurrent orders and drivers to verify that your dispatch logic scales, that claim resolution works under contention, and that the fleet dashboard remains responsive.

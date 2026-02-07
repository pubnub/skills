# PubNub Delivery Status Management

This reference covers the complete order lifecycle, status transitions, ETA calculation, geofence triggers, push notifications, and validation logic for delivery status management through PubNub.

## Order Lifecycle States

Every delivery order moves through a defined set of states. Each transition is published to the order's status channel so that customers, drivers, and dispatchers all see updates in real time.

### State Definitions

| State | Description | Triggered By |
|-------|-------------|-------------|
| `placed` | Customer has submitted the order | Customer app |
| `confirmed` | Merchant has accepted the order | Merchant system |
| `preparing` | Merchant is actively preparing the order | Merchant system |
| `ready` | Order is ready for pickup | Merchant system |
| `dispatched` | A driver has been assigned | Dispatch system |
| `driver-arrived-pickup` | Driver has arrived at merchant | Geofence / driver app |
| `picked-up` | Driver has collected the order | Driver app |
| `en-route` | Driver is heading to the customer | Driver app |
| `driver-nearby` | Driver is within 200m of delivery address | Geofence trigger |
| `delivered` | Order has been handed off to customer | Driver app |
| `failed` | Delivery could not be completed | Driver app / system |
| `cancelled` | Order was cancelled before delivery | Customer / merchant / system |

### Allowed State Transitions

| From State | Allowed Next States |
|-----------|-------------------|
| `placed` | `confirmed`, `cancelled` |
| `confirmed` | `preparing`, `cancelled` |
| `preparing` | `ready`, `cancelled` |
| `ready` | `dispatched`, `cancelled` |
| `dispatched` | `driver-arrived-pickup`, `cancelled` |
| `driver-arrived-pickup` | `picked-up`, `cancelled` |
| `picked-up` | `en-route` |
| `en-route` | `driver-nearby`, `failed` |
| `driver-nearby` | `delivered`, `failed` |
| `delivered` | (terminal state) |
| `failed` | `dispatched` (reassign) |
| `cancelled` | (terminal state) |

## Status Message Format

Every status update published to PubNub follows a consistent structure.

```javascript
const statusUpdate = {
  orderId: 'order-5678',
  status: 'en-route',
  previousStatus: 'picked-up',
  timestamp: Date.now(),
  driverId: 'driver-1234',
  eta: {
    estimatedArrival: Date.now() + 15 * 60 * 1000,
    distanceRemaining: 4200,   // meters
    durationRemaining: 900      // seconds
  },
  location: {
    lat: 40.7484,
    lng: -73.9857
  },
  metadata: {
    note: 'Traffic is light, arriving ahead of schedule'
  }
};
```

## Publishing Status Updates

### Basic Status Publisher

```javascript
class OrderStatusPublisher {
  constructor(pubnub) {
    this.pubnub = pubnub;
  }

  async publishStatus(orderId, status, extras = {}) {
    const message = {
      orderId,
      status,
      timestamp: Date.now(),
      ...extras
    };

    // Publish to the order-specific status channel
    await this.pubnub.publish({
      channel: `order.${orderId}.status`,
      message,
      storeInHistory: true
    });

    // Publish to the dispatch aggregation channel
    await this.pubnub.publish({
      channel: 'dispatch.status-updates',
      message,
      storeInHistory: true
    });

    return message;
  }

  async transitionTo(orderId, newStatus, currentStatus, extras = {}) {
    if (!this.isValidTransition(currentStatus, newStatus)) {
      throw new Error(
        `Invalid transition: ${currentStatus} -> ${newStatus} for order ${orderId}`
      );
    }

    return this.publishStatus(orderId, newStatus, {
      previousStatus: currentStatus,
      ...extras
    });
  }

  isValidTransition(from, to) {
    const transitions = {
      'placed': ['confirmed', 'cancelled'],
      'confirmed': ['preparing', 'cancelled'],
      'preparing': ['ready', 'cancelled'],
      'ready': ['dispatched', 'cancelled'],
      'dispatched': ['driver-arrived-pickup', 'cancelled'],
      'driver-arrived-pickup': ['picked-up', 'cancelled'],
      'picked-up': ['en-route'],
      'en-route': ['driver-nearby', 'failed'],
      'driver-nearby': ['delivered', 'failed'],
      'failed': ['dispatched']
    };

    const allowed = transitions[from];
    return allowed ? allowed.includes(to) : false;
  }
}
```

### Usage Example

```javascript
const statusPublisher = new OrderStatusPublisher(pubnub);

// Merchant confirms the order
await statusPublisher.transitionTo('order-5678', 'confirmed', 'placed', {
  estimatedPrepTime: 20 * 60 * 1000  // 20 minutes
});

// Driver picks up the order
await statusPublisher.transitionTo('order-5678', 'picked-up', 'driver-arrived-pickup', {
  driverId: 'driver-1234',
  pickupTimestamp: Date.now()
});
```

## Status Validation via PubNub Functions

Use a PubNub Function (Before Publish or Fire) on order status channels to validate transitions server-side, preventing invalid states even if the client has bugs.

```javascript
// PubNub Function: Before Publish on order.*.status
export default (request) => {
  const message = request.message;
  const kvstore = require('kvstore');

  const validTransitions = {
    'placed': ['confirmed', 'cancelled'],
    'confirmed': ['preparing', 'cancelled'],
    'preparing': ['ready', 'cancelled'],
    'ready': ['dispatched', 'cancelled'],
    'dispatched': ['driver-arrived-pickup', 'cancelled'],
    'driver-arrived-pickup': ['picked-up', 'cancelled'],
    'picked-up': ['en-route'],
    'en-route': ['driver-nearby', 'failed'],
    'driver-nearby': ['delivered', 'failed'],
    'failed': ['dispatched']
  };

  return kvstore.get(`order-status-${message.orderId}`).then((currentStatus) => {
    // First status for a new order
    if (!currentStatus && message.status === 'placed') {
      return kvstore.set(`order-status-${message.orderId}`, 'placed').then(() => {
        return request.ok();
      });
    }

    const allowed = validTransitions[currentStatus];
    if (!allowed || !allowed.includes(message.status)) {
      console.log(
        `Blocked invalid transition: ${currentStatus} -> ${message.status} ` +
        `for order ${message.orderId}`
      );
      return request.abort(`Invalid transition from ${currentStatus} to ${message.status}`);
    }

    // Update stored state and allow the publish
    return kvstore.set(`order-status-${message.orderId}`, message.status).then(() => {
      return request.ok();
    });
  });
};
```

## ETA Calculation and Updates

### Server-Side ETA Calculation

Calculate ETA using the driver's current position, speed, and the remaining route distance. Publish ETA updates on a regular interval or when conditions change significantly.

```javascript
class ETACalculator {
  constructor(pubnub) {
    this.pubnub = pubnub;
    this.activeDeliveries = new Map();
  }

  registerDelivery(orderId, driverId, destinationLat, destinationLng) {
    this.activeDeliveries.set(orderId, {
      driverId,
      destination: { lat: destinationLat, lng: destinationLng },
      lastETA: null,
      lastDriverLocation: null
    });
  }

  updateDriverLocation(driverId, lat, lng, speed) {
    for (const [orderId, delivery] of this.activeDeliveries) {
      if (delivery.driverId !== driverId) continue;

      delivery.lastDriverLocation = { lat, lng, speed };

      const distance = haversineDistance(
        lat, lng,
        delivery.destination.lat, delivery.destination.lng
      );

      // Simple ETA: distance / speed with a minimum speed floor
      const effectiveSpeed = Math.max(speed, 5); // At least 5 m/s (~18 km/h)
      const durationSeconds = distance / effectiveSpeed;

      // Add buffer for stops, traffic, and parking
      const bufferMultiplier = distance < 500 ? 1.5 : 1.3;
      const adjustedDuration = durationSeconds * bufferMultiplier;

      const eta = {
        estimatedArrival: Date.now() + adjustedDuration * 1000,
        distanceRemaining: Math.round(distance),
        durationRemaining: Math.round(adjustedDuration)
      };

      // Only publish if ETA changed by more than 30 seconds
      if (delivery.lastETA &&
          Math.abs(delivery.lastETA.durationRemaining - eta.durationRemaining) < 30) {
        return;
      }

      delivery.lastETA = eta;
      this.publishETA(orderId, eta);
    }
  }

  async publishETA(orderId, eta) {
    await this.pubnub.publish({
      channel: `order.${orderId}.status`,
      message: {
        orderId,
        type: 'eta-update',
        eta,
        timestamp: Date.now()
      },
      storeInHistory: false  // ETA updates are ephemeral
    });
  }
}
```

### Client-Side ETA Display

```javascript
function formatETA(etaData) {
  const minutes = Math.ceil(etaData.durationRemaining / 60);

  if (minutes <= 1) {
    return 'Arriving now';
  } else if (minutes < 60) {
    return `${minutes} min away`;
  } else {
    const hours = Math.floor(minutes / 60);
    const remainingMin = minutes % 60;
    return `${hours}h ${remainingMin}m away`;
  }
}

function formatDistance(meters) {
  if (meters < 1000) {
    return `${Math.round(meters)} m`;
  }
  return `${(meters / 1000).toFixed(1)} km`;
}
```

## Geofence Triggers

Use geofence checks to automatically trigger status transitions when a driver enters or exits a defined area.

### Geofence via PubNub Function

```javascript
// PubNub Function: After Publish on driver.*.location
export default (request) => {
  const kvstore = require('kvstore');
  const pubnub = require('pubnub');
  const message = request.message;

  const driverId = message.driverId;

  return kvstore.get(`driver-delivery-${driverId}`).then((delivery) => {
    if (!delivery) return request.ok();

    const distToPickup = haversine(
      message.lat, message.lng,
      delivery.pickupLat, delivery.pickupLng
    );
    const distToDropoff = haversine(
      message.lat, message.lng,
      delivery.dropoffLat, delivery.dropoffLng
    );

    // Driver arrived at pickup location
    if (delivery.status === 'dispatched' && distToPickup < 50) {
      return publishStatusChange(
        pubnub, delivery.orderId, 'driver-arrived-pickup', driverId
      ).then(() => {
        delivery.status = 'driver-arrived-pickup';
        return kvstore.set(`driver-delivery-${driverId}`, delivery);
      });
    }

    // Driver is nearby the customer
    if (delivery.status === 'en-route' && distToDropoff < 200) {
      return publishStatusChange(
        pubnub, delivery.orderId, 'driver-nearby', driverId
      ).then(() => {
        delivery.status = 'driver-nearby';
        return kvstore.set(`driver-delivery-${driverId}`, delivery);
      });
    }

    return request.ok();
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

  function publishStatusChange(pubnub, orderId, status, driverId) {
    return pubnub.publish({
      channel: `order.${orderId}.status`,
      message: {
        orderId,
        status,
        driverId,
        timestamp: Date.now(),
        triggeredBy: 'geofence'
      }
    });
  }
};
```

## Push Notification Patterns

Pair PubNub real-time messages with mobile push notifications so customers see updates even when the app is backgrounded.

### Configuring Push Notifications

```javascript
// Register device for push on the order status channel
async function registerForPush(pubnub, orderId, deviceToken, platform) {
  const pushGateway = platform === 'ios' ? 'apns2' : 'gcm';

  if (pushGateway === 'apns2') {
    await pubnub.push.addChannels({
      channels: [`order.${orderId}.status`],
      device: deviceToken,
      pushGateway: 'apns2',
      environment: 'production',
      topic: 'com.yourapp.delivery'
    });
  } else {
    await pubnub.push.addChannels({
      channels: [`order.${orderId}.status`],
      device: deviceToken,
      pushGateway: 'gcm'
    });
  }
}
```

### Publishing with Push Payloads

```javascript
async function publishStatusWithPush(pubnub, orderId, status, extras = {}) {
  const pushTitle = getPushTitle(status);
  const pushBody = getPushBody(status, extras);

  await pubnub.publish({
    channel: `order.${orderId}.status`,
    message: {
      orderId,
      status,
      timestamp: Date.now(),
      ...extras,
      pn_apns: {
        aps: {
          alert: { title: pushTitle, body: pushBody },
          sound: 'default',
          'mutable-content': 1
        },
        orderId,
        status
      },
      pn_gcm: {
        notification: {
          title: pushTitle,
          body: pushBody
        },
        data: { orderId, status }
      }
    },
    storeInHistory: true
  });
}

function getPushTitle(status) {
  const titles = {
    'confirmed': 'Order Confirmed',
    'preparing': 'Order Being Prepared',
    'dispatched': 'Driver Assigned',
    'picked-up': 'Order Picked Up',
    'en-route': 'Driver On The Way',
    'driver-nearby': 'Driver Almost There',
    'delivered': 'Order Delivered',
    'failed': 'Delivery Issue'
  };
  return titles[status] || 'Order Update';
}

function getPushBody(status, extras) {
  switch (status) {
    case 'confirmed':
      return 'Your order has been confirmed by the restaurant.';
    case 'dispatched':
      return `A driver has been assigned. ETA: ${Math.ceil((extras.eta?.durationRemaining || 0) / 60)} min.`;
    case 'en-route':
      return 'Your driver is on the way with your order!';
    case 'driver-nearby':
      return 'Your driver is almost there. Please be ready!';
    case 'delivered':
      return 'Your order has been delivered. Enjoy!';
    case 'failed':
      return 'There was an issue with your delivery. We are working on it.';
    default:
      return 'Your order status has been updated.';
  }
}
```

## Error Handling for Failed Deliveries

### Failed Delivery Flow

```javascript
async function handleFailedDelivery(pubnub, orderId, driverId, reason) {
  // Publish failure status
  await pubnub.publish({
    channel: `order.${orderId}.status`,
    message: {
      orderId,
      status: 'failed',
      driverId,
      reason,
      timestamp: Date.now(),
      requiresReassignment: true
    },
    storeInHistory: true
  });

  // Notify dispatch system for reassignment
  await pubnub.publish({
    channel: 'dispatch.failed-deliveries',
    message: {
      orderId,
      previousDriverId: driverId,
      failureReason: reason,
      timestamp: Date.now(),
      customerLocation: { lat: 40.7128, lng: -74.006 },
      retryCount: 1
    }
  });

  // Free the driver for new assignments
  await pubnub.publish({
    channel: `driver.${driverId}.commands`,
    message: {
      type: 'delivery-cancelled',
      orderId,
      instruction: 'Return the order to the merchant or await further instructions.'
    }
  });
}
```

### Retry and Reassignment

```javascript
async function reassignDelivery(pubnub, orderId, newDriverId, retryCount) {
  // Validate retry limit
  if (retryCount >= 3) {
    await pubnub.publish({
      channel: `order.${orderId}.status`,
      message: {
        orderId,
        status: 'cancelled',
        reason: 'Maximum delivery attempts exceeded',
        timestamp: Date.now()
      },
      storeInHistory: true
    });
    return;
  }

  // Assign new driver
  await pubnub.publish({
    channel: `order.${orderId}.status`,
    message: {
      orderId,
      status: 'dispatched',
      driverId: newDriverId,
      previousStatus: 'failed',
      retryCount,
      timestamp: Date.now()
    },
    storeInHistory: true
  });

  // Send assignment command to new driver
  await pubnub.publish({
    channel: `driver.${newDriverId}.commands`,
    message: {
      type: 'new-assignment',
      orderId,
      isReassignment: true,
      retryCount
    }
  });
}
```

## Best Practices

1. **Validate transitions on the server.** Never rely solely on client-side validation. Use PubNub Functions (Before Publish) to enforce the state machine so that buggy or malicious clients cannot push invalid status changes.

2. **Include the previous status in every update.** This makes it easy for subscribers to detect if they missed a transition and request the full history to reconcile.

3. **Persist status messages.** Always set `storeInHistory: true` for status updates. Customers who reload the tracking page should be able to fetch the complete timeline from message history.

4. **Use separate message types for ETA vs. status.** Add a `type` field (e.g., `status-change` vs `eta-update`) so subscribers can handle them with different UI logic and different persistence rules.

5. **Debounce ETA updates.** Do not publish a new ETA on every GPS tick. Only publish when the ETA changes by more than 30 seconds or the distance changes by more than 100 meters to reduce message volume.

6. **Send push notifications selectively.** Not every status change warrants a push notification. Focus on key moments: order confirmed, driver dispatched, driver nearby, and delivered.

7. **Handle terminal states cleanly.** When an order reaches `delivered` or `cancelled`, unsubscribe the customer from the driver location channel and clean up channel group memberships. Revoke access tokens.

8. **Log failed deliveries.** Store failure reasons in your backend database alongside the PubNub message history. Use this data to identify problematic addresses, unreliable drivers, or systemic issues.

9. **Use idempotent status updates.** If a status message is published twice due to a network retry, subscribers should detect the duplicate (via `orderId` + `status` + `timestamp`) and discard it.

10. **Test the full lifecycle.** Write integration tests that simulate the complete journey from `placed` to `delivered`, including failure and reassignment paths, verifying every subscriber receives every expected message.

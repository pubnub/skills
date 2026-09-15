# PubNub Delivery Status Management

This reference covers the order lifecycle on PubNub status channels, Functions validation, geofence-triggered publishes, push payloads, and failed-delivery orchestration.

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
    distanceRemaining: 4200,
    durationRemaining: 900
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

    await this.pubnub.publish({
      channel: `order.${orderId}.status`,
      message,
      storeInHistory: true
    });

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

## Status Validation via PubNub Functions

Use a PubNub Function (Before Publish or Fire) on order status channels to validate transitions server-side, preventing invalid states even if the client has bugs.

```javascript
// PubNub Function: Before Publish on order.*.status
export default async (request) => {
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

  const currentStatus = await kvstore.get(`order-status-${message.orderId}`);
  if (!currentStatus && message.status === 'placed') {
    await kvstore.set(`order-status-${message.orderId}`, 'placed');
    return request.ok();
  }

  const allowed = validTransitions[currentStatus];
  if (!allowed || !allowed.includes(message.status)) {
    return request.abort(`Invalid transition from ${currentStatus} to ${message.status}`);
  }

  await kvstore.set(`order-status-${message.orderId}`, message.status);
  return request.ok();
};
```

## ETA Calculation and Updates

Compute duration in your routing service. On PubNub, publish `type: 'eta-update'` to `order.{orderId}.status` with `storeInHistory: false`. Debounce: only publish when duration changes by more than 30 seconds or remaining distance by more than 100 meters so GPS ticks do not flood the channel.

```javascript
async function publishETA(pubnub, orderId, eta) {
  await pubnub.publish({
    channel: `order.${orderId}.status`,
    message: {
      orderId,
      type: 'eta-update',
      eta,
      timestamp: Date.now()
    },
    storeInHistory: false
  });
}
```

## Geofence Triggers

Run proximity checks in a Function on `driver.*.location`. When a threshold is crossed, **publish a status transition** to `order.{orderId}.status` (do not invent a second channel for geofence events).

Typical PubNub wiring:

| Current KV status | Condition (your distance check) | Publish status |
|-------------------|---------------------------------|----------------|
| `dispatched` | near pickup | `driver-arrived-pickup` |
| `en-route` | near dropoff | `driver-nearby` |

```javascript
// PubNub Function: After Publish on driver.*.location
export default async (request) => {
  const kvstore = require('kvstore');
  const pubnub = require('pubnub');
  const message = request.message;
  const delivery = await kvstore.get(`driver-delivery-${message.driverId}`);
  if (!delivery) return request.ok();

  // Domain: decide arrived/nearby from coordinates; then publish:
  if (shouldMarkArrivedPickup(delivery, message)) {
    await pubnub.publish({
      channel: `order.${delivery.orderId}.status`,
      message: {
        orderId: delivery.orderId,
        status: 'driver-arrived-pickup',
        driverId: message.driverId,
        timestamp: Date.now(),
        triggeredBy: 'geofence'
      }
    });
    delivery.status = 'driver-arrived-pickup';
    await kvstore.set(`driver-delivery-${message.driverId}`, delivery);
  }

  return request.ok();
};
```

## Push Notification Patterns

Pair PubNub real-time messages with mobile push notifications so customers see updates even when the app is backgrounded.

### Configuring Push Notifications

```javascript
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

  await pubnub.publish({
    channel: 'dispatch.failed-deliveries',
    message: {
      orderId,
      previousDriverId: driverId,
      failureReason: reason,
      timestamp: Date.now(),
      retryCount: 1
    }
  });

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

8. **Use idempotent status updates.** If a status message is published twice due to a network retry, subscribers should detect the duplicate (via `orderId` + `status` + `timestamp`) and discard it.

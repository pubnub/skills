# PubNub Delivery Tracking Setup

This reference covers the foundational setup for building real-time delivery tracking with PubNub, including channel architecture, GPS location publishing, SDK initialization, tracking page implementation, and battery-efficient mobile updates.

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
// Add a driver's location channel to the fleet group when they go on duty
async function addDriverToFleet(pubnub, driverId, fleetId) {
  await pubnub.channelGroups.addChannels({
    channelGroup: `fleet-${fleetId}-locations`,
    channels: [`driver.${driverId}.location`]
  });
}

// Remove when driver goes off duty
async function removeDriverFromFleet(pubnub, driverId, fleetId) {
  await pubnub.channelGroups.removeChannels({
    channelGroup: `fleet-${fleetId}-locations`,
    channels: [`driver.${driverId}.location`]
  });
}

// Fleet dashboard subscribes to the group
pubnub.subscribe({
  channelGroups: ['fleet-main-locations']
});
```

## SDK Initialization

### Driver App Initialization

The driver app needs publish and subscribe permissions. Set a unique `userId` tied to the driver's identity.

```javascript
const PubNub = require('pubnub');

const pubnub = new PubNub({
  publishKey: process.env.PUBNUB_PUBLISH_KEY,
  subscribeKey: process.env.PUBNUB_SUBSCRIBE_KEY,
  userId: `driver-${driverProfile.id}`,
  restore: true,           // Automatically reconnect and catch up
  autoNetworkDetection: true,
  heartbeatInterval: 30    // Presence heartbeat every 30 seconds
});

// Set driver state for presence
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

Use PubNub Access Manager (PAM) to control who can publish and subscribe to which channels.

```javascript
// Server-side: Grant a customer read access to their order and driver channels
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

// Server-side: Grant a driver publish access to their location and order channels
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

## GPS Location Publishing

### Adaptive Frequency Publishing

Adjust publishing frequency based on driver speed and activity. This saves bandwidth and battery while maintaining accuracy when it matters most.

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
    this.lastLocation = null;
    this.destinationCoords = null;
    this.intervalId = null;
  }

  getPublishInterval(speed, distanceToDestination) {
    // Near destination: max frequency
    if (distanceToDestination !== null && distanceToDestination < 200) {
      return 1000;
    }
    if (speed < 1) return 30000;
    if (speed < 15) return 5000;
    if (speed < 60) return 3000;
    return 2000;
  }

  publish(position) {
    const { latitude, longitude, heading, speed } = position;

    const message = {
      lat: latitude,
      lng: longitude,
      heading: heading || 0,
      speed: speed || 0,
      accuracy: position.accuracy || null,
      timestamp: Date.now(),
      driverId: this.driverId
    };

    this.pubnub.publish({
      channel: this.channel,
      message: message,
      storeInHistory: false  // Location data is ephemeral
    });

    this.lastLocation = message;
  }

  setDestination(lat, lng) {
    this.destinationCoords = { lat, lng };
  }

  start() {
    if (navigator.geolocation) {
      this.watchId = navigator.geolocation.watchPosition(
        (pos) => {
          const distToDest = this.destinationCoords
            ? haversineDistance(
                pos.coords.latitude, pos.coords.longitude,
                this.destinationCoords.lat, this.destinationCoords.lng
              )
            : null;

          const interval = this.getPublishInterval(pos.coords.speed || 0, distToDest);

          // Throttle publishing to the calculated interval
          const now = Date.now();
          if (!this.lastPublishTime || (now - this.lastPublishTime) >= interval) {
            this.publish(pos.coords);
            this.lastPublishTime = now;
          }
        },
        (err) => console.error('Geolocation error:', err),
        { enableHighAccuracy: true, maximumAge: 1000 }
      );
    }
  }

  stop() {
    if (this.watchId) {
      navigator.geolocation.clearWatch(this.watchId);
    }
  }
}
```

### Haversine Distance Utility

Used throughout the delivery system for distance calculations.

```javascript
function haversineDistance(lat1, lng1, lat2, lng2) {
  const R = 6371e3; // Earth's radius in meters
  const toRad = (deg) => (deg * Math.PI) / 180;

  const dLat = toRad(lat2 - lat1);
  const dLng = toRad(lng2 - lng1);

  const a =
    Math.sin(dLat / 2) * Math.sin(dLat / 2) +
    Math.cos(toRad(lat1)) * Math.cos(toRad(lat2)) *
    Math.sin(dLng / 2) * Math.sin(dLng / 2);

  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
  return R * c; // Distance in meters
}
```

## Tracking Page Implementation

### Customer Tracking Page

The tracking page subscribes to both the order status channel and the driver location channel, rendering updates on a map and a status timeline.

```javascript
class DeliveryTracker {
  constructor(pubnub, orderId, driverId, mapElement) {
    this.pubnub = pubnub;
    this.orderId = orderId;
    this.driverId = driverId;
    this.map = new google.maps.Map(mapElement, {
      zoom: 15,
      center: { lat: 0, lng: 0 }
    });
    this.driverMarker = null;
    this.destinationMarker = null;
    this.routeLine = null;
  }

  start(deliveryAddress) {
    // Place destination marker
    this.destinationMarker = new google.maps.Marker({
      position: deliveryAddress,
      map: this.map,
      icon: '/icons/destination-pin.png',
      title: 'Delivery Address'
    });

    // Subscribe to channels
    this.pubnub.subscribe({
      channels: [
        `order.${this.orderId}.status`,
        `driver.${this.driverId}.location`
      ]
    });

    this.pubnub.addListener({
      message: (event) => this.handleMessage(event)
    });

    // Fetch last known driver location from history
    this.fetchLastLocation();
  }

  async fetchLastLocation() {
    const response = await this.pubnub.history({
      channel: `driver.${this.driverId}.location`,
      count: 1
    });

    if (response.messages.length > 0) {
      this.updateDriverPosition(response.messages[0].entry);
    }
  }

  handleMessage(event) {
    if (event.channel === `driver.${this.driverId}.location`) {
      this.updateDriverPosition(event.message);
    } else if (event.channel === `order.${this.orderId}.status`) {
      this.updateOrderStatus(event.message);
    }
  }

  updateDriverPosition(location) {
    const position = { lat: location.lat, lng: location.lng };

    if (!this.driverMarker) {
      this.driverMarker = new google.maps.Marker({
        position: position,
        map: this.map,
        icon: '/icons/driver-car.png',
        title: 'Your Driver'
      });
      this.map.setCenter(position);
    } else {
      this.animateMarker(this.driverMarker, position);
    }
  }

  animateMarker(marker, newPosition) {
    const start = marker.getPosition();
    const end = new google.maps.LatLng(newPosition.lat, newPosition.lng);
    const frames = 30;
    let frame = 0;

    const animate = () => {
      frame++;
      const progress = frame / frames;
      const lat = start.lat() + (end.lat() - start.lat()) * progress;
      const lng = start.lng() + (end.lng() - start.lng()) * progress;
      marker.setPosition({ lat, lng });

      if (frame < frames) {
        requestAnimationFrame(animate);
      }
    };
    animate();
  }

  updateOrderStatus(statusMessage) {
    const statusElement = document.getElementById('order-status');
    statusElement.textContent = formatStatus(statusMessage.status);

    if (statusMessage.eta) {
      const etaElement = document.getElementById('eta-display');
      etaElement.textContent = formatETA(statusMessage.eta);
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

## Battery-Efficient Mobile Location Updates

### iOS (Swift) Background Location

```swift
import CoreLocation
import PubNub

class DriverLocationManager: NSObject, CLLocationManagerDelegate {
    private let locationManager = CLLocationManager()
    private let pubnub: PubNub
    private let driverId: String
    private var lastPublishTime: Date = .distantPast
    private var isNearDestination = false

    init(pubnub: PubNub, driverId: String) {
        self.pubnub = pubnub
        self.driverId = driverId
        super.init()

        locationManager.delegate = self
        locationManager.desiredAccuracy = kCLLocationAccuracyBest
        locationManager.allowsBackgroundLocationUpdates = true
        locationManager.pausesLocationUpdatesAutomatically = false
        locationManager.showsBackgroundLocationIndicator = true
        locationManager.distanceFilter = 10 // Minimum 10m movement
    }

    func startTracking() {
        locationManager.requestAlwaysAuthorization()
        locationManager.startUpdatingLocation()
    }

    func stopTracking() {
        locationManager.stopUpdatingLocation()
    }

    func setNearDestination(_ near: Bool) {
        isNearDestination = near
        locationManager.distanceFilter = near ? 5 : 10
    }

    func locationManager(_ manager: CLLocationManager,
                         didUpdateLocations locations: [CLLocation]) {
        guard let location = locations.last else { return }

        let interval: TimeInterval = isNearDestination ? 1.0 : 3.0
        guard Date().timeIntervalSince(lastPublishTime) >= interval else { return }

        let message: [String: Any] = [
            "lat": location.coordinate.latitude,
            "lng": location.coordinate.longitude,
            "heading": location.course,
            "speed": max(location.speed, 0),
            "accuracy": location.horizontalAccuracy,
            "timestamp": Int(Date().timeIntervalSince1970 * 1000),
            "driverId": driverId
        ]

        pubnub.publish(
            channel: "driver.\(driverId).location",
            message: message
        ) { result in
            // Handle publish result
        }

        lastPublishTime = Date()
    }
}
```

### Android (Kotlin) Background Location

```kotlin
class DriverLocationService : Service() {
    private lateinit var pubnub: PubNub
    private lateinit var fusedLocationClient: FusedLocationProviderClient
    private var driverId: String = ""
    private var lastPublishTime: Long = 0

    override fun onCreate() {
        super.onCreate()
        fusedLocationClient = LocationServices.getFusedLocationProviderClient(this)
    }

    fun startTracking(driverId: String) {
        this.driverId = driverId

        val locationRequest = LocationRequest.Builder(
            Priority.PRIORITY_HIGH_ACCURACY, 2000L
        ).setMinUpdateIntervalMillis(1000L)
         .setMinUpdateDistanceMeters(5f)
         .build()

        fusedLocationClient.requestLocationUpdates(
            locationRequest,
            locationCallback,
            Looper.getMainLooper()
        )
    }

    private val locationCallback = object : LocationCallback() {
        override fun onLocationResult(result: LocationResult) {
            val location = result.lastLocation ?: return
            val now = System.currentTimeMillis()

            if (now - lastPublishTime < 2000) return

            val message = mapOf(
                "lat" to location.latitude,
                "lng" to location.longitude,
                "heading" to location.bearing,
                "speed" to location.speed,
                "accuracy" to location.accuracy,
                "timestamp" to now,
                "driverId" to driverId
            )

            pubnub.publish(
                channel = "driver.$driverId.location",
                message = message
            ).async { result ->
                // Handle result
            }

            lastPublishTime = now
        }
    }

    override fun onBind(intent: Intent?): IBinder? = null
}
```

## Best Practices

1. **Store location history selectively.** Disable `storeInHistory` for GPS publishes to avoid filling message persistence storage with ephemeral data. Only store significant waypoints or events.

2. **Use `storeInHistory: true` for status updates.** Order status messages should be persisted so customers can reload the tracking page and see the full timeline.

3. **Set up presence for driver availability.** Use PubNub presence events (`join`, `leave`, `timeout`) on a shared `drivers.available` channel to know which drivers are online.

4. **Implement reconnection handling.** Driver apps in tunnels or dead zones must queue location updates locally and publish the latest position once reconnected, discarding stale queued points.

5. **Secure channels with Access Manager.** Never let customers publish to driver channels or other customers' order channels. Issue time-limited tokens scoped to the exact channels needed for one delivery.

6. **Smooth the tracking map.** Interpolate between received GPS points with animation (as shown above) to prevent the driver marker from jumping. A 30-frame linear interpolation at 60fps gives a half-second smooth transition.

7. **Monitor battery impact.** On mobile, test location publishing in the background over a full simulated shift (8 hours) and verify that battery drain stays within acceptable limits. Use the distance filter to reduce unnecessary callbacks.

8. **Plan for scale.** A fleet of 1,000 drivers publishing every 3 seconds generates about 20,000 messages per minute. Factor this into your PubNub plan and consider aggregating fleet positions server-side if the dashboard does not need per-driver granularity.

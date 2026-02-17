# Order Notification Pipeline

## Problem/Feature Description

Our logistics team needs a PubNub Function that fires whenever a new order is placed on any of our order channels (e.g., `orders.us`, `orders.eu`). When an order message comes through, the function must forward a notification payload to an external Slack webhook endpoint and also send a stripped-down event to an internal analytics channel so that dashboards update in real time. The analytics event should not be delivered directly to subscribers -- only internal processing functions should receive it.

The Slack webhook URL and any other external credentials must not appear in the source code. The operations team will handle configuring them separately.

The function runs after the original order message has already been delivered to subscribers, so it does not need to modify or block the message.

## Output Specification

Produce the following files:

1. **`order-notify.js`** -- The complete PubNub Function source code.
2. **`deployment.md`** -- Deployment guide covering trigger type, channel pattern, secrets configuration (explain where and how to store them), and the recommended workflow before going live.

# Authenticated Rate-Limited Message Gateway

## Problem/Feature Description

Our platform needs a PubNub Function that acts as a security gateway for messages on our secure API channels. Every incoming message carries a JWT token that must be verified before the message is allowed through. After verification, the function should enforce per-user rate limiting: free-tier users are allowed 100 messages per hour, while pro-tier users get 1000 messages per hour. The user's tier is encoded in the JWT payload.

When rate-limited, the message should be blocked. When authenticated and under the limit, the function should strip the raw token from the message, attach the decoded user identity, and let the message proceed. The JWT signing secret must be securely managed -- it cannot appear anywhere in the code.

The function should run on all channels under the `secure.api` namespace (e.g., `secure.api.v1`, `secure.api.v2`).

## Output Specification

Produce the following files:

1. **`auth-gateway.js`** -- The complete PubNub Function source code.
2. **`deployment.md`** -- Deployment guide covering trigger type, channel pattern, how to configure the JWT secret, and any pre-production steps.

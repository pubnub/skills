<!-- canonical-for: IP_WHITELISTING -->
<!-- used-by: -->

> **Cross-references:** Configured per [keyset](../../pubnub-keyset-management/references/keysets-and-environments.md). Pair with [Access Manager grants](access-manager.md) for layered defense. For [event delivery to your services](../../pubnub-events-and-actions/SKILL.md), allowlist PubNub's outbound IPs on your side. SDKs identify with [userId/UUID](../../pubnub-app-developer/references/sdk-patterns.md). Verification uses the [REST API surface (`https://ps.pndsn.com`)](../../pubnub-app-developer/references/rest-api.md).

# IP Allowlisting (IP Whitelisting)

PubNub supports restricting traffic on a sub-key by source IP address. This is a **defense-in-depth** layer; it does not replace [Access Manager](access-manager.md).

## When to Use

| Scenario | Recommended |
|---|---|
| Server-to-server publish from a known data center | Yes |
| Functions / Lambdas with stable egress IPs | Yes |
| Mobile / web clients on the public Internet | **No** — unworkable |
| Limit Admin API access to your office / VPN | Yes |
| As the only access control | **No** — must combine with PAM |

## How It Works

For an IP-allowlisted sub-key, PubNub edge POPs reject any HTTP request whose source IP is not in the allowlist before processing. The rejection is at TCP/HTTP layer, not at the SDK layer; clients see a network error.

## Configuration

1. Admin Portal → your app → keyset → **Security** → **IP Allowlist** (Enterprise feature; contact PubNub Support if not visible).
2. Add CIDR ranges (`10.0.0.0/24`, single `203.0.113.5/32`, etc.).
3. Save. Changes propagate within minutes.

## Pitfalls

- **NAT and load balancers** — the IP that PubNub sees is the *egress* IP of your network, not the internal client IP. Verify with `curl https://ps.pndsn.com/time/0` from inside the network.
- **Cloud auto-scaling** — if your egress IP can change (e.g., AWS Lambda without a NAT Gateway), the allowlist will reject new instances. Use a NAT Gateway with an Elastic IP.
- **Per-environment** — dev / staging / prod have separate sub-keys with separate allowlists. Test each.
- **Mobile / browser clients can't be allowlisted** in any meaningful way (their IPs change). Use [Access Manager tokens](access-manager.md) instead.

## Verification

After enabling, run from an allowed host:

```bash
curl -i "https://ps.pndsn.com/time/0?uuid=test"
```

Expect `200 OK`. From a non-allowed host, expect a connection-level rejection or HTTP 403.

## Removing IP Allowlisting

Remove via Admin Portal. There is no rollback grace period — once removed, all IPs accept the sub-key again. Coordinate with [usage-metrics monitoring](../../pubnub-observability/references/usage-metrics.md) so you can detect a sudden traffic shape change.

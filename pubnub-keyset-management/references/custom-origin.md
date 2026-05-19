<!-- canonical-for: CUSTOM_ORIGIN -->
<!-- used-by: pubnub-choose-docs-path -->

# Custom Origin (Vanity Domain)

The canonical reference for configuring a custom origin (custom CNAME / vanity domain) so PubNub traffic flows through a subdomain branded to your company.

## What a Custom Origin Is

By default, PubNub SDKs connect to PubNub-owned hostnames like `ps.pndsn.com`. A custom origin lets you replace that hostname with one of:

- A subdomain of your own domain: `realtime.yourcompany.com`
- A subdomain under PubNub's vanity space: `xyzcorp.pubnubapi.com` (12 chars or fewer for the prefix)

## When to Use a Custom Origin

| Reason | Use? |
|---|---|
| Brand-consistent network requests in browser DevTools and corporate proxy logs | Yes |
| Easier corporate firewall allowlisting on a single subdomain you control | Sometimes |
| PubNub Support has identified custom routing benefits for your traffic profile | Yes |
| You expect to host TLS certificates yourself for the subdomain | Yes |
| You think it will increase performance (it generally won't) | No |
| You want to bypass browser per-host connection limits (it doesn't) | No |

Most projects do **not** need a custom origin. Get one only when there's a concrete business or operational reason.

## Eligibility

- Available only for paid PubNub plans.
- Coordinated through PubNub Support — not self-service.
- TLS certificate hosting (if PubNub serves the cert) typically incurs an additional monthly fee.

## How to Request

1. **Open a support ticket** at [support@pubnub.com](mailto:support@pubnub.com) or via the support portal with subject "Request Custom Origin".

2. **Include in the ticket**:
   - The PubNub account email associated with the **paid production keyset(s)** that will use the custom origin.
   - **Three subdomain choices** in order of preference. Examples:
     - `realtime.yourcompany.com` (your own domain — you'll need to add a CNAME)
     - `updates.yourapp.com`
     - `messaging.yourcorp.io`
     - or `xyzcorp.pubnubapi.com` (12-char-or-less prefix under PubNub's domain)
   - **PubNub SDKs in use** — language and version for each (e.g., JavaScript v8.x, Java v9.x, Swift v7.x). PubNub will return correct initialization code per SDK.
   - Whether you have **customers or devices in China**, which can affect routing.

3. **DNS work** (if using your own domain):
   - PubNub will provide a target endpoint.
   - You add a CNAME record in your DNS pointing your subdomain at that endpoint.

4. **Confirmation**:
   - PubNub returns the active custom origin name and SDK-specific initialization code.

## SDK Initialization with a Custom Origin

The exact parameter name varies by SDK (`origin`, `customOrigin`, `host`, etc.). Use the snippet PubNub Support provides for your SDK + version.

JavaScript example:

```javascript
const pubnub = new PubNub({
  publishKey: process.env.PN_PUBLISH_KEY,
  subscribeKey: process.env.PN_SUBSCRIBE_KEY,
  userId: getCurrentUserId(),
  origin: 'realtime.yourcompany.com',
  ssl: true,
});
```

For full SDK initialization patterns and userId requirements, see [pubnub-app-developer/references/sdk-patterns.md](../../pubnub-app-developer/references/sdk-patterns.md).

## What a Custom Origin Does Not Do

- **Does not increase per-host connection limits.** Browsers limit ~6–8 concurrent persistent connections per hostname; using your own subdomain doesn't change that.
- **Does not improve latency.** PubNub's PoP routing is the same; the hostname is just the entry point.
- **Does not bypass IP allowlist requirements** — the underlying PubNub PoP IPs still apply. See [pubnub-security/references/ip-whitelisting.md](../../pubnub-security/references/ip-whitelisting.md).
- **Does not work without DNS configuration** if using your own domain — clients will fail to resolve the host until the CNAME is in place.

## Operational Notes

- Test with a single client first; confirm it connects to the new origin in DevTools / packet capture before rolling out to all clients.
- Keep a fallback initialization that falls back to the default PubNub origin in case the custom origin has a DNS issue (your incident playbook should include "revert to default origin").
- If you host the TLS certificate, add it to your renewal calendar — an expired cert blocks all traffic on the custom origin.

## MCP Tools

- **`manage_keysets`** — confirm which keysets have a custom origin associated with them.

Custom-origin requests themselves are handled via support, not MCP.

## Related Reading

- [keysets-and-environments.md](keysets-and-environments.md) — keyset structure
- [pubnub-security/references/ip-whitelisting.md](../../pubnub-security/references/ip-whitelisting.md) — corporate firewall configuration

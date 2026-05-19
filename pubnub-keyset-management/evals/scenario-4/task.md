# Draft a Custom Origin Support Request

## Problem/Feature Description

A platform team running on PubNub's Tier 2 plan has been asked by their network and brand teams to route all PubNub traffic through a `realtime.acme.com` subdomain so that corporate firewall allowlists and DevTools logs show the company's own domain instead of `ps.pndsn.com`. They've also been told that an alternative under PubNub's vanity domain (`acmecorp.pubnubapi.com`) would be acceptable as a fallback.

Their account email on the PubNub paid keysets is `platform-eng@acme.com`. They use the JavaScript v8.x SDK in browsers and Java v9.x SDK on backend services. They have customers in China.

The platform engineering manager wants two artifacts:
1. A support ticket draft (Markdown) that includes everything PubNub Support needs to provision the custom origin in one round trip.
2. A JavaScript snippet showing how the SDK will be initialized once PubNub provides the custom origin name.

## Output Specification

Create two files in the same directory:

### File 1: `support-ticket.md`

A Markdown document with these exact section headings (in this order):

```markdown
# Subject: Request Custom Origin

## Account

## Subdomain Preferences

## SDKs in Use

## China Routing

## DNS Plan

## Operational Notes
```

Requirements:

- The "Subject" heading must be exactly `# Subject: Request Custom Origin`.
- The "Account" section must include the email `platform-eng@acme.com` and explicitly mention that this is the account on the paid production keyset(s).
- The "Subdomain Preferences" section must list THREE choices in order of preference, with `realtime.acme.com` as the first choice and `acmecorp.pubnubapi.com` as one of the alternatives.
- The "SDKs in Use" section must list both JavaScript v8.x and Java v9.x.
- The "China Routing" section must explicitly state that the team has customers in China.
- The "DNS Plan" section must say that the team will add a CNAME record once PubNub provides the target endpoint, for `realtime.acme.com`.
- The "Operational Notes" section must include at least one statement explicitly noting that a custom origin does NOT improve latency, does NOT bypass IP allowlist requirements, and does NOT increase per-host connection limits.

### File 2: `custom-origin-init.js`

A small JavaScript file that exports `initWithCustomOrigin(customOrigin, userId)`. The function initializes a PubNub instance loading `publishKey` and `subscribeKey` from `process.env.PN_PUBLISH_KEY` and `process.env.PN_SUBSCRIBE_KEY`, sets the `origin` to the `customOrigin` argument, sets `ssl: true`, and uses the `userId` argument. Add a top-of-file comment noting that this is paid-plan only and that the custom origin must be obtained via PubNub Support before the snippet works.

Use ES module syntax in the JS file.

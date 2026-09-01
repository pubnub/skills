---
name: pubnub-security
description: Secure PubNub applications with Access Manager v3, end-to-end AES-256 encryption, TLS 1.2+, IP allowlisting, DoS mitigation, and compliance posture (SOC 2, HIPAA, GDPR). Use when designing access control, issuing/revoking tokens, encrypting message and file payloads, hardening network access, or producing compliance evidence. Foundational keyset and rotation concerns are owned by pubnub-keyset-management.
license: PubNub
metadata:
  author: pubnub
  version: "0.3.0"
  domain: real-time
  triggers: pubnub, security, access manager, encryption, aes, tls, auth, ip allowlist, ip whitelist, dos, ddos, soc 2, hipaa, gdpr, compliance
  role: specialist
  scope: implementation
  output-format: code
---

# PubNub Security Specialist

You are the PubNub security specialist. Your role is to help developers secure real-time applications across access control, payload confidentiality, network hardening, and compliance.


> **Precedence:** PubNub MCP tools and official documentation are authoritative for API shapes, limits, and configuration values. This skill owns decisions, orchestration, assembly, patterns, tradeoffs, and validation.

## When to Use This Skill

Invoke this skill when:
- Implementing access control with Access Manager v3
- Issuing and rotating authentication tokens (server-side `grantToken`)
- Configuring AES-256 message and file encryption
- Verifying TLS configuration
- Enabling IP allowlisting for sub-key access
- Mitigating denial-of-service or burst attacks
- Producing compliance evidence (SOC 2, HIPAA, GDPR, ISO 27001)

> Foundational concerns — keyset structure, environment separation, secret-key rotation, [demo keys](../pubnub-keyset-management/references/demo-keys.md), [custom origin](../pubnub-keyset-management/references/custom-origin.md) — live in [pubnub-keyset-management](../pubnub-keyset-management/SKILL.md). Do not duplicate that material here. For routing security events to external systems use [Events & Actions action targets](../pubnub-events-and-actions/references/event-types.md).

## Core Workflow

1. **Enable Access Manager** in Admin Portal (requires the [Secret Key from your keyset](../pubnub-keyset-management/references/keysets-and-environments.md)).
2. **Issue tokens server-side** using `grantToken()` with the Secret Key; never put the Secret Key on a client.
3. **Configure clients** with `pubnub.setToken()`.
4. **Enable encryption** via CryptoModule for end-to-end AES-256.
5. **Verify TLS 1.2+** for all connections.
6. **Lock down network surface** — IP allowlist, DoS protection, custom origin.
7. **Audit periodically** — minimize permissions, rotate keys (see [key rotation owner](../pubnub-keyset-management/references/key-rotation-and-hygiene.md)), pull compliance evidence.

## Reference Guide

| Reference | Purpose |
|-----------|---------|
| [access-manager.md](references/access-manager.md) | Access Manager v3 setup, token grants, permissions, revocation |
| [encryption.md](references/encryption.md) | AES-256 message/file encryption, TLS configuration |
| [security-best-practices.md](references/security-best-practices.md) | Auth patterns, key handling, channel architecture |
| [ip-whitelisting.md](references/ip-whitelisting.md) | Restrict sub-key access by source IP / CIDR |
| [dos-mitigation.md](references/dos-mitigation.md) | Rate caps, abuse detection, attack response |
| [compliance-reports.md](references/compliance-reports.md) | SOC 2, HIPAA, GDPR, ISO 27001 evidence requests |

## Key Implementation Requirements


| Layer | Rule |
|-------|------|
| Token issuance | Server-side only with Secret Key; never on client |
| Client auth | `setToken()` after init; refresh before TTL expiry |
| Encryption | CryptoModule (not bare legacy `cipherKey` alone) for AES-256-CBC |
| Permissions | Smallest viable channel/group/user resource set per role |

## Constraints

- **NEVER expose the Secret Key in client code.** It belongs in [Vault / a secrets manager](../pubnub-keyset-management/references/key-rotation-and-hygiene.md).
- Use `grantToken()` + `setToken()` for new work; `authKey` + `grant()` is legacy.
- TLS 1.2+ is **required** as of February 2025.
- Token TTLs should be short (minutes, not days) for sensitive operations.
- Token revocations may take up to 60 seconds to propagate.
- IP allowlists apply at the sub-key tier; verify before deploying behind a NAT (see [ip-whitelisting.md](references/ip-whitelisting.md)).
- Cipher keys cannot be rotated without re-encrypting historical messages — design key rotation up front.

## MCP Tools

- **`grant_token`** — model token issuance from a real grant payload
- **`get_sdk_documentation`** — pull SDK-specific Access Manager and CryptoModule APIs (see [intent-to-tool routing](../pubnub-choose-docs-path/references/intent-to-tool.md))

## See Also

- **pubnub-keyset-management** — owns [keysets, key rotation, custom origin, demo keys](../pubnub-keyset-management/SKILL.md). Anything about *managing* the keys themselves.
- **pubnub-app-developer** — owns [SDK init, userId/UUID, pub/sub basics](../pubnub-app-developer/SKILL.md).
- **pubnub-functions** — Functions sign with the [secret key from Vault](../pubnub-functions/references/db-triggers-and-runtime-quirks.md).
- **pubnub-events-and-actions** — webhook auth (HMAC, headers) for [action targets](../pubnub-events-and-actions/references/action-targets.md).
- **pubnub-app-context** — restrict who can read/write [user and channel metadata](../pubnub-app-context/references/users.md) via grants.
- **pubnub-observability** — audit access via [usage metrics](../pubnub-observability/references/usage-metrics.md) and the [incident runbook](../pubnub-observability/references/incident-runbook.md).
- **pubnub-reliability** — pair short token TTL with [retry/backoff on auth failure](../pubnub-reliability/references/backoff-and-jitter.md).
- **pubnub-choose-docs-path** — for routing other PubNub questions.

## Output Format

When providing implementations:
1. Clearly separate server-side and client-side code.
2. Show `grantToken` + `setToken` first; mention legacy `authKey` only when explicitly asked.
3. Include permission grant examples scoped to the smallest viable resource set.
4. Note token TTL, revocation latency, and key rotation implications.
5. Provide complete error handling for access-denied scenarios.

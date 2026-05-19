<!-- canonical-for: COMPLIANCE_REPORTS -->
<!-- used-by: -->

> **Cross-references:** Compliance posture builds on [Access Manager](access-manager.md) (auth/authz), [encryption](encryption.md) (confidentiality), [keyset hygiene](../../pubnub-keyset-management/references/key-rotation-and-hygiene.md) (key management), and [usage metrics](../../pubnub-observability/references/usage-metrics.md) (audit). User identifiers in audit logs use the [SDK userId/UUID](../../pubnub-app-developer/references/sdk-patterns.md) convention. [Functions On Request endpoints](../../pubnub-functions/references/functions-basics.md) sometimes back compliance APIs.

# Compliance Reports

PubNub maintains the following compliance attestations / certifications. Use this page to (a) understand what's available, (b) request the right document, and (c) know your shared-responsibility cut line.

## Available Reports

| Standard | Type | Available to | How to Request |
|---|---|---|---|
| **SOC 2 Type II** | Independent attestation | Customers under NDA | Account Manager / `compliance@pubnub.com` |
| **HIPAA** | BAA + technical safeguards | Enterprise customers in healthcare | Account Manager (BAA execution required) |
| **GDPR** | DPA + processor commitments | All customers (EU + global) | Standard DPA available; sign and return |
| **ISO 27001** | Certification | Customers under NDA | Account Manager |
| **CCPA** | Processor terms | Self-serve in DPA | Standard DPA |

## SOC 2 Type II — Common Requests

The SOC 2 report is the most-requested document in security reviews. It typically covers:
- Logical access controls (PubNub-internal access to customer data)
- Encryption in transit (TLS 1.2+) and at rest
- Change management
- Incident response
- Subprocessors and data residency

**Coverage period:** SOC 2 Type II covers a 12-month observation window, refreshed annually. Always request the latest report.

## HIPAA Engagement Flow

For customers in healthcare carrying Protected Health Information (PHI) over PubNub:

1. Request a Business Associate Agreement (BAA) from your Account Manager.
2. Execute the BAA before any production PHI traffic.
3. Enable [end-to-end message encryption](encryption.md) (cipher key managed by you, not PubNub).
4. Use [Access Manager](access-manager.md) for least-privilege grants.
5. Configure [IP allowlisting](ip-whitelisting.md) for server-side integrations.
6. Maintain audit logs via [usage metrics](../../pubnub-observability/references/usage-metrics.md) and external SIEM.

## GDPR Shared Responsibility

| Concern | PubNub | You (Customer) |
|---|---|---|
| Sub-processor list | Maintained and published | Review and accept |
| Data residency (EU pinning) | Multi-region routing available | Request pinning per keyset |
| Right to erasure (Art. 17) | Provides `deleteMessages` API | Trigger deletes per request |
| Encryption | TLS in transit, optional E2E | Manage your cipher key |
| Audit | SOC 2 / ISO 27001 | Pull and retain [usage metrics](../../pubnub-observability/references/usage-metrics.md) |

## Right-to-Erasure Workflow

Triggered by a user request:

1. Delete user's metadata: `pubnub.objects.removeUUIDMetadata({ uuid })` — see [App Context users](../../pubnub-app-context/references/users.md).
2. Delete history per channel: `pubnub.deleteMessages({ channel, start, end })` — see [history retention owner](../../pubnub-history/references/retention-and-storage.md).
3. Revoke any outstanding tokens for that `userId` via [Access Manager](access-manager.md).
4. If using [Functions](../../pubnub-functions/SKILL.md), purge any KVStore entries keyed by `userId`.
5. Log the erasure with the four [correlation fields](../../pubnub-observability/references/logging-correlation.md) for your audit trail.

## Subprocessors

Current subprocessor list is published at PubNub's Trust page (request URL from Account Manager). New subprocessors are notified per the DPA.

## Data Residency

By default, PubNub routes through the nearest POP. Enterprise customers can request EU-only or US-only pinning per sub-key. Pinning may add latency; benchmark before committing.

## Evidence Bundle Checklist

When a customer of yours asks for compliance evidence about *your* PubNub-backed system:

- [ ] PubNub SOC 2 Type II (latest)
- [ ] Your DPA signed with PubNub
- [ ] BAA if PHI is in scope
- [ ] Architecture diagram showing TLS + E2E encryption boundaries
- [ ] [Access Manager](access-manager.md) policy summary (TTL, scopes)
- [ ] [Key rotation cadence](../../pubnub-keyset-management/references/key-rotation-and-hygiene.md)
- [ ] [Incident response runbook](../../pubnub-observability/references/incident-runbook.md)
- [ ] [Usage metrics retention](../../pubnub-observability/references/usage-metrics.md) (how long you keep audit logs)

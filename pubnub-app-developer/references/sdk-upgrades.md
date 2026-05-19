<!-- canonical-for: SDK_VERSION_UPGRADES -->
<!-- used-by: pubnub-observability -->

> **Cross-references:** For [Access Manager](../../pubnub-security/references/access-manager.md), [PNNetworkDownCategory dropped-connection](../../pubnub-presence/references/dropped-connections.md), [App Context](../../pubnub-app-context/references/users.md), and the `get_sdk_documentation` MCP tool ([routing](../../pubnub-choose-docs-path/references/intent-to-tool.md)) see the canonical owners.

# SDK Version Upgrades and Migration

The canonical reference for upgrading PubNub SDKs across major versions, recognizing breaking changes, and migrating safely.

## When to Upgrade

| Trigger | Action |
|---|---|
| Critical security advisory in current version | Upgrade immediately |
| New required feature in a newer version | Upgrade |
| Current version is deprecated and approaching end-of-support | Plan upgrade in next sprint |
| Routine maintenance | Upgrade ~ annually |
| Just because there's a new release | No urgency; review changelog |

## Pre-Upgrade Checklist

1. **Read the full changelog** between current and target versions, not just the latest entry.
2. **Identify breaking changes.** Major version bumps almost always include API changes.
3. **Check `enableEventEngine` default.** Recent JavaScript SDK versions changed the default to `true` — affects connection behavior.
4. **Identify deprecated methods you're using.** They may emit warnings now and be removed in the next major.
5. **Check the `userId` parameter name.** Older SDKs used `uuid`; newer SDKs use `userId`. They are aliases but the canonical name has shifted.
6. **Check listener category names.** `PNNetworkDownCategory` etc. are stable but new categories may appear.
7. **Plan a staged rollout.** Don't ship to 100% of users on day 1.

## Common Breaking Changes by SDK

### JavaScript

- v7 → v8: `enableEventEngine` default became `true`. Some custom retry strategies behave differently.
- v6 → v7: `uuid` parameter renamed to `userId` (alias kept).
- Older to v7+: requires `userId` (was optional in very old versions).

### Swift

- v6 → v7: namespace and module reorganization.
- iOS minimum version bumps roughly per major version.

### Kotlin / Java

- v9 → v10: Coroutines-first API surface; callback-based methods deprecated.
- Stricter null handling in newer Kotlin SDKs.

### Python

- v6 → v7: full async/await support; older callback patterns deprecated.

For SDK-specific upgrade guides, use the **`get_sdk_documentation`** MCP tool with your language and target version.

## The Migration Pattern

### 1. Lockstep Upgrade in a Branch

```bash
git checkout -b pubnub-sdk-upgrade
# Update package
npm install pubnub@latest
# Or pip install --upgrade pubnub
# Or update Podfile / build.gradle / etc.
```

### 2. Compile / Type-Check

Most breaking changes will surface as compile errors. Address each one referencing the changelog. Don't paper over with `as any` casts.

### 3. Run the Test Suite

Including [integration tests](../../pubnub-observability/references/test-pyramid.md) on a non-prod keyset.

### 4. Manual QA on Critical Paths

- Init + connect
- Subscribe + receive
- Publish + receive on another client
- Reconnect after network interruption (especially with `enableEventEngine` changes)
- Access Manager token usage
- App Context CRUD

### 5. Stage Rollout

Deploy to internal users first, then a small percentage of production, then full rollout. Monitor:

- `pubnub.publish.error` rate
- `pubnub.receive.process_error` rate
- Reconnect rate
- Any new status categories appearing in logs

For [logging conventions](../../pubnub-observability/references/logging-correlation.md) and [incident response](../../pubnub-observability/references/incident-runbook.md) see the canonical owners.

## Coordinating with Schema Versioning

If your SDK upgrade also bumps your message envelope `schema_version`, sequence the rollouts independently:

1. Ship the SDK upgrade with **both** schema_version handlers (old + new).
2. Verify in staging.
3. Roll out new SDK to all clients.
4. Then bump producers to write the new schema_version.

For [schema versioning](../../pubnub-reliability/references/schema-versioning.md) details see the canonical owner.

## Rollback Plan

Keep the rollback simple:

- Pin the package in `package.json` / `Podfile.lock` / `build.gradle` / `requirements.txt` to the previous version.
- Tag a release of your app with the previous PubNub SDK before merging the upgrade.
- Roll back the deploy if any of the staged-rollout metrics regress.

## Common Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Upgrading multiple major versions in one shot | Step through each major version, not skip |
| No staged rollout | 5% → 25% → 100% with monitoring at each step |
| Ignoring deprecation warnings | They become errors in the next major |
| Upgrading SDK and `schema_version` simultaneously | Sequence them; SDK first with dual support, then schema |
| No rollback plan | Pin the version; tag the previous release |
| Reading only the latest changelog entry | Read the full changelog from current to target |

## Related Reading

- [sdk-patterns.md](sdk-patterns.md) — current SDK patterns
- [pubnub-reliability/references/schema-versioning.md](../../pubnub-reliability/references/schema-versioning.md) — coordinate envelope changes
- [pubnub-observability/references/test-pyramid.md](../../pubnub-observability/references/test-pyramid.md) — test before/after upgrade

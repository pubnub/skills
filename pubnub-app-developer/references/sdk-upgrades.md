<!-- canonical-for: SDK_VERSION_UPGRADES -->
<!-- used-by: pubnub-observability -->

> **Cross-references:** For [Access Manager](../../pubnub-security/references/access-manager.md), [PNNetworkDownCategory dropped-connection](../../pubnub-presence/references/dropped-connections.md), [App Context](../../pubnub-app-context/references/users.md), and the `get_sdk_documentation` MCP tool ([routing](../../pubnub-choose-docs-path/references/intent-to-tool.md)) see the canonical owners.

# SDK Upgrades — Orchestration


## When to upgrade

| Trigger | Action |
|---------|--------|
| Security advisory | Upgrade immediately |
| Required new SDK feature | Upgrade on a planned branch |
| Deprecated / end-of-support version | Schedule next sprint |
| Routine maintenance | ~annually; read changelog first |
| New release available | No urgency by itself |

## Pre-upgrade checklist

1. Read the **full changelog** from current → target (not just latest entry).
2. Retrieve breaking changes via **`get_sdk_migration_guide`**.
3. Inventory deprecated APIs you use — warnings become hard breaks on next major.
4. Note connection-management defaults (e.g. Event Engine) — reconnect behavior may change.
5. Plan **staged rollout** — never 100% day one.

## Migration sequence

1. Branch + bump dependency — pin rollback version in lockfile first.
2. Fix compile/type errors using migration guide — don't mask with casts.
3. Run automated tests on a non-prod keyset ([test pyramid](../../pubnub-observability/references/test-pyramid.md)).
4. Manual QA: init, subscribe/receive, publish, reconnect, AM tokens, App Context if used.
5. Staged deploy (e.g. 5% → 25% → 100%) watching publish/receive errors and reconnect rate ([logging correlation](../../pubnub-observability/references/logging-correlation.md)).

## Coordinate with schema versioning

Do **not** bump SDK and message `schema_version` in the same release:

1. Ship SDK upgrade with **dual** schema handlers.
2. Roll out SDK to all clients.
3. Then move producers to the new schema version.

See [schema-versioning.md](../../pubnub-reliability/references/schema-versioning.md).

## Rollback

- Tag release before merge; keep previous SDK pin in lockfile for fast revert.
- Roll back deploy if staged metrics regress.

## Anti-patterns

| Anti-pattern | Fix |
|--------------|-----|
| Skip major versions in one jump | Step through each major |
| No staged rollout | Percentage rollout with monitoring |
| Ignore deprecation warnings | Fix before next major |
| SDK + schema_version same release | Sequence SDK first |
| Changelog entry only | Read full delta |

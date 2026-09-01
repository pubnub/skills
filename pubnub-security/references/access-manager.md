<!-- canonical-for: ACCESS_MANAGER -->
<!-- used-by: -->

> **Cross-references:** Requires the [Secret Key from your keyset](../../pubnub-keyset-management/references/keysets-and-environments.md) (server-side only). Pair with [end-to-end message encryption (AES-256)](encryption.md) and [IP allowlisting](ip-whitelisting.md) for layered defense. Token issuance integrates with [SDK initialization (`new PubNub(`, `userId`/UUID)](../../pubnub-app-developer/references/sdk-patterns.md) and the [`pubnub.publish(`](../../pubnub-app-developer/references/publish-subscribe.md) call. Channel-group grants are owned by [pubnub-scale](../../pubnub-scale/references/scaling-patterns.md).

# Access Manager — Orchestration and Decisions


## Enabling AM changes the default

Once Access Manager is enabled on a keyset, **all channel access is token-gated**. Plan a server token-minting path before enabling in production.

## Token flow (orchestration)

1. **Server** (with secret key) mints token via `grantToken` for the authenticated user's `authorizedUUID` and required channel/group/uuid resources.
2. **Client** initializes PubNub with publish/subscribe keys + persistent `userId`, then **`setToken(token)`** (not legacy `authKey` for new apps).
3. **Client** handles `PNAccessDeniedCategory` by re-authing and refreshing the token.
4. **Schedule refresh** before TTL expiry (e.g. 5 minutes early) — do not wait for hard failure.

## TTL guidance by sensitivity

| Use case | Typical TTL direction |
|----------|----------------------|
| Demo / short session | Shorter (tens of minutes) |
| Normal user session | Hours to one day |
| Long-lived service account | Days (within platform max) |
| High-sensitivity channels | Shorter TTL + narrower grants |

Retrieve current TTL limits from SDK docs before hard-coding values.

## Grant design decisions

- Prefer **explicit channel lists** when cardinality is low and ACLs differ per room.
- Use **patterns** when many channels share one policy (e.g. `public-*` read-only).
- Grant **channel-group** permissions separately when using Stream Controller groups.
- Pair grants with [encryption](encryption.md) and [IP allowlisting](ip-whitelisting.md) for layered defense.

## Revocation and propagation

Token revocation can take **~60 seconds** to propagate due to caching — do not assume instant lockout for abuse response; also block at your app layer if needed.

## Legacy `grant()` / `authKey`

Older apps may use server `grant()` + client `authKey`. **New implementations should use `grantToken()` + `setToken()`.** Chat SDK uses **`authKey`** (not `token`) for the AM token field name — retrieve Chat SDK init docs when mixing stacks.

## Validation checklist

- [ ] Secret key never shipped to clients
- [ ] Token TTL matches session model
- [ ] Access denied triggers refresh, not infinite retry
- [ ] Presence channels (`-pnpres`) granted when using presence on restricted channels

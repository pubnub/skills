# Environment-Aware PubNub Initialization

## Problem/Feature Description

A backend team has just bootstrapped a new PubNub-backed service. The infrastructure includes three deployment environments (dev, staging, production), each with its own PubNub keyset. The team's standard practice is to keep every secret out of source control by loading from environment variables, and to fail fast at process startup if a required variable is missing rather than silently initialize with an undefined key.

The team's senior engineer wants a small, reusable initialization module that any backend service will import. The module must choose the right env-var names based on the current environment, throw a clear error if any required var is missing, and never embed any key literal.

## Output Specification

Create a JavaScript file called `pubnub-init.js` that exports the following:

1. `getEnvOrThrow(name)` - Reads `process.env[name]`. Returns the value if present and non-empty; otherwise throws `Error('Missing required env var: <name>')`.
2. `initServerPubNub(appName, environment)` - Returns an initialized PubNub instance. `appName` (e.g., `"acme-chat"`) and `environment` (`"dev" | "staging" | "prod"`) are normalized into the env-var prefix `<APPNAME>_<ENV>` (uppercase, hyphens replaced with underscores). The function then loads:
   - `<PREFIX>_PUBLISH_KEY` → `publishKey`
   - `<PREFIX>_SUBSCRIBE_KEY` → `subscribeKey`
   - `<PREFIX>_SECRET_KEY` → `secretKey`
   - sets `userId` to `'server-' + os.hostname()`
3. `initClientPubNub(appName, environment, userId)` - Returns an initialized PubNub instance that loads ONLY the publish and subscribe keys (no secret key) using the same prefix scheme. `userId` is the supplied argument.

At the top of the file, include a comment that explicitly states:
- The secret key must never appear in client-side code.
- Keys are loaded only from environment variables; no literals appear anywhere in this module.

Use ES module syntax. Import PubNub via `import PubNub from 'pubnub';` (or the equivalent ESM import).

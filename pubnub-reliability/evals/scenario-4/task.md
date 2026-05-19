# Tolerant-Reader Schema Version Router

## Problem/Feature Description

The product is rolling out a schema_version bump from `1` to `2` for its chat messages. The receiver code base currently has an `if/else` chain that crashes when it encounters an unknown field or an unknown version. The platform engineering lead wants a small, well-tested receiver module that:

- Reads `schema_version` first;
- Routes to per-version handlers (v1, v2);
- Drops with a log line when the message is below MIN_SUPPORTED or above MAX_SUPPORTED;
- Prompts the user to update the app on too-new versions;
- Never crashes on unknown fields inside a handler.

The team also wants a brief migration-flow note pinned at the top of the file so future maintainers don't ship a producer change ahead of receiver dual-support.

## Output Specification

Create a JavaScript file called `tolerant-reader.js` that exports the following:

1. Constants `MIN_SUPPORTED = 1` and `MAX_SUPPORTED = 2`.
2. A `HANDLERS` object with at least the keys `1` and `2`, each mapped to a no-op stub function (the caller will replace them in tests). The stub function signature is `(payload) => void`.
3. `process(envelope, logger, promptUserToUpdate)` - Async (or sync, agent's choice). Behavior:
   - If `envelope` is null/undefined or `envelope.schema_version` is not a `number`, call `logger.warn('schema.version.missing', { message_id: envelope?.message_id })` and return without throwing.
   - Else if `envelope.schema_version < MIN_SUPPORTED`, call `logger.warn('schema.version.too_old', { v: envelope.schema_version, message_id: envelope.message_id })` and return.
   - Else if `envelope.schema_version > MAX_SUPPORTED`, call `logger.warn('schema.version.too_new', { v: envelope.schema_version, message_id: envelope.message_id })`, then call `promptUserToUpdate()`, and return.
   - Otherwise call `HANDLERS[envelope.schema_version](envelope.payload)`.
4. `applyMigrationStep(stepName)` - Returns a Markdown string describing the documented migration flow with these named steps in order:
   - `"T0"` → "Producer + receiver both at v1."
   - `"T1"` → "Receiver updated to handle BOTH v1 and v2."
   - `"T2"` → "Producer updated to write v2."
   - `"T3"` → "After confidence period, drop v1 handler in receivers."
   - For any other input, return `"Unknown step. Valid steps: T0, T1, T2, T3."`.

Add a top-of-file comment explicitly stating:
- Schema version is a monotonic integer; v1 is forever v1 (never reuse a version number for a different shape).
- The receiver is a tolerant reader; unknown fields are ignored, not crashed on.
- Migration order is T0 → T1 → T2 → T3; never go T0 → T2 directly.

Use ES module syntax.

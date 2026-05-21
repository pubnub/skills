<!-- canonical-for: FUNCTION_BUNDLING -->
<!-- used-by: -->

> **Cross-references:** [Handler shapes and async/await](functions-basics.md). [Handler-scoped `require`](db-triggers-and-runtime-quirks.md#quirk-8-require-is-handler-scoped). [3-call XHR/PubNub cap](db-triggers-and-runtime-quirks.md#quirk-2-3-call-cap-on-xhr--pubnub-api-calls) and [10-call vault cap](functions-modules.md#vault-module) constrain what the bundled handler can do at runtime. For pub/sub semantics inside the bundled handler, see the [pub/sub canonical owner](../../pubnub-app-developer/references/publish-subscribe.md). This document is **build-system-specific**: the constraints from §3 onward apply only when you bundle/transpile code (esbuild, Rollup, webpack, tsc) before uploading. If you author directly in the PubNub Admin Portal, you can skip them.

# Bundling and TypeScript for PubNub Functions

PubNub Functions accept JavaScript modules with an `export default` handler. Anything you bundle, transpile, or minify before uploading must end up looking like that handler — and several toolchain defaults break the contract. This reference covers what to do and what to avoid.

## 1. Functions adapter vs Node adapter

A two-adapter strategy lets you share business logic between local Node (tests, local CLI runs) and PubNub Functions (production):

- **Functions adapter** (the one that runs in Functions): uses Functions built-ins via `require()` — `xhr`, `kvstore`, `vault`, `pubnub`, `crypto`, `jwt`, etc.
- **Node adapter** (the one that runs locally): uses Node.js APIs and libraries — Node `crypto`, `fs`, `node-fetch`, `jose` for JWT, etc.

Author your business logic against an internal abstraction (`Db.get(key)`, `Http.fetch(url)`, `Signer.sign(payload)`) and select the adapter at build time. The Functions bundle imports the Functions adapter; local tests import the Node adapter. Keep the abstraction surface small so neither adapter accumulates surprises.

## 2. 64 KB bundle size guard

The maximum bundle size accepted by Functions is approximately **64 KB**. Enforce this in your build so you catch regressions before deploy:

```bash
# Example: fail the build if the bundle exceeds 64 KB
SIZE=$(stat -f%z dist-functions/myfn.js 2>/dev/null || stat -c%s dist-functions/myfn.js)
if [ "$SIZE" -gt 65536 ]; then
  echo "Bundle is $SIZE bytes; exceeds 64KB limit" >&2
  exit 1
fi
```

**Stay-under tactics:**

- **Tree-shake aggressively.** ESM imports + esbuild's tree-shaker drop unused exports.
- **Externalize Functions built-ins** (see §3). They're provided by the runtime; bundling them in is both wasted bytes and broken behavior.
- **Don't import full libraries for one helper.** Use `import { specific } from 'lib'` rather than `import * as lib from 'lib'`.
- **Use minification.** `--minify` typically halves the bundle.

## 3. Externalize Functions built-ins (esbuild `--external:` list)

When bundling for Functions, never bundle the built-in modules. Mark them all as externals so esbuild leaves the `require('module')` call intact:

```
--external:pubnub
--external:xhr
--external:kvstore
--external:vault
--external:crypto
--external:jwt
--external:jsonpath
--external:advanced_math
--external:uuid
--external:utils
--external:ugc
--external:codec/auth
--external:codec/base64
--external:codec/query_string
```

If you forget an external, esbuild will try to bundle a fake `kvstore` from `node_modules` (or fail to resolve) and your handler will run against the wrong implementation.

## 4. `__require` / `Dynamic require of` shim stripping

esbuild's **ESM** output sometimes rewrites `require()` calls into a helper named `__require` to support dynamic `require(...)` resolution. The Functions runtime does not provide `__require`, so the bundle must use plain `require(...)`.

**Detect** — search the minified bundle for either:

- The token `__require`
- The string `"Dynamic require of "` (esbuild's runtime error message — a reliable marker that the shim was emitted)

**Strip** — post-process the bundle to (1) remove the `__require` shim definition entirely, and (2) rewrite any `__require(...)` calls back to `require(...)`. The Functions runtime's handler-scoped `require` (see [Quirk 8](db-triggers-and-runtime-quirks.md#quirk-8-require-is-handler-scoped)) then handles the rewritten calls correctly.

```bash
# Quick detect
grep -nE '__require|Dynamic require of' dist-functions/myfn.js
```

If either grep matches, your bundle is broken until you fix it.

## 5. Default export shape

The Functions runtime expects exactly one `export default <handler-expression>` and prefers it to be the first non-comment statement in the bundle. Common toolchain outputs to watch for:

| Output shape | Status |
|---|---|
| `export default (request, response) => { ... }` | **Preferred.** Inline arrow as the first statement. |
| `function handler(req, res) { ... } export { handler as default };` | Risky. Some runtime versions reject the renamed-default form. Rewrite (see §10). |
| `function handler(req, res) { ... } export default handler;` | Usually fine. Easy to rewrite to inline form if needed. |
| Multiple `export default` statements | **Broken.** Reject in build. |
| `export default` wrapped inside an IIFE or after `globalThis` assignments | Risky. Aliases can be uninitialized at the moment the runtime captures the export. |

**Rewrite rule:** when in doubt, post-process the bundle so `export default (request, response) => { return handler(request, response); };` (or the request-only variant for event triggers) is the **first** statement.

## 6. Require placement inside the handler

Inside the Functions editor, top-level `require()` usually works. When bundling, **prefer requires inside the handler body**:

```javascript
export default async (request) => {
  const db  = require('kvstore');   // captured inside the handler
  const xhr = require('xhr');
  // ...
};
```

Two reasons:

1. Bundlers can rewrite top-level `require(...)` into ESM imports or hoisted helpers, breaking the contract with the Functions runtime.
2. Per [Quirk 8](db-triggers-and-runtime-quirks.md#quirk-8-require-is-handler-scoped), `require` is only reliably bound inside the handler scope.

**Minified-output check:** after minification, search the bundle for `require(`. The arguments to each call should still be plain string literals (`require("kvstore")`, etc.) — not an aliased shim and not `__require`.

```bash
# Quick check
grep -oE 'require\([^)]+\)' dist-functions/myfn.js | sort -u
```

Avoid dynamic-require tricks like `Function('return require')()` or `eval('require')` — they fail in constrained runtimes and trip the `__require` shim emission.

## 7. Minification and hoisting

Minifiers sometimes rewrite function declarations into `const`-bound arrow functions when they think it's safe:

```javascript
// Before minification
function getDb() { return require('kvstore'); }
export default async (request) => {
  const db = getDb();   // hoisted — works
};

// After aggressive minification
const getDb = () => require('kvstore');
export default async (request) => {
  const db = getDb();   // still works because getDb is declared before use
};
```

The trap is when the entry **calls a helper before its definition** in source order. Function declarations are hoisted; `const` arrows are not. If you rely on hoisting, **use function declarations** (`function foo() {}`) and configure the minifier to preserve them:

```bash
# esbuild: --keep-names preserves function names
--keep-names
```

**Symptom of getting this wrong:** runtime error `TypeError: X is not a function` at handler entry, only in the bundled/minified output, never in dev.

## 8. Avoid fragile helper re-exports

If you rewrite the export shape in post-processing, avoid relying on re-exported helper objects:

```javascript
// FRAGILE: helpers re-exported from a sub-module, then the default-export
// rewriter moves things around — helpers can end up undefined at runtime
export { logger, db } from './shared';
export default async (request) => {
  // ...
};
```

**Safer patterns:**

- **Direct imports** in the entry file. No re-exports.
- **Inline constants** in the entry file for small config.
- **Lazy getters** for handler-bound dependencies: `const getDb = () => require('kvstore');`.

## 9. Manual esbuild command

A working command for a TypeScript Function (single entry, ESM output, externalized built-ins, size-friendly):

```bash
npx esbuild functions/my-handler.ts \
  --bundle \
  --platform=node \
  --format=esm \
  --target=es2020 \
  --minify \
  --keep-names \
  --external:pubnub \
  --external:xhr \
  --external:kvstore \
  --external:vault \
  --external:crypto \
  --external:jwt \
  --external:jsonpath \
  --external:advanced_math \
  --external:uuid \
  --external:utils \
  --external:ugc \
  --external:codec/auth \
  --external:codec/base64 \
  --external:codec/query_string \
  --outfile=dist-functions/my-handler.js
```

If you have many handlers, wrap this in `npm run functions:build` and iterate over `functions/*.ts`.

## 10. Post-process default-export fixup script

If your bundler emits `export { handler as default }` (e.g., when transpiling some TS class shapes), rewrite it to inline-arrow form. Save as `fix-export.mjs`:

```javascript
import { readFileSync, writeFileSync } from 'node:fs';

const filePath = process.argv[2];
if (!filePath) {
  console.error('usage: node fix-export.mjs <bundle.js>');
  process.exit(1);
}

let content = readFileSync(filePath, 'utf8');

// 1) `export { foo as default };` -> inline arrow that calls foo
content = content.replace(
  /export\s*\{\s*([A-Za-z0-9_$]+)\s+as\s+default\s*\};?/,
  'export default (request, response) => { return $1(request, response); };'
);

// 2) `export default foo;` -> inline arrow that calls foo
content = content.replace(
  /export\s+default\s+([A-Za-z0-9_$]+)\s*;?/,
  'export default (request, response) => { return $1(request, response); };'
);

// 3) Move the rewritten export-default line to the top of the file
const m = content.match(
  /export default \(request, response\) => \{ return [A-Za-z0-9_$]+\([^)]*\); \};/
);
if (m) {
  content = content.replace(m[0], '').trimStart();
  content = `${m[0]}\n${content}`;
}

writeFileSync(filePath, content, 'utf8');
console.log('Rewrote default export in', filePath);
```

Run after esbuild:

```bash
node fix-export.mjs dist-functions/my-handler.js
```

**Adjust the signature** for event triggers (`request` only) vs On Request (`request, response`):

```javascript
// For Before/After Publish, Presence, On Interval:
'export default (request) => { return $1(request); };'
```

## 11. LLM do / don't checklist

Use this checklist when generating or reviewing Functions code that will be bundled:

**Do**

- Use the correct handler signature for the trigger type (see [`functions-basics.md`](functions-basics.md)).
- Return the trigger's completion call: `request.ok()`, `request.abort()`, `event.ok()`, `event.abort()`, or `response.send(...)`.
- Put `require()` calls **inside** the handler body.
- Mark every Functions built-in as `--external:` for the bundler (full list in §3).
- Use `async`/`await` for new code; Promise chains are acceptable when returned from the handler (see [`functions-basics.md`](functions-basics.md#promise-chains-are-acceptable-if-you-return-them)).
- Keep external calls minimal — see the 3-call cap in [Quirk 2](db-triggers-and-runtime-quirks.md#quirk-2-3-call-cap-on-xhr--pubnub-api-calls).
- Keep vault lookups minimal — 10 per execution (see [Vault Module](functions-modules.md#vault-module)).
- Guard `vault.get(...)` and check `typeof vault.get === 'function'` (see [Quirk 10](db-triggers-and-runtime-quirks.md#quirk-10-vault-may-be-unavailable-or-return-not-found)).
- Use `function` declarations for helpers if your entry calls them before their textual definition (see §7).
- Verify the minified bundle's `require(...)` calls are plain (no `__require`, no `Dynamic require of`).
- Enforce the 64 KB size guard in CI (see §2).

**Don't**

- Don't assume Node.js built-ins exist — no `fs`, no Node `crypto`, no `process`, no `Buffer`, no Node-flavored `setTimeout`/`setInterval` semantics.
- Don't install npm packages for runtime usage that aren't already provided by the Functions module list.
- Don't omit the completion return (`ok` / `abort` / `send`). The runtime has no fallback for "no return".
- Don't fan out N HTTP calls in a loop — you'll trip the 3-call cap on the second iteration.
- Don't rely on `globalThis.require` being set by the runtime (see [Quirk 8](db-triggers-and-runtime-quirks.md#quirk-8-require-is-handler-scoped)).
- Don't ship bundles that still contain `__require` or `Dynamic require of` — they will fail at first invocation.
- Don't use dynamic-require tricks (`Function('return require')()`, `eval('require')`).
- Don't author multiple `export default` statements. Exactly one.
- Don't use `import` for Functions built-ins (e.g., `import * as kv from 'kvstore'`). The runtime expects `require('kvstore')`.

# Secret-Key Rotation Runbook Module

## Problem/Feature Description

A security incident response team has just been notified that an engineer with secret-key access has left the company. Their policy requires the secret key to be rotated within 24 hours. The platform engineering manager wants a reusable Node.js runbook module that walks through the documented rotation steps in order, asks for an explicit confirmation before each destructive step, and produces a structured audit log of what was done — without ever printing the actual secret key.

## Output Specification

Create a JavaScript file called `secret-key-rotation.js` that exports an async function `rotateSecretKey(input)` returning an `{ ok, audit }` object. `input` has shape:

```
{
  appName: string,
  environment: 'dev' | 'staging' | 'prod',
  newSecretKey: string,
  confirmRotation: 'yes-rotate-' + environment,
  reissueTokens: () => Promise<{ count: number }>,
  verifyToken: () => Promise<boolean>,
  auditSecretManagerAccess: () => Promise<string[]>
}
```

The function must perform the following steps in this exact order, recording each in the returned `audit` array with `{ step, status, detail? }` entries. It must short-circuit and return `{ ok: false, audit }` at the first failure:

1. **Confirm**: verify that `input.confirmRotation === 'yes-rotate-' + input.environment`. If not, record the audit entry and short-circuit. Otherwise, append `{ step: 'confirmed', status: 'ok' }`.
2. **Update server secret store**: assume the new key is already provisioned in the env vars; record `{ step: 'server-secret-store-updated', status: 'ok' }`.
3. **Re-grant Access Manager tokens**: call `input.reissueTokens()`. Record `{ step: 're-grant-tokens', status: 'ok', detail: { count } }`.
4. **Verify**: call `input.verifyToken()`. If it returns false, record a failure and short-circuit.
5. **Audit secrets manager access**: call `input.auditSecretManagerAccess()`. Record the returned principal list under `detail`.

In NO circumstance may the audit log contain the value of `input.newSecretKey` or any substring of it. Do not log it to stdout, do not store it in the audit object.

Add a top-of-file comment that explicitly states:
- This module is server-only.
- The new secret key value is never written to the audit log.
- The runbook short-circuits on the first failure rather than continuing.

Use ES module syntax.

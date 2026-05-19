# Pre-commit Hook to Block PubNub Keys in Staged Changes

## Problem/Feature Description

A PubNub project's security review found that publish, subscribe, and secret keys had been accidentally committed to source control on two separate occasions over the past quarter — each requiring an emergency rotation. The platform engineering manager wants every developer's machine to run a git pre-commit hook that detects PubNub key prefixes in staged content and blocks the commit before it lands.

The hook should be implemented as a small, dependency-free Node.js script that reads the staged diff via `git diff --cached` and exits non-zero if any line contains a substring matching the documented PubNub key prefixes.

## Output Specification

Create a Node.js script called `pre-commit-pubnub-keys.js` whose top-of-file shebang is `#!/usr/bin/env node`. The script must:

1. Execute `git diff --cached` (using `child_process.execSync` or equivalent) and capture stdout as a UTF-8 string.
2. Scan the staged diff for matches against the regex `/(pub-c-|sub-c-|sec-c-)[a-f0-9-]{36}/`. Use a regex literal — do not call out to an external scanner.
3. If any match is found:
   - Print to stderr: `ERROR: PubNub key detected in staged changes.` followed by a second line: `Move it to a secrets manager and unstage the file.`
   - Exit with code 1.
4. If no match is found:
   - Print to stdout: `pre-commit-pubnub-keys: no PubNub keys detected in staged changes`
   - Exit with code 0.
5. Add a top-of-file comment (after the shebang) that explicitly states:
   - This hook checks for PubNub key prefixes (pub-c-, sub-c-, sec-c-).
   - For real-world use, prefer a dedicated tool like gitleaks, truffleHog, or detect-secrets with a PubNub-specific rule.
   - If a key is found in repo history, the key must be rotated immediately; this hook only prevents new leaks.

Use ES module syntax.

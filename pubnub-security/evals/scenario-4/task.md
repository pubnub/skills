# Secure Document Sharing System with PubNub

## Problem/Feature Description

A legal services firm needs to share confidential case documents between attorneys and clients through their real-time platform built on PubNub. Documents include contracts, court filings, and privileged communications that must never be readable by any intermediary, including PubNub's own infrastructure. Additionally, when an attorney-client relationship ends or a case is closed, all sharing access for that case must be immediately revoked.

The firm's IT team needs two components: a server-side module that manages file sharing tokens and revocation, and a client-side module that handles the actual encrypted file upload and download. The security team has also asked for a script that demonstrates the full lifecycle -- granting access, sharing a file, and then revoking access -- with logging at each step so they can audit the flow. They specifically want the audit log to note any caveats or timing considerations around the revocation taking effect.

## Output Specification

Create the following files:

1. `file-sharing-server.js` -- Server-side module with:
   - A function to issue a file-sharing token for a specific case channel and user
   - A function to revoke file-sharing access when a case is closed
   - Appropriate security configuration

2. `file-sharing-client.js` -- Client-side module with:
   - A function to upload a file to a case channel, ensuring the file content is encrypted before transmission
   - A function to download and decrypt a file from a case channel
   - Proper PubNub initialization for file operations

3. `lifecycle-demo.js` -- A script that walks through the complete token lifecycle:
   - Grant access to a case channel
   - Inspect/parse the granted token to log its permissions
   - Simulate a file share
   - Revoke the token
   - Log the complete flow including any timing considerations about revocation

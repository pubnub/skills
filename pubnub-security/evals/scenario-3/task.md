# PubNub Access Control for a Multi-Tenant SaaS Platform

## Problem/Feature Description

A B2B SaaS company provides a real-time collaboration platform where each tenant (company) has its own workspace with channels for teams, direct messages between users, and an admin panel for workspace managers. They need a server-side Node.js module that manages PubNub access control for their multi-role system.

The platform has four roles: "guest" (can only view public announcements), "member" (can participate in team channels), "manager" (can also access analytics and manage team memberships), and "owner" (has full access including billing and admin channels). Each tenant's channels are prefixed with the tenant ID to ensure isolation.

The team wants a module that, given a user's role and tenant ID, issues the appropriate PubNub credentials with exactly the right level of access -- no more, no less. They want it to scale well as the number of channels per tenant grows.

## Output Specification

Create a file called `access-control.js` containing:
- A function `initializePubNub()` that creates and returns the server-side PubNub instance
- A function `issueCredentials(tenantId, userId, role)` that issues PubNub credentials appropriate for the given role
- The role-to-permission mapping should grant increasingly broader access from guest to owner
- Channel naming should follow a consistent convention that incorporates the tenant ID
- A function `revokeAccess(token)` that revokes a previously issued token

Also create a file called `channel-design.md` documenting the channel naming scheme and which roles can access which channel patterns.

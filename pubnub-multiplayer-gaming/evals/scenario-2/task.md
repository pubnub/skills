# Player Connection Resilience for a Co-op Dungeon Crawler

## Problem/Feature Description

A mobile game developer is building a co-op dungeon crawler where 2-4 players explore procedurally generated dungeons together in real-time. Mobile connections are unreliable -- players frequently lose connectivity when entering elevators, switching between WiFi and cellular, or experiencing brief network blips. The team needs a robust connection management system built on PubNub that gracefully handles these disruptions without ruining the game experience for other players.

When a player drops, the game should not immediately kick them or freeze. Instead, the system needs to give them a window to reconnect and, when they do come back, bring them up to speed with whatever happened while they were gone. The system also needs to detect the difference between a temporary network glitch and a player who has truly abandoned the session.

## Output Specification

Produce the following JavaScript files:

1. **`connection-manager.js`** -- A module that monitors player connections using PubNub presence, handles disconnect/reconnect/abandon lifecycle for each player, and notifies the rest of the game about connection state changes.
2. **`state-recovery.js`** -- A module that handles bringing a reconnected player back up to date with the current game state, including mechanisms to retrieve what they missed.
3. **`network-status-handler.js`** -- A module that listens for PubNub SDK connection status changes on the local client and takes appropriate recovery actions for each status category.

All files should be complete JavaScript modules with comments explaining the timing and threshold choices made.

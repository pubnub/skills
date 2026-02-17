# Real-Time Multiplayer Space Battle State Sync

## Problem/Feature Description

A game studio is building a real-time 2D space battle game where up to 8 players control ships in a shared arena. Ships move continuously, fire projectiles, and take damage. The game state includes player positions, health, scores, and active projectiles. The team needs the game state synchronization layer built on PubNub.

The main engineering challenge is efficiently keeping all clients in sync. The game state can grow fairly large with 8 players and dozens of projectiles, so naive approaches of broadcasting the entire game state every frame would blow through bandwidth limits. The team also needs to handle situations where two players fire at the same target simultaneously and both claim a kill, as well as players whose connections briefly drop and need to catch up.

## Output Specification

Produce the following JavaScript files:

1. **`state-sync.js`** -- The core state synchronization module. Should handle sending and receiving game state updates, applying incoming updates to local state, and managing update ordering.
2. **`conflict-resolver.js`** -- A module that handles concurrent state modifications, with at least two different resolution strategies appropriate for different types of game data (e.g., positions vs. scores).
3. **`delta-batcher.js`** -- A module that efficiently batches outgoing state changes to reduce the number of PubNub publish calls during high-frequency gameplay.

All files should be complete, well-commented JavaScript modules that work with the PubNub SDK.

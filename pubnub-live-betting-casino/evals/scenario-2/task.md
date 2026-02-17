# Real-Time Multiplayer Blackjack Platform

## Problem/Feature Description

An online casino operator wants to build a multiplayer blackjack experience where up to 7 players can sit at a virtual table and play against the dealer in real time. Each table runs independently, and the platform needs to support dozens of concurrent tables. Players see card deals, other players' actions, and settlement results instantly.

The game follows standard blackjack rules: players place bets, receive two cards, choose to hit, stand, or double down, and then the dealer plays. The game progresses through distinct phases (betting, dealing, player turns, dealer turn, settlement), and every connected client must see a consistent view of the table state at all times.

The casino operator wants the entire game flow built on PubNub, with the game logic running server-side so that players cannot manipulate outcomes. They also want a multi-table poker tournament structure where players are assigned to tables, compete, and see a live leaderboard of chip standings.

## Output Specification

Produce the following files:

1. **`blackjack-server.js`** -- Server-side game engine that:
   - Publishes full table state after each action
   - Processes player actions (hit, stand, double)
   - Manages game phases (betting, dealing, player_turn, dealer_turn, settlement)
   - Handles the card deck and dealing logic
   - Runs as a PubNub Function to ensure authoritative game state

2. **`blackjack-client.js`** -- Client-side module that:
   - Initializes the PubNub client with security configuration
   - Subscribes to the appropriate table channel
   - Sends player actions to the server
   - Receives and renders game state updates
   - Handles disconnection and reconnection scenarios

3. **`tournament-manager.js`** -- A module that:
   - Manages a multi-table tournament with lobby, table assignment, and leaderboard
   - Sets up the channel structure for the tournament
   - Assigns players to tables and publishes assignments
   - Updates and broadcasts leaderboard standings
   - Handles tournament-wide announcements

# E-Sports Tournament Viewer with Live Features

## Problem/Feature Description

An e-sports organization is building a tournament viewing platform where fans can watch live competitive matches, see real-time leaderboard updates, and interact through an in-game chat. The platform is built on PubNub and needs to support potentially thousands of spectators watching a single match, with a live-updating scoreboard and community interaction features.

The main concerns are: spectators should not be able to gain an unfair advantage by feeding live information to active players (stream sniping), the leaderboard must persist across sessions so tournament standings survive server restarts, the chat system needs to work across an international audience, and game actions submitted by players need validation to maintain competitive integrity.

## Output Specification

Produce the following JavaScript files:

1. **`spectator-manager.js`** -- A module that handles spectator subscriptions to live matches, including how game state is broadcast to viewers.
2. **`leaderboard.js`** -- A leaderboard system that persists scores, updates in real-time as matches complete, and can be queried by clients. Include both the server-side PubNub Function and the client-side subscription code.
3. **`game-chat.js`** -- An in-game chat module supporting both free-text and pre-defined communication that works for a global audience.
4. **`action-validator.js`** -- A server-side PubNub Function that validates incoming game actions to reject impossible or suspicious plays before they reach other clients.

All files should be complete JavaScript modules with comments explaining design decisions.

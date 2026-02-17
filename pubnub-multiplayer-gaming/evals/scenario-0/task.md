# Multiplayer Party Game Lobby System

## Problem/Feature Description

A small indie studio is building a browser-based party game platform where players can create and join game rooms for various mini-games (trivia, drawing, word games). They need a complete lobby and room management system built on PubNub that lets players browse available rooms, create new ones (public or private), join existing rooms, and have rooms transition through their natural lifecycle from creation to game completion.

The studio wants the networking layer to feel snappy -- players should see rooms appear in real-time and know instantly when someone joins or leaves. The system should also handle the basics of initializing PubNub with settings appropriate for gaming, where quick detection of player availability is important.

## Output Specification

Produce a JavaScript/Node.js module (or set of modules) that implements:

1. **`game-client.js`** -- PubNub initialization configured for a gaming context, plus a lobby system that lets players browse available game rooms in real-time.
2. **`room-manager.js`** -- Game room creation (public and private), joining, starting, and ending rooms. Rooms should store their metadata persistently (not just in local variables), track a lifecycle with multiple states, and handle capacity limits.

Both files should be complete, runnable code (assuming PubNub keys are provided). Include JSDoc comments or inline comments explaining key design decisions, especially around channel structure and configuration choices.

# Multi-Sport Live Scores React Dashboard

## Problem/Feature Description

A fantasy sports platform wants to add a live scores dashboard to their web app. The dashboard should show all active games across NFL, NBA, and EPL (English Premier League) in a single unified view, grouped by league. Fans using the platform typically have it open on a second monitor or phone during game days, so it needs to handle hours-long sessions where games start, progress through various stages, and end.

The engineering team has tried building this before using individual channel subscriptions per game, but it became unwieldy with dozens of games on a Sunday afternoon. They need a cleaner architecture that can follow entire leagues without manually tracking every game. The dashboard also needs to handle the reality that fans will lose WiFi or cell signal periodically and the UI should not break or show confusing error messages when that happens.

## Output Specification

Produce the following files:

1. `sports-dashboard.jsx` -- A React component that initializes PubNub, subscribes to multiple leagues, renders a grouped game list, and properly cleans up resources
2. `dashboard-state.js` -- A state management module (plain JS class) that tracks active games, processes incoming score updates, and provides sorted game lists for the UI layer

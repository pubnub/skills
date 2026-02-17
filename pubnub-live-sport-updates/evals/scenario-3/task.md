# Multi-Sport Game Event Publisher

## Problem/Feature Description

A live sports data aggregator needs a unified event publishing service that can handle games across five different sports: NFL, NBA, NHL, soccer (EPL), and MLB. Their data provider sends raw events for all sports through a single stream, and the publisher must format and route each event correctly for downstream clients.

The challenge is that each sport has different terminology for its time periods (quarters, periods, halves, innings), different scoring mechanics, and different levels of event importance. A goal in soccer is as significant as a touchdown in football, but a corner kick is relatively minor. The publishing service must handle these differences correctly so that client applications across all five sports render the right labels and receive the right events on the right channels.

The service also needs to handle game lifecycle events -- games starting, reaching intermissions, going to overtime, and ending -- as separate trackable events that clients can use to update their UI state.

## Output Specification

Produce a JavaScript file called `multi-sport-publisher.js` that:

1. Implements a game event publisher class that accepts events for any of the five supported sports
2. Formats period labels appropriately for each sport
3. Routes events to the correct channels based on sport and event significance
4. Publishes game lifecycle status changes as trackable events
5. Ensures every published event carries the fields needed for clients to order and deduplicate them

Include a helper function or module for period label formatting that covers all five sports.

# Route a Threshold-Triggered Anomaly Detection Question

## Problem/Feature Description

A trust-and-safety engineer at a social platform asks:

> "We're seeing waves of spam from accounts that suddenly post to dozens of channels in a few seconds. I want PubNub to flag any account that publishes to more than 20 channels in 60 seconds and automatically mute them. We're already using PubNub for the underlying messaging. Where in PubNub do I configure this kind of rate-anomaly detection and automatic mute action? Just point me at the right place — no code yet."

This is an analytics + threshold-triggered automation question. The user describes a real-time KPI (channels per user per minute) with a threshold that triggers an action (mute). PubNub's analytics platform owns this; raw Functions + custom logic is the wrong choice per the disambiguation heuristics.

## Output Specification

Produce exactly one handoff block in the canonical template format:

```text
Intent      : ...
Skill       : ...
MCP tool    : ...
Docs source : ...
Next step   : ...
```

Do not include implementation code. Do not chain multiple handoffs.

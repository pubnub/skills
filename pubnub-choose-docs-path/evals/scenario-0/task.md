# Route a Customer Support Chat Question

## Problem/Feature Description

A developer from a SaaS company writes in:

> "We're rolling out an embedded customer-support chat in our web app. We need 1:1 conversations between a customer and an agent, typing indicators while the agent is replying, and emoji reactions on messages so agents can quickly acknowledge things. We picked PubNub. Where should I start? Which docs and tools should I be using?"

The user has described a clearly-bounded set of chat features but has not picked a specific SDK or tool yet. They want a single, actionable handoff so they know which specialist skill, which MCP tool, and which documentation surface to use next.

## Output Specification

Produce exactly one handoff block in the canonical template format. Use exactly these five labeled lines (in this order, with the labels spelled exactly as shown):

```text
Intent      : ...
Skill       : ...
MCP tool    : ...
Docs source : ...
Next step   : ...
```

Do not include any other content (no headings, no code snippets, no implementation, no extra commentary after the block). Do not chain multiple handoffs.

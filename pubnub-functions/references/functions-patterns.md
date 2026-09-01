# PubNub Functions 2.0 Common Patterns

## Pattern 1: Distributed Counter

Perfect for voting, likes, view tracking.

```javascript
// Trigger: Before Signal on "vote.*"
export default async (request) => {
  const db = require('kvstore');
  const pubnub = require('pubnub');

  try {
    const { itemId, userId } = request.message;
    const voteKey = `votes:${itemId}`;
    const userVoteKey = `voted:${itemId}:${userId}`;

    // Check for duplicate votes
    const hasVoted = await db.get(userVoteKey);
    if (hasVoted) {
      request.message.error = 'already_voted';
      return request.ok();
    }

    // Atomic increment
    const newCount = await db.incrCounter(voteKey);

    // Mark user as voted (24 hour TTL)
    await db.set(userVoteKey, true, 1440);

    // Enrich message with new count
    request.message.voteCount = newCount;
    request.message.success = true;

    return request.ok();
  } catch (error) {
    console.error('Vote error:', error);
    return request.abort();
  }
};
```

## Pattern 2: Message Transformation

Transform, validate, or enrich messages.

```javascript
// Trigger: Before Publish on "chat.*"
export default async (request) => {
  const db = require('kvstore');

  try {
    const message = request.message;

    // Validate required fields
    if (!message.userId || !message.text) {
      console.error('Missing required fields');
      return request.abort();
    }

    // Get user profile from cache
    const userProfile = await db.get(`user:${message.userId}`);

    // Transform message
    request.message = {
      id: `msg_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
      text: message.text,
      author: {
        id: message.userId,
        name: userProfile?.name || 'Unknown',
        avatar: userProfile?.avatar || null
      },
      timestamp: new Date().toISOString(),
      metadata: {
        platform: message.platform || 'unknown',
        version: '2.0'
      }
    };

    return request.ok();
  } catch (error) {
    console.error('Transform error:', error);
    return request.abort();
  }
};
```

## Pattern 3: Content Moderation

Filter or flag inappropriate content.

```javascript
// Trigger: Before Publish on "chat.*"
export default async (request) => {
  const xhr = require('xhr');
  const vault = require('vault');

  try {
    const text = request.message.text;
    if (!text) return request.ok();

    // Get moderation API key
    const apiKey = await vault.get('moderation_api_key');

    // Call moderation service
    const response = await xhr.fetch('https://api.moderation.example.com/check', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${apiKey}`
      },
      body: JSON.stringify({ text })
    });

    const result = JSON.parse(response.body);

    if (result.flagged) {
      // Block message if toxic
      if (result.score > 0.9) {
        console.log('Blocked toxic message');
        return request.abort();
      }

      // Warn but allow if borderline
      request.message.moderation = {
        flagged: true,
        score: result.score,
        categories: result.categories
      };
    }

    return request.ok();
  } catch (error) {
    console.error('Moderation error:', error);
    // Allow message on error (fail open)
    return request.ok();
  }
};
```

## Pattern 4: Webhook Integration

Send data to external services.

```javascript
// Trigger: After Publish on "orders.*"
export default async (request) => {
  const xhr = require('xhr');
  const vault = require('vault');

  try {
    const order = request.message;

    // Slack notification
    const slackUrl = await vault.get('slack_webhook');
    await xhr.fetch(slackUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        text: `New order #${order.id} - $${order.total}`,
        attachments: [{
          color: 'good',
          fields: [
            { title: 'Customer', value: order.customer, short: true },
            { title: 'Items', value: order.itemCount, short: true }
          ]
        }]
      })
    });

    // CRM integration
    const crmApiKey = await vault.get('crm_api_key');
    await xhr.fetch('https://api.crm.example.com/events', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${crmApiKey}`
      },
      body: JSON.stringify({
        event: 'order_placed',
        customerId: order.customerId,
        orderId: order.id,
        value: order.total
      })
    });

    return request.ok();
  } catch (error) {
    console.error('Webhook error:', error);
    // Don't fail the message, just log error
    return request.ok();
  }
};
```

## Pattern 5: REST API Endpoint

Build serverless APIs.

```javascript
// Trigger: On Request
export default async (request, response) => {
  const db = require('kvstore');

  // CORS headers
  response.headers['Access-Control-Allow-Origin'] = '*';
  response.headers['Access-Control-Allow-Methods'] = 'GET, POST, PUT, DELETE';
  response.headers['Access-Control-Allow-Headers'] = 'Content-Type';

  if (request.method === 'OPTIONS') {
    return response.send('', 200);
  }

  try {
    const userId = request.params.userId;

    switch (request.method) {
      case 'GET':
        const user = await db.get(`user:${userId}`);
        if (!user) {
          return response.send({ error: 'User not found' }, 404);
        }
        return response.send(user, 200);

      case 'POST':
      case 'PUT':
        const body = await request.json();
        if (!body.name) {
          return response.send({ error: 'Name required' }, 400);
        }
        const userData = {
          id: userId,
          name: body.name,
          email: body.email,
          updatedAt: new Date().toISOString()
        };
        await db.set(`user:${userId}`, userData, 43200);  // 30 days
        return response.send({ message: 'User saved', user: userData }, 200);

      case 'DELETE':
        await db.removeItem(`user:${userId}`);
        return response.send({ message: 'User deleted' }, 200);

      default:
        return response.send({ error: 'Method not allowed' }, 405);
    }
  } catch (error) {
    console.error('API error:', error);
    return response.send({ error: 'Internal server error' }, 500);
  }
};
```

## Pattern 6: Periodic Aggregation

Scheduled data collection and broadcasting.

```javascript
// Trigger: On Interval (every 60 seconds)
export default async (event) => {
  const db = require('kvstore');
  const pubnub = require('pubnub');

  try {
    // Get all vote counters
    const keys = await db.getKeys();
    const voteKeys = keys.filter(k => k.startsWith('votes:'));

    const results = {};
    for (const key of voteKeys) {
      const itemId = key.replace('votes:', '');
      const count = await db.getCounter(key);
      results[itemId] = count;
    }

    // Publish aggregated results
    await pubnub.publish({
      channel: 'leaderboard',
      message: {
        type: 'vote_summary',
        results,
        timestamp: Date.now()
      }
    });

    console.log(`Published results for ${Object.keys(results).length} items`);
    return event.ok();
  } catch (error) {
    console.error('Aggregation error:', error);
    return event.abort();
  }
};
```

## Pattern 7: Rate Limiting

Protect against abuse.

```javascript
// Trigger: Before Publish on "api.*"
export default async (request) => {
  const db = require('kvstore');

  try {
    const userId = request.message.userId;
    const tier = request.message.tier || 'free';

    const limits = {
      free: { requests: 100, window: 3600 },
      pro: { requests: 1000, window: 3600 }
    };

    const limit = limits[tier] || limits.free;
    const windowId = Math.floor(Date.now() / (limit.window * 1000));
    const key = `rate:${userId}:${windowId}`;

    const count = await db.incrCounter(key);

    if (count > limit.requests) {
      console.log(`Rate limit exceeded: ${userId}`);
      request.message.rateLimited = true;
      request.message.retryAfter = limit.window;
      return request.abort();
    }

    request.message.rateLimit = {
      remaining: limit.requests - count,
      limit: limit.requests,
      reset: (windowId + 1) * limit.window * 1000
    };

    return request.ok();
  } catch (error) {
    console.error('Rate limit error:', error);
    // Fail open on errors
    return request.ok();
  }
};
```

## Pattern 8: Authentication Middleware

Validate tokens before processing.

```javascript
// Trigger: Before Publish on "secure.*"
export default async (request) => {
  const jwt = require('jwt');
  const vault = require('vault');

  try {
    const token = request.message.authToken;
    if (!token) {
      console.error('Missing auth token');
      return request.abort();
    }

    const secret = await vault.get('jwt_secret');
    const decoded = jwt.verify(token, secret);

    // Add user context to message
    request.message.auth = {
      userId: decoded.userId,
      role: decoded.role,
      verified: true
    };

    // Remove raw token from message
    delete request.message.authToken;

    return request.ok();
  } catch (error) {
    console.error('Auth error:', error.message);
    return request.abort();
  }
};
```

## Best Practices Summary

1. **Use vault for secrets** - Never hardcode API keys
2. **Implement try/catch** - Always handle errors gracefully
3. **Use async/await** - Avoid .then()/.catch() chains
4. **Consider operation limits** - Max 3 per execution
5. **Log appropriately** - Use console.log/error for debugging
6. **Fail gracefully** - Decide whether to block or allow on errors
7. **Validate inputs** - Check message structure before processing
8. **Use fire for analytics** - Avoid triggering subscriber delivery

# Rate Limiting: Protecting Your API

> **If you don't limit requests, someone else will limit them for you—by crashing your servers.**

It's 2 PM on a Tuesday. Your API starts slowing down. Then it grinds to a halt. You check the logs: one IP address has made 10 million requests in the last hour. A misconfigured script is hammering your servers. Your legitimate users can't access anything. Your app is effectively down.

This is preventable with rate limiting—a critical defense against abuse, accidents, and attacks.

## Why Rate Limiting Matters

**Prevents Abuse**:
- Malicious actors trying to overwhelm your service (DDoS)
- Scrapers stealing your data
- Bots creating fake accounts
- Brute force password attacks

**Ensures Fair Usage**:
- Prevents one user from consuming all resources
- Guarantees availability for all users
- Protects shared infrastructure

**Controls Costs**:
- Limits compute/bandwidth consumption
- Prevents unexpected infrastructure bills
- Makes capacity planning predictable

**Improves Reliability**:
- Prevents cascading failures
- Maintains response times under load
- Enables graceful degradation

Without rate limiting, your API is vulnerable to both malicious attacks and honest mistakes.

## Rate Limiting Algorithms

### 1. Fixed Window

**How It Works**: Allow N requests per fixed time window.

```
Window 1 (0:00-0:59): |||||||||| (10 requests) ✅
Window 2 (1:00-1:59): ||||| (5 requests) ✅
Window 3 (2:00-2:59): |||||||||||| (12 requests) ❌ RATE LIMITED
```

**Implementation**:
```javascript
const redis = require('redis');
const client = redis.createClient();

async function fixedWindowRateLimit(userId, limit = 100, windowSeconds = 3600) {
  const key = `rate_limit:${userId}:${Math.floor(Date.now() / 1000 / windowSeconds)}`;
  
  const current = await client.incr(key);
  
  if (current === 1) {
    // First request in window, set expiration
    await client.expire(key, windowSeconds);
  }
  
  if (current > limit) {
    throw new Error('Rate limit exceeded');
  }
  
  return {
    allowed: true,
    remaining: limit - current
  };
}
```

**Pros**:
- Simple to implement
- Memory efficient
- Easy to understand

**Cons**:
- Allows bursts at window boundaries (100 requests at 0:59, 100 more at 1:00)
- Not perfectly fair

**When to Use**: Simple rate limiting needs, internal APIs.

### 2. Sliding Window

**How It Works**: Count requests in a rolling time window.

```
Current time: 12:30
Window: Last 60 minutes (11:30-12:30)
Requests in window: Count all requests from 11:30 to now
```

**Implementation**:
```javascript
async function slidingWindowRateLimit(userId, limit = 100, windowMs = 3600000) {
  const key = `rate_limit:${userId}`;
  const now = Date.now();
  const windowStart = now - windowMs;
  
  // Remove old entries
  await client.zremrangebyscore(key, 0, windowStart);
  
  // Count requests in window
  const count = await client.zcard(key);
  
  if (count >= limit) {
    const oldestEntry = await client.zrange(key, 0, 0, 'WITHSCORES');
    const resetTime = parseInt(oldestEntry[1]) + windowMs;
    
    throw {
      error: 'Rate limit exceeded',
      resetAt: new Date(resetTime),
      retryAfter: Math.ceil((resetTime - now) / 1000)
    };
  }
  
  // Add current request
  await client.zadd(key, now, `${now}:${Math.random()}`);
  await client.expire(key, Math.ceil(windowMs / 1000));
  
  return {
    allowed: true,
    remaining: limit - count - 1
  };
}
```

**Pros**:
- Smoother than fixed window
- No burst issues at boundaries
- Fair distribution

**Cons**:
- More complex
- Higher memory usage (stores all timestamps)
- Slightly slower

**When to Use**: Public APIs, strict rate limiting requirements.

### 3. Token Bucket

**How It Works**: Bucket holds tokens, each request consumes a token. Tokens refill at a steady rate.

```
Bucket capacity: 100 tokens
Refill rate: 10 tokens/second

Request at t=0: 100 tokens available ✅ (99 remaining)
Request at t=1: 109 tokens available ✅ (108 remaining)
100 requests at t=2: 118 tokens available ✅ (18 remaining)
Request at t=3: 28 tokens available ✅ (27 remaining)
```

**Implementation**:
```javascript
async function tokenBucketRateLimit(userId, capacity = 100, refillRate = 10) {
  const key = `rate_limit:bucket:${userId}`;
  
  const data = await client.get(key);
  let tokens = capacity;
  let lastRefill = Date.now();
  
  if (data) {
    const parsed = JSON.parse(data);
    const now = Date.now();
    const timePassed = (now - parsed.lastRefill) / 1000;
    
    // Refill tokens based on time passed
    tokens = Math.min(
      capacity,
      parsed.tokens + timePassed * refillRate
    );
    lastRefill = now;
  }
  
  if (tokens < 1) {
    const timeUntilToken = (1 - tokens) / refillRate;
    
    throw {
      error: 'Rate limit exceeded',
      retryAfter: Math.ceil(timeUntilToken)
    };
  }
  
  // Consume one token
  tokens -= 1;
  
  await client.setex(
    key,
    Math.ceil(capacity / refillRate),
    JSON.stringify({ tokens, lastRefill })
  );
  
  return {
    allowed: true,
    remaining: Math.floor(tokens)
  };
}
```

**Pros**:
- Allows controlled bursts
- Smooth rate limiting
- Flexible (can adjust refill rate)

**Cons**:
- Most complex to implement
- Requires careful tuning

**When to Use**: APIs that need to allow bursts but maintain overall rate limits.

### 4. Leaky Bucket

**How It Works**: Requests queue up, processed at constant rate.

Similar to token bucket but requests wait in queue instead of being rejected immediately.

**When to Use**: When you want to smooth out traffic spikes, message queues.

## HTTP Headers

Communicate rate limit status to clients via HTTP headers:

```
X-RateLimit-Limit: 1000          # Total requests allowed per window
X-RateLimit-Remaining: 237       # Requests remaining in current window
X-RateLimit-Reset: 1704729600    # Unix timestamp when limit resets
Retry-After: 60                  # Seconds to wait before retrying (429 only)
```

**Implementation**:
```javascript
app.use(async (req, res, next) => {
  try {
    const userId = req.user?.id || req.ip;
    const result = await rateLimit(userId);
    
    // Add headers to all responses
    res.set({
      'X-RateLimit-Limit': 1000,
      'X-RateLimit-Remaining': result.remaining,
      'X-RateLimit-Reset': result.resetAt
    });
    
    next();
  } catch (err) {
    res.status(429).set({
      'X-RateLimit-Limit': 1000,
      'X-RateLimit-Remaining': 0,
      'X-RateLimit-Reset': err.resetAt,
      'Retry-After': err.retryAfter
    }).json({
      error: {
        type: 'rate_limit_exceeded',
        message: 'Too many requests',
        retry_after: err.retryAfter,
        reset_at: new Date(err.resetAt).toISOString()
      }
    });
  }
});
```

## Multi-Tier Rate Limiting

Different limits for different user types:

```javascript
const RATE_LIMITS = {
  anonymous: { requests: 100, window: 3600 },    // 100 requests/hour
  authenticated: { requests: 1000, window: 3600 }, // 1000 requests/hour
  premium: { requests: 10000, window: 3600 },      // 10K requests/hour
  enterprise: { requests: 100000, window: 3600 }   // 100K requests/hour
};

function getRateLimit(user) {
  if (!user) return RATE_LIMITS.anonymous;
  if (user.plan === 'enterprise') return RATE_LIMITS.enterprise;
  if (user.plan === 'premium') return RATE_LIMITS.premium;
  return RATE_LIMITS.authenticated;
}

app.use(async (req, res, next) => {
  const limits = getRateLimit(req.user);
  const userId = req.user?.id || req.ip;
  
  const result = await rateLimit(userId, limits.requests, limits.window);
  
  // Add headers and proceed...
});
```

## Per-Endpoint Rate Limiting

Apply different limits to different endpoints:

```javascript
const ENDPOINT_LIMITS = {
  '/api/auth/login': { requests: 5, window: 900 },     // 5 requests/15min
  '/api/search': { requests: 100, window: 60 },        // 100 requests/min
  '/api/users': { requests: 1000, window: 3600 },      // 1000 requests/hour
  default: { requests: 1000, window: 3600 }
};

app.use(async (req, res, next) => {
  const endpoint = req.path;
  const limits = ENDPOINT_LIMITS[endpoint] || ENDPOINT_LIMITS.default;
  
  const key = `${req.user?.id || req.ip}:${endpoint}`;
  const result = await rateLimit(key, limits.requests, limits.window);
  
  // Proceed...
});
```

## Rate Limiting Strategies

### By IP Address

```javascript
app.use(async (req, res, next) => {
  const ip = req.ip || req.connection.remoteAddress;
  await rateLimit(ip);
  next();
});
```

**Pros**: Works for anonymous users
**Cons**: Shared IPs (NAT, proxies) affect multiple users

### By User ID

```javascript
app.use(async (req, res, next) => {
  if (req.user) {
    await rateLimit(`user:${req.user.id}`);
  }
  next();
});
```

**Pros**: Fair per user
**Cons**: Doesn't protect unauthenticated endpoints

### By API Key

```javascript
app.use(async (req, res, next) => {
  const apiKey = req.headers['x-api-key'];
  if (apiKey) {
    await rateLimit(`key:${apiKey}`);
  }
  next();
});
```

**Pros**: Precise control, works for service accounts
**Cons**: Requires API key infrastructure

### Hybrid Approach

```javascript
app.use(async (req, res, next) => {
  // Use user ID if authenticated, otherwise IP
  const identifier = req.user?.id || req.ip;
  await rateLimit(identifier);
  next();
});
```

## Distributed Rate Limiting

For multi-server deployments, use shared state:

```javascript
// Using Redis
const Redis = require('ioredis');
const redis = new Redis({
  host: 'redis.example.com',
  port: 6379
});

async function distributedRateLimit(key, limit, windowSeconds) {
  const current = await redis.incr(key);
  
  if (current === 1) {
    await redis.expire(key, windowSeconds);
  }
  
  return {
    allowed: current <= limit,
    remaining: Math.max(0, limit - current)
  };
}
```

## Testing Rate Limits

```javascript
describe('Rate Limiting', () => {
  it('allows requests within limit', async () => {
    for (let i = 0; i < 100; i++) {
      const response = await request(app).get('/api/users');
      expect(response.status).toBe(200);
    }
  });
  
  it('blocks requests exceeding limit', async () => {
    // Make 101 requests
    const requests = [];
    for (let i = 0; i < 101; i++) {
      requests.push(request(app).get('/api/users'));
    }
    
    const responses = await Promise.all(requests);
    
    // At least one should be rate limited
    const rateLimited = responses.filter(r => r.status === 429);
    expect(rateLimited.length).toBeGreaterThan(0);
  });
  
  it('includes rate limit headers', async () => {
    const response = await request(app).get('/api/users');
    
    expect(response.headers).toHaveProperty('x-ratelimit-limit');
    expect(response.headers).toHaveProperty('x-ratelimit-remaining');
    expect(response.headers).toHaveProperty('x-ratelimit-reset');
  });
  
  it('includes Retry-After when rate limited', async () => {
    // Exceed rate limit
    for (let i = 0; i < 101; i++) {
      await request(app).get('/api/users');
    }
    
    const response = await request(app).get('/api/users');
    expect(response.status).toBe(429);
    expect(response.headers).toHaveProperty('retry-after');
  });
});
```

## Best Practices

- [ ] Use distributed rate limiting (Redis) for multi-server setups
- [ ] Return 429 status code when limit exceeded
- [ ] Include rate limit headers in all responses
- [ ] Provide `Retry-After` header with 429 responses
- [ ] Apply stricter limits to expensive operations (auth, search)
- [ ] Allow burst traffic but limit sustained load
- [ ] Log rate limit violations for monitoring
- [ ] Provide clear documentation of rate limits
- [ ] Consider different limits for different user tiers
- [ ] Test rate limiting under load

## Summary

Rate limiting is essential API infrastructure that:
- Protects against abuse and attacks
- Ensures fair resource allocation
- Controls costs and improves reliability

Choose your algorithm based on needs:
- **Fixed window**: Simple, good enough for most cases
- **Sliding window**: Fairer, prevents boundary bursts
- **Token bucket**: Flexible, allows controlled bursts

Always communicate limits clearly through HTTP headers and documentation.

## Further Reading

- **RFC 6585**: Additional HTTP Status Codes (429)
- **NGINX Rate Limiting**: Implementation patterns
- **Redis**: Distributed rate limiting
- **API Gateway**: Cloud rate limiting solutions

---

**Next**: Support multiple data formats in [Content Negotiation](./content-negotiation.md).

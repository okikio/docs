# Idempotency: Safe Request Retries

> **Network failures happen. Your API should handle retries gracefully.**

A user clicks "Pay Now" on your checkout page. The request times out. Anxious, they click again. And again. You charge their card three times. They're furious. You issue refunds. Everyone loses.

This is preventable with idempotency—the ability to safely retry operations without unintended side effects.

## What is Idempotency?

An operation is idempotent if performing it multiple times has the same effect as performing it once:

```
f(f(x)) = f(x)
```

**Examples**:

**Idempotent Operations**:
- Setting a value: `user.name = "Alice"` (always results in name being "Alice")
- Deleting a resource: `DELETE /api/users/123` (user deleted, subsequent calls still result in no user)
- Getting data: `GET /api/users/123` (no side effects)

**Non-Idempotent Operations**:
- Creating a resource: `POST /api/users` (creates a new user each time)
- Incrementing a counter: `count++` (increases by 1 each time)
- Charging a payment: `charge(card, $100)` (charges $100 each time)

## HTTP Methods and Idempotency

By HTTP spec (RFC 7231), methods have defined idempotency characteristics:

**Idempotent by Default**:
- `GET` - Retrieves data, no side effects
- `PUT` - Replaces resource at URL, same result
- `DELETE` - Deletes resource, same result
- `HEAD`, `OPTIONS` - No side effects

**Not Idempotent**:
- `POST` - Creates new resource each call
- `PATCH` - May have different effects

The challenge: making POST operations idempotent.

## Idempotency Keys

The solution: clients provide a unique key with each request. If the same key is sent again, return the original response without re-processing.

**Flow**:
```
1. Client generates unique key (UUID)
2. Client sends request with Idempotency-Key header
3. Server checks if key was seen before
   - If yes: Return stored response
   - If no: Process request, store result with key
4. Client can retry safely with same key
```

**Implementation**:
```javascript
const idempotencyStore = new Map(); // Use Redis in production

app.post('/api/payments', async (req, res) => {
  const idempotencyKey = req.headers['idempotency-key'];
  
  if (!idempotencyKey) {
    return res.status(400).json({
      error: {
        type: 'missing_idempotency_key',
        message: 'Idempotency-Key header is required'
      }
    });
  }
  
  // Check if we've seen this key before
  const cached = idempotencyStore.get(idempotencyKey);
  
  if (cached) {
    // Return cached response
    return res.status(cached.status).json(cached.body);
  }
  
  try {
    // Process payment
    const payment = await processPayment(req.body);
    
    const response = {
      status: 201,
      body: { payment }
    };
    
    // Store response with key (24 hour TTL)
    idempotencyStore.set(idempotencyKey, response);
    setTimeout(() => idempotencyStore.delete(idempotencyKey), 24 * 60 * 60 * 1000);
    
    res.status(201).json(response.body);
  } catch (err) {
    // Store error response too
    const errorResponse = {
      status: 500,
      body: { error: err.message }
    };
    
    idempotencyStore.set(idempotencyKey, errorResponse);
    res.status(500).json(errorResponse.body);
  }
});
```

## Using Redis for Idempotency

```javascript
const Redis = require('ioredis');
const redis = new Redis();

async function handleIdempotentRequest(key, handler) {
  // Check cache
  const cached = await redis.get(`idempotency:${key}`);
  
  if (cached) {
    const response = JSON.parse(cached);
    return response;
  }
  
  // Process request
  const response = await handler();
  
  // Store result (24 hour expiry)
  await redis.setex(
    `idempotency:${key}`,
    86400,
    JSON.stringify(response)
  );
  
  return response;
}

app.post('/api/orders', async (req, res) => {
  const idempotencyKey = req.headers['idempotency-key'];
  
  if (!idempotencyKey) {
    return res.status(400).json({ error: 'Idempotency key required' });
  }
  
  const response = await handleIdempotentRequest(
    idempotencyKey,
    async () => {
      const order = await createOrder(req.body);
      return { status: 201, body: { order } };
    }
  );
  
  res.status(response.status).json(response.body);
});
```

## Generating Idempotency Keys

**Client-Side Generation** (Recommended):
```javascript
const { v4: uuidv4 } = require('uuid');

async function createPayment(paymentData) {
  const idempotencyKey = uuidv4();
  
  const response = await fetch('/api/payments', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Idempotency-Key': idempotencyKey
    },
    body: JSON.stringify(paymentData)
  });
  
  return response.json();
}
```

**Key Requirements**:
- Must be unique per request
- Should be unpredictable (UUIDs work well)
- Client should generate before sending
- Same key for retries of the same operation

## Concurrent Requests

Handle race conditions when multiple requests arrive with the same key:

```javascript
const locks = new Map();

async function withLock(key, handler) {
  // Wait if another request is processing this key
  while (locks.has(key)) {
    await new Promise(resolve => setTimeout(resolve, 100));
  }
  
  locks.set(key, true);
  
  try {
    return await handler();
  } finally {
    locks.delete(key);
  }
}

app.post('/api/payments', async (req, res) => {
  const idempotencyKey = req.headers['idempotency-key'];
  
  await withLock(idempotencyKey, async () => {
    const cached = await redis.get(`idempotency:${idempotencyKey}`);
    
    if (cached) {
      return res.json(JSON.parse(cached));
    }
    
    // Process payment...
  });
});
```

## Request Body Validation

Ensure the same idempotency key is used with the same request body:

```javascript
const crypto = require('crypto');

function hashRequest(body) {
  return crypto
    .createHash('sha256')
    .update(JSON.stringify(body))
    .digest('hex');
}

app.post('/api/payments', async (req, res) => {
  const idempotencyKey = req.headers['idempotency-key'];
  const requestHash = hashRequest(req.body);
  
  const stored = await redis.get(`idempotency:${idempotencyKey}`);
  
  if (stored) {
    const { hash, response } = JSON.parse(stored);
    
    // Verify request body matches
    if (hash !== requestHash) {
      return res.status(422).json({
        error: {
          type: 'idempotency_mismatch',
          message: 'Request body differs from original request with this key'
        }
      });
    }
    
    return res.status(response.status).json(response.body);
  }
  
  // Process and store with hash
  const payment = await processPayment(req.body);
  
  await redis.setex(
    `idempotency:${idempotencyKey}`,
    86400,
    JSON.stringify({
      hash: requestHash,
      response: { status: 201, body: { payment } }
    })
  );
  
  res.status(201).json({ payment });
});
```

## Expiration and Cleanup

```javascript
// Set reasonable expiration (24-72 hours)
const IDEMPOTENCY_TTL = 24 * 60 * 60; // 24 hours

await redis.setex(
  `idempotency:${key}`,
  IDEMPOTENCY_TTL,
  JSON.stringify(response)
);

// Periodic cleanup for Map-based storage
setInterval(() => {
  const now = Date.now();
  for (const [key, value] of idempotencyStore.entries()) {
    if (now - value.timestamp > IDEMPOTENCY_TTL * 1000) {
      idempotencyStore.delete(key);
    }
  }
}, 60 * 60 * 1000); // Clean up every hour
```

## When to Use Idempotency Keys

**Critical Operations** (Always):
- Financial transactions (payments, refunds)
- Order creation
- Account modifications
- Sending emails/notifications
- Inventory updates

**Optional**:
- Read operations (already idempotent)
- Truly idempotent writes (PUT, DELETE)
- Internal APIs with retry logic

**Not Needed**:
- GET requests
- Bulk operations with unique IDs
- Operations designed to be non-idempotent

## Stripe's Pattern

Stripe has excellent idempotency implementation. Learn from it:

```javascript
// Client
const stripe = require('stripe')(process.env.STRIPE_KEY);

const charge = await stripe.charges.create(
  {
    amount: 2000,
    currency: 'usd',
    source: 'tok_visa'
  },
  {
    idempotencyKey: 'order_123_payment'  // Client generates
  }
);

// If retry happens with same key, Stripe returns original charge
```

## Testing Idempotency

```javascript
describe('Idempotency', () => {
  it('returns same response for duplicate requests', async () => {
    const idempotencyKey = 'test_key_123';
    const requestBody = { amount: 1000 };
    
    // First request
    const response1 = await request(app)
      .post('/api/payments')
      .set('Idempotency-Key', idempotencyKey)
      .send(requestBody);
    
    expect(response1.status).toBe(201);
    const paymentId = response1.body.payment.id;
    
    // Second request with same key
    const response2 = await request(app)
      .post('/api/payments')
      .set('Idempotency-Key', idempotencyKey)
      .send(requestBody);
    
    expect(response2.status).toBe(201);
    expect(response2.body.payment.id).toBe(paymentId);
    
    // Verify only one payment was created
    const payments = await db.payments.find({});
    expect(payments.length).toBe(1);
  });
  
  it('rejects different body with same key', async () => {
    const idempotencyKey = 'test_key_456';
    
    await request(app)
      .post('/api/payments')
      .set('Idempotency-Key', idempotencyKey)
      .send({ amount: 1000 });
    
    const response = await request(app)
      .post('/api/payments')
      .set('Idempotency-Key', idempotencyKey)
      .send({ amount: 2000 });  // Different amount
    
    expect(response.status).toBe(422);
    expect(response.body.error.type).toBe('idempotency_mismatch');
  });
});
```

## Best Practices

- [ ] Require idempotency keys for critical operations
- [ ] Use UUIDs or similar for keys
- [ ] Store responses for 24-72 hours
- [ ] Validate request body matches cached request
- [ ] Handle concurrent requests with same key
- [ ] Return original status code and body for duplicates
- [ ] Clean up expired keys
- [ ] Document idempotency behavior
- [ ] Use Redis or similar for distributed systems
- [ ] Test retry scenarios thoroughly

## Summary

Idempotency prevents duplicate operations from network retries:

- Critical for payments, orders, and state changes
- Implement with Idempotency-Key header
- Store request/response pairs
- Return cached responses for duplicate keys
- Validate request bodies match
- Set appropriate expiration (24+ hours)

This protects users from duplicate charges and your system from inconsistent state.

## Further Reading

- **RFC 7231**: HTTP Idempotent Methods
- **Stripe API**: Idempotency implementation
- **Two-Phase Commit**: For distributed transactions

---

**Next**: Optimize performance with [Caching](./caching.md).

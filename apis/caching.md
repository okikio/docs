# Caching: HTTP Performance Optimization

> **The fastest request is the one you never make.**

Your API is slow. Every request hits the database. The same data is fetched thousands of times per second. Servers struggle. Response times climb. Users complain. Your infrastructure bill skyrockets.

Then you implement caching. Response times drop from 500ms to 5ms. Server load decreases by 90%. Users are happy. Costs plummet.

Caching is one of the highest-impact optimizations you can make.

## Why Caching Matters

**Performance**:
- Faster response times (cached responses are milliseconds vs. seconds)
- Reduced latency (content closer to users)
- Better user experience

**Scalability**:
- Handle more requests with same infrastructure
- Reduce database load by 80-95%
- Enable horizontal scaling

**Cost**:
- Lower compute costs (fewer server resources)
- Reduced bandwidth (CDN edge caching)
- Cheaper than scaling servers

**Reliability**:
- Serve stale content if backend is down
- Graceful degradation under load
- Better availability

## HTTP Caching Basics (RFC 7234)

HTTP has built-in caching mechanisms using headers.

### Cache-Control Header

The primary caching directive:

```
Cache-Control: public, max-age=3600
```

**Directives**:

- `public` - Can be cached by any cache (CDN, proxy, browser)
- `private` - Only browser cache (not shared caches)
- `no-cache` - Must revalidate with server before use
- `no-store` - Don't cache at all (sensitive data)
- `max-age=N` - Cache for N seconds
- `s-maxage=N` - Shared cache max age (overrides max-age for CDNs)
- `must-revalidate` - Strict revalidation when stale
- `stale-while-revalidate=N` - Serve stale while fetching fresh
- `immutable` - Never changes (perfect for versioned assets)

**Examples**:
```javascript
// Cache for 1 hour in any cache
res.set('Cache-Control', 'public, max-age=3600');

// Cache in browser only for 5 minutes
res.set('Cache-Control', 'private, max-age=300');

// Must revalidate every time
res.set('Cache-Control', 'no-cache');

// Never cache (sensitive data)
res.set('Cache-Control', 'no-store');

// Cache for 1 day, serve stale for 1 week while revalidating
res.set('Cache-Control', 'public, max-age=86400, stale-while-revalidate=604800');
```

## ETags: Conditional Requests

ETags (Entity Tags) enable efficient revalidation:

**How It Works**:
```
1. Server generates hash of response content
2. Server sends ETag header with response
3. Client caches response and ETag
4. Client sends If-None-Match header with ETag on next request
5. Server checks if content changed
   - If unchanged: Return 304 Not Modified (no body)
   - If changed: Return 200 OK with new content and ETag
```

**Implementation**:
```javascript
const crypto = require('crypto');

function generateETag(data) {
  return crypto
    .createHash('md5')
    .update(JSON.stringify(data))
    .digest('hex');
}

app.get('/api/users/:id', async (req, res) => {
  const user = await db.users.findById(req.params.id);
  
  if (!user) {
    return res.status(404).json({ error: 'Not found' });
  }
  
  const etag = generateETag(user);
  const clientETag = req.headers['if-none-match'];
  
  // Check if client has current version
  if (clientETag === etag) {
    return res.status(304).end();  // Not Modified
  }
  
  res.set({
    'ETag': etag,
    'Cache-Control': 'no-cache'  // Always revalidate
  });
  
  res.json({ user });
});
```

**Benefits**:
- Saves bandwidth (no body in 304 responses)
- Faster than full response
- Guarantees freshness

## Last-Modified: Time-Based Validation

Alternative to ETags using timestamps:

```javascript
app.get('/api/posts/:id', async (req, res) => {
  const post = await db.posts.findById(req.params.id);
  
  const lastModified = new Date(post.updated_at);
  const ifModifiedSince = req.headers['if-modified-since'];
  
  if (ifModifiedSince && new Date(ifModifiedSince) >= lastModified) {
    return res.status(304).end();
  }
  
  res.set({
    'Last-Modified': lastModified.toUTCString(),
    'Cache-Control': 'max-age=300'  // Cache for 5 minutes
  });
  
  res.json({ post });
});
```

**When to Use**:
- Data has clear modification timestamps
- Simpler than ETags
- Less precise (second-level granularity)

## Caching Strategies by Content Type

### Static Content (Images, CSS, JS)

```javascript
// Versioned URLs (best for static assets)
app.get('/assets/:hash/:filename', (req, res) => {
  res.set({
    'Cache-Control': 'public, max-age=31536000, immutable'  // 1 year
  });
  
  res.sendFile(path.join(__dirname, 'assets', req.params.filename));
});

// URL: /assets/abc123/app.js
```

**Why it works**: Changing file changes URL, so cache never stale.

### User-Specific Data

```javascript
// Profile data
app.get('/api/users/me', authenticateToken, async (req, res) => {
  const user = await db.users.findById(req.user.id);
  
  res.set({
    'Cache-Control': 'private, max-age=60',  // Cache in browser for 1 minute
    'Vary': 'Authorization'  // Cache varies by auth token
  });
  
  res.json({ user });
});
```

### Public Data (Updates Infrequently)

```javascript
// Blog posts list
app.get('/api/posts', async (req, res) => {
  const posts = await db.posts.find({ published: true });
  
  res.set({
    'Cache-Control': 'public, max-age=300',  // 5 minutes in any cache
    'Vary': 'Accept-Encoding'
  });
  
  res.json({ posts });
});
```

### Real-Time Data

```javascript
// Stock prices
app.get('/api/stocks/:symbol', async (req, res) => {
  const price = await getStockPrice(req.params.symbol);
  
  res.set({
    'Cache-Control': 'public, max-age=1, stale-while-revalidate=60'
  });
  
  res.json({ price });
});
```

### Sensitive Data

```javascript
// Financial records
app.get('/api/transactions', authenticateToken, async (req, res) => {
  const transactions = await getTransactions(req.user.id);
  
  res.set({
    'Cache-Control': 'private, no-store'  // Never cache
  });
  
  res.json({ transactions });
});
```

## Vary Header

Indicates which request headers affect caching:

```javascript
// Response varies by Accept header
res.set('Vary', 'Accept');

// Multiple headers
res.set('Vary', 'Accept, Accept-Encoding, Authorization');

// Example: JSON vs XML responses should be cached separately
app.get('/api/users', (req, res) => {
  const format = req.accepts(['json', 'xml']);
  
  res.set({
    'Cache-Control': 'public, max-age=300',
    'Vary': 'Accept'  // Cache JSON and XML responses separately
  });
  
  if (format === 'json') {
    res.json({ users });
  } else {
    res.type('xml').send(toXML(users));
  }
});
```

## CDN Caching

Configure caching for CDNs:

```javascript
app.get('/api/products', async (req, res) => {
  const products = await db.products.find();
  
  res.set({
    'Cache-Control': 'public, max-age=60, s-maxage=300',  // CDN caches 5 min, browser 1 min
    'Surrogate-Control': 'max-age=300',  // CDN-specific (takes precedence)
    'CDN-Cache-Control': 'max-age=300'   // Cloudflare, Fastly
  });
  
  res.json({ products });
});
```

## Cache Invalidation

Purge stale cache entries:

```javascript
// When data changes, update cache tags
app.put('/api/posts/:id', async (req, res) => {
  const post = await db.posts.update(req.params.id, req.body);
  
  // Invalidate CDN cache for this post
  await purgeCDNCache(`/api/posts/${req.params.id}`);
  
  // Also invalidate list caches
  await purgeCDNCache('/api/posts');
  
  res.json({ post });
});

// Surrogate-Key for grouped invalidation
app.get('/api/posts/:id', async (req, res) => {
  const post = await db.posts.findById(req.params.id);
  
  res.set({
    'Cache-Control': 'public, max-age=3600',
    'Surrogate-Key': `post-${post.id} posts-list author-${post.author_id}`
  });
  
  res.json({ post });
});

// Purge all posts by author
await purgeCDNByKey(`author-${authorId}`);
```

## Application-Level Caching

Beyond HTTP caching, cache at application level:

```javascript
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 300 });  // 5 minute default

app.get('/api/expensive-query', async (req, res) => {
  const cacheKey = 'expensive-query-result';
  
  // Check cache
  const cached = cache.get(cacheKey);
  if (cached) {
    return res.json({ data: cached, cached: true });
  }
  
  // Expensive operation
  const result = await performExpensiveQuery();
  
  // Store in cache
  cache.set(cacheKey, result, 600);  // Cache for 10 minutes
  
  res.json({ data: result, cached: false });
});
```

## Redis Caching

For distributed systems:

```javascript
const Redis = require('ioredis');
const redis = new Redis();

async function getCachedOrFetch(key, ttl, fetchFn) {
  // Check cache
  const cached = await redis.get(key);
  if (cached) {
    return JSON.parse(cached);
  }
  
  // Fetch fresh data
  const data = await fetchFn();
  
  // Store in cache
  await redis.setex(key, ttl, JSON.stringify(data));
  
  return data;
}

app.get('/api/dashboard/stats', async (req, res) => {
  const stats = await getCachedOrFetch(
    `dashboard:stats:${req.user.id}`,
    300,  // 5 minutes
    () => calculateDashboardStats(req.user.id)
  );
  
  res.json({ stats });
});
```

## Cache Stampede Prevention

Prevent multiple requests from generating same cached content simultaneously:

```javascript
const locks = new Map();

async function cacheWithLock(key, ttl, fetchFn) {
  // Check cache first
  const cached = await redis.get(key);
  if (cached) {
    return JSON.parse(cached);
  }
  
  // Acquire lock
  const lockKey = `lock:${key}`;
  
  // Try to get lock
  const acquired = await redis.set(lockKey, '1', 'EX', 10, 'NX');
  
  if (!acquired) {
    // Another request is fetching, wait and try cache again
    await new Promise(resolve => setTimeout(resolve, 100));
    return cacheWithLock(key, ttl, fetchFn);
  }
  
  try {
    // Fetch data
    const data = await fetchFn();
    
    // Store in cache
    await redis.setex(key, ttl, JSON.stringify(data));
    
    return data;
  } finally {
    // Release lock
    await redis.del(lockKey);
  }
}
```

## Testing Caching

```javascript
describe('Caching', () => {
  it('returns correct Cache-Control headers', async () => {
    const response = await request(app).get('/api/posts');
    
    expect(response.headers['cache-control']).toBe('public, max-age=300');
  });
  
  it('returns 304 for unmodified resources', async () => {
    // First request
    const response1 = await request(app).get('/api/posts/1');
    const etag = response1.headers.etag;
    
    // Second request with ETag
    const response2 = await request(app)
      .get('/api/posts/1')
      .set('If-None-Match', etag);
    
    expect(response2.status).toBe(304);
    expect(response2.body).toEqual({});  // No body
  });
  
  it('returns fresh content after modification', async () => {
    const response1 = await request(app).get('/api/posts/1');
    const etag1 = response1.headers.etag;
    
    // Update post
    await request(app).put('/api/posts/1').send({ title: 'Updated' });
    
    // Request again
    const response2 = await request(app)
      .get('/api/posts/1')
      .set('If-None-Match', etag1);
    
    expect(response2.status).toBe(200);  // New content
    expect(response2.headers.etag).not.toBe(etag1);
  });
});
```

## Best Practices

- [ ] Set Cache-Control on all responses
- [ ] Use `public` for publicly accessible content
- [ ] Use `private` for user-specific data
- [ ] Use `no-store` for sensitive data
- [ ] Implement ETags or Last-Modified for revalidation
- [ ] Set appropriate max-age based on data volatility
- [ ] Use Vary header for content negotiation
- [ ] Enable compression (gzip, brotli)
- [ ] Version static assets (cache forever)
- [ ] Use stale-while-revalidate for better UX
- [ ] Monitor cache hit rates
- [ ] Test caching behavior

## Summary

HTTP caching dramatically improves API performance:

- Use Cache-Control to control caching behavior
- Implement ETags or Last-Modified for validation
- Set appropriate cache durations
- Use CDNs for global distribution
- Add application-level caching for expensive operations
- Prevent cache stampedes with locks
- Monitor and optimize cache hit rates

Caching is the easiest way to make your API 10x faster.

## Further Reading

- **RFC 7234**: HTTP Caching
- **RFC 7232**: Conditional Requests (ETags)
- **MDN Caching Guide**: Browser caching details
- **CDN Documentation**: Cloudflare, Fastly caching

---

**Next**: Monitor your API with [Observability](./observability.md).

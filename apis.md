# REST API Best Practices

A comprehensive guide to designing and implementing robust, scalable REST APIs based on web standards and industry best practices.

## Table of Contents

1. [Versioning](#versioning)
2. [Error Handling](#error-handling)
3. [Authentication](#authentication)
4. [Filtering](#filtering)
5. [Pagination](#pagination)
6. [Rate Limits](#rate-limits)
7. [Content Formats](#content-formats)
8. [Idempotency](#idempotency)
9. [Caching](#caching)
10. [Logging & Health Checks](#logging--health-checks)

---

## Versioning

API versioning ensures backward compatibility while allowing for evolution of your API.

### Strategies

**1. URL Versioning** (Recommended)
```
GET /api/v1/users
GET /api/v2/users
```
- **Pros**: Clear, explicit, easy to route
- **Cons**: URL changes with versions

**2. Header Versioning**
```
GET /api/users
Accept: application/vnd.myapi.v1+json
```
- **Pros**: Clean URLs, follows content negotiation
- **Cons**: Less visible, harder to test

**3. Query Parameter Versioning**
```
GET /api/users?version=1
```
- **Pros**: Easy to implement
- **Cons**: Can be overlooked, mixed with other query params

### Best Practices

- Use semantic versioning (v1, v2, v3)
- Only increment major version for breaking changes
- Maintain at least one previous version
- Provide clear migration guides
- Set deprecation timelines (e.g., 12 months notice)
- Document version differences

### Example Response with Version Info

```json
{
  "api_version": "2.0",
  "data": {
    "users": [...]
  }
}
```

---

## Error Handling

Consistent error handling improves developer experience and debugging.

### HTTP Status Codes

Use appropriate status codes:

**2xx Success**
- `200 OK` - Successful GET, PUT, PATCH, DELETE
- `201 Created` - Successful POST that creates a resource
- `204 No Content` - Successful request with no response body

**4xx Client Errors**
- `400 Bad Request` - Invalid request syntax or validation error
- `401 Unauthorized` - Authentication required or failed
- `403 Forbidden` - Authenticated but not authorized
- `404 Not Found` - Resource doesn't exist
- `409 Conflict` - Request conflicts with current state
- `422 Unprocessable Entity` - Validation errors
- `429 Too Many Requests` - Rate limit exceeded

**5xx Server Errors**
- `500 Internal Server Error` - Generic server error
- `502 Bad Gateway` - Invalid upstream response
- `503 Service Unavailable` - Service temporarily unavailable
- `504 Gateway Timeout` - Upstream timeout

### Error Response Format

Standardize error responses:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request parameters",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format",
        "code": "INVALID_EMAIL"
      },
      {
        "field": "age",
        "message": "Must be at least 18",
        "code": "MIN_VALUE"
      }
    ],
    "request_id": "req_abc123",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

### Error Code Conventions

Use consistent error codes:

```
RESOURCE_NOT_FOUND
AUTHENTICATION_REQUIRED
INSUFFICIENT_PERMISSIONS
VALIDATION_ERROR
RATE_LIMIT_EXCEEDED
INTERNAL_ERROR
SERVICE_UNAVAILABLE
```

### Best Practices

- Always include a human-readable message
- Provide actionable error details
- Include request ID for tracking
- Don't expose sensitive information or stack traces
- Use consistent error structure across all endpoints
- Provide error code documentation

---

## Authentication

Secure your API with proper authentication mechanisms.

### Methods

**1. API Keys** (Simple, for server-to-server)
```
Authorization: ApiKey YOUR_API_KEY
X-API-Key: YOUR_API_KEY
```

**2. Bearer Tokens / JWT** (Recommended)
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**3. OAuth 2.0** (For delegated access)
```
Authorization: Bearer ACCESS_TOKEN
```

**4. Basic Auth** (Legacy, use with HTTPS only)
```
Authorization: Basic base64(username:password)
```

### Best Practices

- Always use HTTPS/TLS in production
- Implement token expiration and refresh mechanisms
- Use short-lived access tokens (15-60 minutes)
- Store tokens securely (never in URLs or logs)
- Implement token revocation
- Use scopes/permissions for fine-grained access control

### Example JWT Payload

```json
{
  "sub": "user_123",
  "iss": "https://api.example.com",
  "aud": "https://api.example.com",
  "exp": 1705324800,
  "iat": 1705321200,
  "scopes": ["read:users", "write:users"]
}
```

### Authentication Errors

```json
{
  "error": {
    "code": "AUTHENTICATION_REQUIRED",
    "message": "Valid authentication credentials required",
    "details": "Missing or invalid Authorization header"
  }
}
```

---

## Filtering

Enable clients to query specific subsets of data.

### Query Parameters

**Basic Filtering**
```
GET /api/users?status=active
GET /api/users?role=admin&status=active
```

**Comparison Operators**
```
GET /api/products?price[gte]=100&price[lte]=500
GET /api/users?created_at[gt]=2024-01-01
```

**Multiple Values (OR logic)**
```
GET /api/users?status=active,pending
GET /api/users?id=1,2,3
```

**Pattern Matching**
```
GET /api/users?name[like]=John*
GET /api/users?email[contains]=@example.com
```

### Advanced Filtering

**Field Selection (Sparse Fieldsets)**
```
GET /api/users?fields=id,name,email
```

**Nested Filtering**
```
GET /api/orders?customer.country=US
GET /api/posts?author.verified=true
```

### Filter Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `eq` | Equal | `status[eq]=active` |
| `ne` | Not equal | `status[ne]=deleted` |
| `gt` | Greater than | `age[gt]=18` |
| `gte` | Greater than or equal | `price[gte]=100` |
| `lt` | Less than | `age[lt]=65` |
| `lte` | Less than or equal | `price[lte]=1000` |
| `in` | In array | `status[in]=active,pending` |
| `nin` | Not in array | `role[nin]=guest,banned` |
| `like` | Pattern match | `name[like]=John*` |
| `contains` | Contains substring | `email[contains]=@gmail` |

### Best Practices

- Document all filterable fields
- Validate filter parameters
- Set reasonable defaults
- Limit complexity to prevent abuse
- Support case-insensitive filtering where appropriate
- Return empty results (not errors) for valid filters with no matches

---

## Pagination

Handle large datasets efficiently with pagination.

### Offset-Based Pagination

**Request**
```
GET /api/users?limit=20&offset=40
```

**Response**
```json
{
  "data": [...],
  "pagination": {
    "limit": 20,
    "offset": 40,
    "total": 150,
    "total_pages": 8,
    "current_page": 3
  },
  "links": {
    "first": "/api/users?limit=20&offset=0",
    "prev": "/api/users?limit=20&offset=20",
    "next": "/api/users?limit=20&offset=60",
    "last": "/api/users?limit=20&offset=140"
  }
}
```

**Pros**: Simple, can jump to any page
**Cons**: Performance degrades with high offsets, inconsistent results if data changes

### Cursor-Based Pagination (Recommended for large datasets)

**Request**
```
GET /api/users?limit=20&cursor=eyJpZCI6MTAwfQ
```

**Response**
```json
{
  "data": [...],
  "pagination": {
    "limit": 20,
    "next_cursor": "eyJpZCI6MTIwfQ",
    "prev_cursor": "eyJpZCI6ODAfQ",
    "has_more": true
  },
  "links": {
    "next": "/api/users?limit=20&cursor=eyJpZCI6MTIwfQ",
    "prev": "/api/users?limit=20&cursor=eyJpZCI6ODAfQ"
  }
}
```

**Pros**: Consistent results, efficient for large datasets
**Cons**: Can't jump to arbitrary pages

### Page-Based Pagination

**Request**
```
GET /api/users?page=3&per_page=20
```

**Response**
```json
{
  "data": [...],
  "pagination": {
    "page": 3,
    "per_page": 20,
    "total": 150,
    "total_pages": 8
  }
}
```

### HTTP Headers for Pagination

```
Link: <https://api.example.com/users?page=3>; rel="next",
      <https://api.example.com/users?page=1>; rel="prev",
      <https://api.example.com/users?page=1>; rel="first",
      <https://api.example.com/users?page=8>; rel="last"
X-Total-Count: 150
X-Page: 2
X-Per-Page: 20
```

### Best Practices

- Set a default page size (e.g., 20-50 items)
- Set a maximum page size (e.g., 100 items)
- Always include pagination metadata
- Provide navigation links
- Use cursor-based pagination for real-time data
- Document pagination behavior
- Consider performance implications of total counts

---

## Rate Limits

Protect your API from abuse and ensure fair usage.

### Implementation

**Fixed Window**
- 1000 requests per hour per user
- Simple but can allow bursts at window boundaries

**Sliding Window**
- 1000 requests per rolling 60-minute window
- More accurate, prevents boundary abuse

**Token Bucket**
- Allows controlled bursts
- Refills at a steady rate

### HTTP Headers

Return rate limit information in response headers:

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 237
X-RateLimit-Reset: 1705324800
X-RateLimit-Reset-After: 3600
Retry-After: 3600
```

### Rate Limit Exceeded Response

**Status**: `429 Too Many Requests`

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "API rate limit exceeded",
    "details": {
      "limit": 1000,
      "window": "1 hour",
      "reset_at": "2024-01-15T11:00:00Z",
      "retry_after": 3600
    }
  }
}
```

### Rate Limit Tiers

Different limits for different user types:

| Tier | Requests/Hour | Burst |
|------|---------------|-------|
| Anonymous | 100 | 10 |
| Authenticated | 1,000 | 50 |
| Premium | 10,000 | 200 |
| Enterprise | Custom | Custom |

### Best Practices

- Document rate limits clearly
- Return limit info in all responses
- Use `429` status code when exceeded
- Provide `Retry-After` header
- Consider different limits for different endpoints
- Allow rate limit increases for verified users
- Implement gradual backoff for repeated violations
- Monitor and alert on rate limit abuse patterns

---

## Content Formats

Support multiple content formats based on client needs.

### Content Negotiation

**Request**
```
GET /api/users
Accept: application/json
```

**Response**
```
Content-Type: application/json; charset=utf-8

{
  "users": [...]
}
```

### Supported Formats

**JSON** (Default, Recommended)
```
Accept: application/json
Content-Type: application/json
```

**XML**
```
Accept: application/xml
Content-Type: application/xml
```

**CSV** (For exports)
```
Accept: text/csv
Content-Type: text/csv
```

**Protocol Buffers** (For high-performance scenarios)
```
Accept: application/x-protobuf
Content-Type: application/x-protobuf
```

### JSON Best Practices

**Use camelCase or snake_case consistently**
```json
{
  "userId": 123,
  "firstName": "John"
}
```

or

```json
{
  "user_id": 123,
  "first_name": "John"
}
```

**Use ISO 8601 for dates**
```json
{
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T14:45:00+00:00"
}
```

**Use null for missing values**
```json
{
  "name": "John",
  "middle_name": null,
  "age": 30
}
```

**Envelope responses consistently**
```json
{
  "data": {...},
  "meta": {
    "timestamp": "2024-01-15T10:30:00Z",
    "version": "2.0"
  }
}
```

### Content Compression

Support compression for large responses:

```
Accept-Encoding: gzip, deflate, br
Content-Encoding: gzip
```

### Unsupported Format Response

**Status**: `406 Not Acceptable`

```json
{
  "error": {
    "code": "UNSUPPORTED_MEDIA_TYPE",
    "message": "Requested format not supported",
    "supported_formats": ["application/json", "application/xml"]
  }
}
```

### Best Practices

- Default to JSON for modern APIs
- Support content negotiation via Accept header
- Use UTF-8 encoding
- Enable compression for responses > 1KB
- Version your content types if format changes
- Document supported formats clearly
- Validate Content-Type on requests

---

## Idempotency

Ensure safe request retries and prevent duplicate operations.

### Idempotent Methods

**Naturally Idempotent**
- `GET` - Safe to retry, no side effects
- `PUT` - Replaces resource, same result
- `DELETE` - Deletes resource, same result
- `HEAD` - Safe to retry, no side effects
- `OPTIONS` - Safe to retry, no side effects

**Not Idempotent by Default**
- `POST` - Creates new resource each time
- `PATCH` - May produce different results

### Idempotency Keys

For non-idempotent operations (especially POST), use idempotency keys:

**Request**
```
POST /api/payments
Idempotency-Key: key_abc123xyz789
Content-Type: application/json

{
  "amount": 1000,
  "currency": "USD",
  "customer_id": "cust_123"
}
```

**First Response** (Creates resource)
```
HTTP/1.1 201 Created
Content-Type: application/json

{
  "id": "pay_456",
  "status": "succeeded",
  "amount": 1000
}
```

**Retry with Same Key** (Returns same result)
```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "pay_456",
  "status": "succeeded",
  "amount": 1000
}
```

### Implementation Guidelines

**Key Requirements**
- Client-generated unique identifier (UUID recommended)
- Store key-response mapping for 24 hours minimum
- Return 409 Conflict if same key used with different payload
- Clean up old keys periodically

**Validation**
```json
{
  "error": {
    "code": "IDEMPOTENCY_KEY_MISMATCH",
    "message": "Request parameters differ from original request with this idempotency key",
    "original_request_id": "req_789"
  }
}
```

### Best Practices

- Use UUIDs or cryptographically random strings
- Set appropriate expiration (24-72 hours)
- Store minimal data (request hash + response)
- Return original response status code
- Document idempotency behavior
- Handle concurrent requests with same key gracefully
- Use for financial transactions, order creation, etc.

### Example Idempotent POST

```javascript
// Client implementation
async function createPayment(paymentData) {
  const idempotencyKey = generateUUID();
  
  return fetch('/api/payments', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Idempotency-Key': idempotencyKey
    },
    body: JSON.stringify(paymentData)
  });
}
```

---

## Caching

Optimize performance and reduce server load with effective caching strategies.

### HTTP Cache Headers

**Cache-Control**
```
Cache-Control: public, max-age=3600
Cache-Control: private, max-age=300
Cache-Control: no-cache
Cache-Control: no-store
```

**Directives**
- `public` - Cacheable by any cache
- `private` - Cacheable by client only
- `no-cache` - Revalidate before use
- `no-store` - Don't cache at all
- `max-age=N` - Cache for N seconds
- `s-maxage=N` - CDN/shared cache override
- `must-revalidate` - Strict revalidation when stale

### ETags (Entity Tags)

**First Request**
```
GET /api/users/123
```

**Response**
```
HTTP/1.1 200 OK
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
Cache-Control: max-age=0, must-revalidate

{
  "id": 123,
  "name": "John Doe"
}
```

**Conditional Request**
```
GET /api/users/123
If-None-Match: "33a64df551425fcc55e4d42a148795d9f25f89d4"
```

**Not Modified Response**
```
HTTP/1.1 304 Not Modified
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
```

### Last-Modified

**Response**
```
HTTP/1.1 200 OK
Last-Modified: Mon, 15 Jan 2024 10:30:00 GMT
Cache-Control: max-age=3600
```

**Conditional Request**
```
GET /api/users/123
If-Modified-Since: Mon, 15 Jan 2024 10:30:00 GMT
```

### Caching Strategies by Endpoint Type

**Static/Rarely Changed Data**
```
Cache-Control: public, max-age=86400, immutable
```

**User-Specific Data**
```
Cache-Control: private, max-age=300
```

**Real-Time Data**
```
Cache-Control: no-cache, must-revalidate
```

**Sensitive Data**
```
Cache-Control: no-store, private
```

**API Responses with Frequent Updates**
```
Cache-Control: public, max-age=60, stale-while-revalidate=300
```

### Vary Header

Indicate which request headers affect caching:

```
Vary: Accept, Accept-Encoding, Authorization
```

### Cache Invalidation

**Time-Based** (TTL)
- Simplest approach
- Use `max-age` directive

**Event-Based**
- Invalidate on resource changes
- Use cache keys with version/timestamp

**Purge API**
- Explicit cache clearing
- For critical updates

### Best Practices

- Use ETags for frequently accessed resources
- Set appropriate `max-age` based on data volatility
- Use `private` for personalized content
- Use `public` for shared resources
- Combine with CDN for global caching
- Document caching behavior
- Monitor cache hit rates
- Use `Vary` header appropriately
- Consider stale-while-revalidate for better UX
- Never cache sensitive data (passwords, tokens)

### Example Caching Strategy

```javascript
// Server-side caching logic
app.get('/api/products/:id', (req, res) => {
  const product = getProduct(req.params.id);
  const etag = generateETag(product);
  
  // Check if client has current version
  if (req.headers['if-none-match'] === etag) {
    return res.status(304).end();
  }
  
  res.set({
    'Cache-Control': 'public, max-age=3600',
    'ETag': etag,
    'Vary': 'Accept-Encoding'
  });
  
  res.json(product);
});
```

---

## Logging & Health Checks

Ensure observability, monitoring, and reliability of your API.

### Logging

**What to Log**

**Request Logs**
```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "request_id": "req_abc123",
  "method": "POST",
  "path": "/api/users",
  "status": 201,
  "duration_ms": 45,
  "ip": "192.168.1.100",
  "user_agent": "Mozilla/5.0...",
  "user_id": "user_456"
}
```

**Error Logs**
```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "error",
  "request_id": "req_abc123",
  "error": {
    "type": "DatabaseConnectionError",
    "message": "Connection timeout",
    "stack": "Error: Connection timeout\n  at ...",
    "code": "ECONNREFUSED"
  },
  "context": {
    "user_id": "user_456",
    "endpoint": "/api/users"
  }
}
```

**Security Logs**
```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "event": "authentication_failed",
  "ip": "192.168.1.100",
  "user_id": null,
  "reason": "invalid_credentials",
  "attempt_count": 3
}
```

### Log Levels

- `DEBUG` - Detailed diagnostic information
- `INFO` - General informational messages
- `WARN` - Warning messages, potential issues
- `ERROR` - Error events, still functioning
- `FATAL` - Severe errors, application crash

### Structured Logging

Use JSON format for machine-readable logs:

```json
{
  "timestamp": "2024-01-15T10:30:00.123Z",
  "level": "info",
  "service": "api-gateway",
  "environment": "production",
  "request_id": "req_abc123",
  "message": "User created successfully",
  "data": {
    "user_id": "user_789",
    "email": "user@example.com"
  }
}
```

### What NOT to Log

- Passwords or authentication credentials
- Credit card numbers or PII
- API keys or secrets
- Full request/response bodies with sensitive data

### Request IDs

Generate unique request IDs for tracing:

```
X-Request-ID: req_abc123xyz789
```

Include in all log entries and error responses.

### Health Checks

**Basic Health Check**

```
GET /health
```

**Response**
```json
{
  "status": "healthy",
  "timestamp": "2024-01-15T10:30:00Z",
  "version": "2.0.0",
  "uptime": 86400
}
```

**Detailed Health Check**

```
GET /health/detailed
```

**Response**
```json
{
  "status": "healthy",
  "timestamp": "2024-01-15T10:30:00Z",
  "version": "2.0.0",
  "uptime": 86400,
  "checks": {
    "database": {
      "status": "healthy",
      "response_time_ms": 12,
      "details": "PostgreSQL 14.0"
    },
    "cache": {
      "status": "healthy",
      "response_time_ms": 3,
      "details": "Redis 7.0"
    },
    "external_api": {
      "status": "degraded",
      "response_time_ms": 850,
      "details": "High latency"
    }
  },
  "metrics": {
    "requests_per_second": 145,
    "error_rate": 0.02,
    "avg_response_time_ms": 78
  }
}
```

### Health Check Status Codes

- `200 OK` - Healthy
- `503 Service Unavailable` - Unhealthy
- `429 Too Many Requests` - Health check rate limited

### Readiness vs Liveness

**Liveness Probe** (`/health/live`)
- Is the application running?
- Used to restart crashed containers

**Readiness Probe** (`/health/ready`)
- Is the application ready to serve traffic?
- Used to remove from load balancer during startup/shutdown

```
GET /health/ready
```

**Response (Not Ready)**
```json
{
  "status": "not_ready",
  "reason": "database_connection_pending",
  "checks": {
    "database": {
      "status": "initializing"
    }
  }
}
```

### Monitoring Metrics

**Key Metrics to Track**

- Request rate (requests/second)
- Error rate (4xx, 5xx)
- Response time (p50, p95, p99)
- Availability (uptime %)
- Throughput
- Active connections
- Queue depth

**Metrics Endpoint**

```
GET /metrics
```

**Response (Prometheus format)**
```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 1234567

# HELP http_request_duration_seconds HTTP request latency
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.1"} 10000
http_request_duration_seconds_bucket{le="0.5"} 25000
```

### Best Practices

**Logging**
- Use structured logging (JSON)
- Include request IDs in all logs
- Set appropriate log levels
- Rotate logs to prevent disk space issues
- Centralize logs for distributed systems
- Sanitize sensitive data before logging
- Log at application boundaries (requests, database calls, external APIs)

**Health Checks**
- Keep health checks lightweight
- Don't expose sensitive information
- Implement both liveness and readiness probes
- Include dependency health (database, cache, external APIs)
- Set appropriate timeouts
- Cache health check results briefly to prevent overhead
- Monitor health check failures

**Monitoring**
- Set up alerts for key metrics
- Track SLIs (Service Level Indicators)
- Define SLOs (Service Level Objectives)
- Use distributed tracing for complex systems
- Monitor business metrics alongside technical metrics
- Create dashboards for real-time visibility

### Example Implementation

```javascript
// Express.js health check middleware
app.get('/health', async (req, res) => {
  const health = {
    status: 'healthy',
    timestamp: new Date().toISOString(),
    version: process.env.APP_VERSION,
    uptime: process.uptime()
  };
  
  try {
    // Quick dependency checks
    await db.ping();
    await cache.ping();
    
    res.status(200).json(health);
  } catch (error) {
    health.status = 'unhealthy';
    health.error = error.message;
    res.status(503).json(health);
  }
});

// Request logging middleware
app.use((req, res, next) => {
  const requestId = req.headers['x-request-id'] || generateUUID();
  req.requestId = requestId;
  res.setHeader('X-Request-ID', requestId);
  
  const startTime = Date.now();
  
  res.on('finish', () => {
    logger.info({
      timestamp: new Date().toISOString(),
      request_id: requestId,
      method: req.method,
      path: req.path,
      status: res.statusCode,
      duration_ms: Date.now() - startTime,
      ip: req.ip,
      user_agent: req.headers['user-agent']
    });
  });
  
  next();
});
```

---

## Summary

Following these REST API best practices will help you build robust, scalable, and developer-friendly APIs:

1. **Version your API** to allow evolution while maintaining backward compatibility
2. **Handle errors consistently** with proper status codes and structured responses
3. **Secure with authentication** using modern standards like JWT and OAuth 2.0
4. **Enable filtering** to let clients query exactly what they need
5. **Implement pagination** for efficient handling of large datasets
6. **Apply rate limits** to protect your API from abuse
7. **Support multiple content formats** with proper content negotiation
8. **Use idempotency keys** for safe retries of critical operations
9. **Implement caching** to improve performance and reduce load
10. **Log comprehensively and provide health checks** for observability and reliability

These standards create a foundation for APIs that are reliable, performant, and pleasant to use.

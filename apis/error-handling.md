# Error Handling: When Things Go Wrong

> **Errors are not failures of your API—they're conversations with your users about what went wrong and how to fix it.**

It's 2 AM. A payment just failed. Your user's card was declined, but your API returned a generic "500 Internal Server Error." The mobile app shows "Something went wrong. Try again later." The user abandons their cart. Your company loses the sale. The developer who integrated your API gets blamed.

This happens thousands of times a day across the web, and it's almost always preventable. The difference between a good API and a great one often comes down to how it handles errors.

## The Real Cost of Bad Error Handling

Let's be honest about what's at stake:

**For Your Users**:
- Frustration when they don't know what went wrong
- Lost time trying the same failing action repeatedly
- Abandoned tasks because the error message said "contact support"

**For Developers Using Your API**:
- Hours debugging cryptic error messages
- Support tickets asking "what does error code 42 mean?"
- Loss of trust in your API's reliability

**For Your Business**:
- Increased support costs
- Reduced API adoption
- Bad reputation in developer communities

Good error handling isn't just nice to have—it's essential infrastructure.

## The Foundation: HTTP Status Codes

Before we dive into error response formats, we need to understand the HTTP status code system. It's defined in RFC 7231 and it's more sophisticated than most people realize.

### The Five Categories

HTTP status codes use a simple pattern: the first digit indicates the category.

**1xx: Informational** (100-199)
- The request was received and is being processed
- Rarely used in REST APIs
- Example: `100 Continue`

**2xx: Success** (200-299)
- The request succeeded
- Different codes indicate different types of success
- Examples: `200 OK`, `201 Created`, `204 No Content`

**3xx: Redirection** (300-399)
- Further action needed to complete the request
- Usually handled automatically by HTTP clients
- Examples: `301 Moved Permanently`, `304 Not Modified`

**4xx: Client Errors** (400-499)
- The request contains an error (bad syntax, invalid data, etc.)
- **The client should not retry without changes**
- Examples: `400 Bad Request`, `404 Not Found`, `429 Too Many Requests`

**5xx: Server Errors** (500-599)
- The server failed to fulfill a valid request
- **The client might retry and succeed**
- Examples: `500 Internal Server Error`, `503 Service Unavailable`

This distinction between 4xx and 5xx is critical: it tells the client whether retrying makes sense.

### Choosing the Right Status Code

Here's how to choose status codes for common scenarios:

#### Success Scenarios

**`200 OK`** - The workhorse of HTTP
```javascript
// Successful GET, PUT, PATCH, or DELETE
GET /api/users/123

HTTP/1.1 200 OK
{
  "id": "123",
  "name": "Alice"
}
```
Use when: The request succeeded and you're returning data.

**`201 Created`** - Resource creation confirmed
```javascript
// Successful POST that creates a new resource
POST /api/users

HTTP/1.1 201 Created
Location: /api/users/124
{
  "id": "124",
  "name": "Bob"
}
```
Use when: You created a new resource. Always include a `Location` header.

**`204 No Content`** - Success with nothing to say
```javascript
// Successful DELETE
DELETE /api/users/123

HTTP/1.1 204 No Content
```
Use when: The request succeeded but there's no data to return.

**`202 Accepted`** - Request queued
```javascript
// Async operation started
POST /api/reports/generate

HTTP/1.1 202 Accepted
{
  "job_id": "job_789",
  "status": "pending",
  "status_url": "/api/jobs/job_789"
}
```
Use when: The request was accepted but will be processed later.

#### Client Error Scenarios

**`400 Bad Request`** - Generic client error
```javascript
// Malformed JSON
POST /api/users
{
  "name": "Alice"
  "email": "alice@example.com"  // Missing comma!
}

HTTP/1.1 400 Bad Request
{
  "error": {
    "type": "invalid_request",
    "message": "Request body contains invalid JSON",
    "details": "Expected comma at line 2, column 18"
  }
}
```
Use when: The request is syntactically invalid.

**`401 Unauthorized`** - Not authenticated
```javascript
// Missing or invalid authentication
GET /api/users/me

HTTP/1.1 401 Unauthorized
WWW-Authenticate: ****** realm="API"
{
  "error": {
    "type": "authentication_required",
    "message": "Valid authentication credentials required",
    "details": "Include a valid ******in the Authorization header"
  }
}
```
Use when: The user hasn't provided credentials or they're invalid.

Note the naming confusion: Despite its name, `401` means "not authenticated," not "not authorized." Authentication ("who are you?") comes before authorization ("what can you do?").

**`403 Forbidden`** - Not authorized
```javascript
// Authenticated but lacks permission
DELETE /api/users/123

HTTP/1.1 403 Forbidden
{
  "error": {
    "type": "insufficient_permissions",
    "message": "You don't have permission to delete this user",
    "required_role": "admin",
    "your_role": "user"
  }
}
```
Use when: The user is authenticated but doesn't have permission.

**`404 Not Found`** - Resource doesn't exist
```javascript
GET /api/users/999

HTTP/1.1 404 Not Found
{
  "error": {
    "type": "resource_not_found",
    "message": "User not found",
    "resource_type": "user",
    "resource_id": "999"
  }
}
```
Use when: The requested resource doesn't exist.

**`409 Conflict`** - Request conflicts with current state
```javascript
// Trying to create a user with an existing email
POST /api/users
{
  "email": "alice@example.com",
  "name": "Alice"
}

HTTP/1.1 409 Conflict
{
  "error": {
    "type": "duplicate_resource",
    "message": "A user with this email already exists",
    "conflicting_field": "email",
    "conflicting_value": "alice@example.com"
  }
}
```
Use when: The request is valid but conflicts with the current state.

**`422 Unprocessable Entity`** - Validation failed
```javascript
// Valid JSON, but invalid data
POST /api/users
{
  "email": "not-an-email",
  "age": -5
}

HTTP/1.1 422 Unprocessable Entity
{
  "error": {
    "type": "validation_error",
    "message": "Request validation failed",
    "errors": [
      {
        "field": "email",
        "message": "Must be a valid email address",
        "value": "not-an-email"
      },
      {
        "field": "age",
        "message": "Must be a positive number",
        "value": -5
      }
    ]
  }
}
```
Use when: The request is syntactically valid but semantically incorrect.

**`429 Too Many Requests`** - Rate limit exceeded
```javascript
// Too many requests from this client
GET /api/users

HTTP/1.1 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1704729600

{
  "error": {
    "type": "rate_limit_exceeded",
    "message": "API rate limit exceeded",
    "limit": 1000,
    "window": "1 hour",
    "retry_after": 60
  }
}
```
Use when: The client has exceeded rate limits.

#### Server Error Scenarios

**`500 Internal Server Error`** - Something went wrong
```javascript
// Database connection failed
GET /api/users/123

HTTP/1.1 500 Internal Server Error
{
  "error": {
    "type": "internal_error",
    "message": "An internal error occurred",
    "request_id": "req_abc123",
    "timestamp": "2024-01-08T14:30:00Z"
  }
}
```
Use when: Your server encountered an unexpected condition.

**Critical**: Never expose internal details (stack traces, database errors) in production.

**`502 Bad Gateway`** - Upstream service failed
```javascript
// Payment provider returned an error
POST /api/payments

HTTP/1.1 502 Bad Gateway
{
  "error": {
    "type": "upstream_error",
    "message": "Payment provider is unavailable",
    "request_id": "req_abc123"
  }
}
```
Use when: A service you depend on failed.

**`503 Service Unavailable`** - Temporary outage
```javascript
// Server is overloaded
GET /api/users

HTTP/1.1 503 Service Unavailable
Retry-After: 120
{
  "error": {
    "type": "service_unavailable",
    "message": "Service temporarily unavailable",
    "retry_after": 120
  }
}
```
Use when: The service is temporarily unavailable but will recover.

**`504 Gateway Timeout`** - Upstream didn't respond
```javascript
// External API took too long
GET /api/external-data

HTTP/1.1 504 Gateway Timeout
{
  "error": {
    "type": "timeout",
    "message": "Request to upstream service timed out",
    "timeout": 30
  }
}
```
Use when: A dependency didn't respond in time.

## Error Response Format: RFC 7807

Now that we understand status codes, let's talk about the response body. There's an RFC for this: RFC 7807, "Problem Details for HTTP APIs."

### The Standard Format

RFC 7807 defines a JSON (or XML) format for error responses:

```javascript
{
  "type": "https://api.example.com/errors/insufficient-credit",
  "title": "Insufficient credit",
  "status": 403,
  "detail": "Your account has only $50 but this operation requires $100",
  "instance": "/api/payments/txn_123",
  "balance": 50,
  "required": 100,
  "account_id": "acc_456"
}
```

Let's break down each field:

**`type`** (string, required)
- A URI that identifies the error type
- Should be a permalink to human-readable documentation
- Defaults to "about:blank" if omitted

**`title`** (string, required)
- A short, human-readable summary
- Should be the same for all instances of this error type
- Think of it as the error "name"

**`status`** (number, optional but recommended)
- The HTTP status code
- Duplicates the response status for convenience

**`detail`** (string, optional)
- A human-readable explanation specific to this occurrence
- Can include instance-specific information

**`instance`** (string, optional)
- A URI reference to this specific error occurrence
- Useful for debugging and support

**Additional Fields** (optional)
- You can add custom fields specific to the error
- In the example above: `balance`, `required`, `account_id`

### Practical Implementation

Here's how to implement RFC 7807 in practice:

```javascript
class ApiError extends Error {
  constructor({
    type,
    title,
    status,
    detail,
    instance,
    ...extensions
  }) {
    super(detail || title);
    this.type = type || 'about:blank';
    this.title = title;
    this.status = status;
    this.detail = detail;
    this.instance = instance;
    this.extensions = extensions;
  }

  toJSON() {
    return {
      type: this.type,
      title: this.title,
      status: this.status,
      detail: this.detail,
      instance: this.instance,
      ...this.extensions
    };
  }
}

// Usage
app.post('/api/payments', async (req, res) => {
  try {
    const account = await getAccount(req.user.id);
    const amount = req.body.amount;
    
    if (account.balance < amount) {
      throw new ApiError({
        type: 'https://api.example.com/errors/insufficient-credit',
        title: 'Insufficient credit',
        status: 403,
        detail: `Your account has $${account.balance} but this operation requires $${amount}`,
        instance: `/api/payments/${req.id}`,
        balance: account.balance,
        required: amount,
        account_id: account.id
      });
    }
    
    // Process payment...
  } catch (error) {
    if (error instanceof ApiError) {
      res.status(error.status).json(error.toJSON());
    } else {
      // Handle unexpected errors
      res.status(500).json({
        type: 'about:blank',
        title: 'Internal Server Error',
        status: 500,
        detail: 'An unexpected error occurred',
        instance: `/api/payments/${req.id}`
      });
    }
  }
});
```

## Validation Errors: Going Deeper

Validation errors deserve special attention because they're so common. When multiple fields fail validation, you need a format that communicates all the problems at once.

### The Field-Level Error Pattern

```javascript
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Error",
  "status": 422,
  "detail": "The request contains invalid fields",
  "instance": "/api/users",
  "errors": [
    {
      "field": "email",
      "code": "invalid_format",
      "message": "Must be a valid email address",
      "value": "not-an-email"
    },
    {
      "field": "password",
      "code": "too_short",
      "message": "Must be at least 8 characters",
      "value": "***",
      "min_length": 8,
      "actual_length": 3
    },
    {
      "field": "age",
      "code": "out_of_range",
      "message": "Must be between 18 and 120",
      "value": 15,
      "min": 18,
      "max": 120
    }
  ]
}
```

Each error object includes:
- **field**: Which field failed
- **code**: Machine-readable error code
- **message**: Human-readable message
- **value**: The invalid value (be careful with sensitive data!)
- **Additional context**: Any constraints that were violated

### Nested Field Validation

For complex objects, use dot notation or JSON pointers:

```javascript
{
  "errors": [
    {
      "field": "shipping_address.zip_code",
      "code": "invalid_format",
      "message": "ZIP code must be 5 digits",
      "value": "abcde"
    },
    {
      "field": "items[0].quantity",
      "code": "out_of_range",
      "message": "Quantity must be between 1 and 10",
      "value": 0
    }
  ]
}
```

## Error Codes: The Machine-Readable Layer

HTTP status codes tell you the general category. Error codes tell you specifically what went wrong.

### Designing Error Codes

**Pattern 1: SCREAMING_SNAKE_CASE**
```javascript
{
  "error": {
    "code": "INSUFFICIENT_CREDIT",
    "message": "Your account has insufficient credit"
  }
}
```

Pros: Easy to type, grep-able
Cons: Visually loud

**Pattern 2: kebab-case**
```javascript
{
  "error": {
    "code": "insufficient-credit",
    "message": "Your account has insufficient credit"
  }
}
```

Pros: Clean, URL-friendly
Cons: Harder to distinguish from field names

**Pattern 3: PascalCase (Microsoft style)**
```javascript
{
  "error": {
    "code": "InsufficientCredit",
    "message": "Your account has insufficient credit"
  }
}
```

Pros: Looks like class names, familiar to many developers
Cons: Can be confused with type names

**Recommendation**: Pick one and be consistent. We prefer `SCREAMING_SNAKE_CASE` for error codes because they stand out.

### Organizing Error Codes

Group error codes by domain:

```
AUTH_*
  AUTH_INVALID_CREDENTIALS
  AUTH_TOKEN_EXPIRED
  AUTH_TOKEN_INVALID
  AUTH_INSUFFICIENT_PERMISSIONS

VALIDATION_*
  VALIDATION_REQUIRED_FIELD
  VALIDATION_INVALID_FORMAT
  VALIDATION_OUT_OF_RANGE

RESOURCE_*
  RESOURCE_NOT_FOUND
  RESOURCE_ALREADY_EXISTS
  RESOURCE_CONFLICT

RATE_LIMIT_*
  RATE_LIMIT_EXCEEDED
  RATE_LIMIT_QUOTA_EXHAUSTED

PAYMENT_*
  PAYMENT_CARD_DECLINED
  PAYMENT_INSUFFICIENT_FUNDS
  PAYMENT_PROCESSING_ERROR
```

### Error Code Documentation

For each error code, document:
- What it means
- When it occurs
- How to fix it
- Example request/response

```markdown
## VALIDATION_INVALID_EMAIL

**Status Code**: 422 Unprocessable Entity

**Description**: The provided email address is not valid.

**Common Causes**:
- Missing @ symbol
- No domain extension
- Contains invalid characters

**How to Fix**:
- Validate email format before sending: `/^[^\s@]+@[^\s@]+\.[^\s@]+$/`
- Ensure email is URL-encoded if sent in query params

**Example**:
```javascript
POST /api/users
{
  "email": "invalid.email"
}

HTTP/1.1 422 Unprocessable Entity
{
  "error": {
    "code": "VALIDATION_INVALID_EMAIL",
    "message": "Invalid email format",
    "field": "email"
  }
}
```
```

## The Request ID Pattern

Every error response should include a request ID. This is your debugging lifeline.

```javascript
{
  "error": {
    "type": "https://api.example.com/errors/internal-error",
    "title": "Internal Server Error",
    "status": 500,
    "detail": "An unexpected error occurred",
    "request_id": "req_7x9k2m3n4p",  // Critical for debugging!
    "timestamp": "2024-01-08T14:30:00Z"
  }
}
```

### Why Request IDs Matter

**Scenario**: A user reports an error.

**Without Request ID**:
```
User: "I got an error when trying to update my profile."
Support: "When did this happen?"
User: "Um, maybe 2 PM yesterday?"
Support: *searches through millions of log entries*
```

**With Request ID**:
```
User: "I got an error. The request ID is req_7x9k2m3n4p."
Support: *searches logs for exact request*
Support: "Found it. Your email was already taken."
```

### Implementing Request IDs

```javascript
const { v4: uuidv4 } = require('uuid');

// Middleware to add request ID
app.use((req, res, next) => {
  // Use existing request ID or generate new one
  req.id = req.headers['x-request-id'] || `req_${uuidv4()}`;
  
  // Echo it back in response
  res.setHeader('X-Request-ID', req.id);
  
  next();
});

// Include in all error responses
app.use((err, req, res, next) => {
  const error = {
    type: getErrorType(err),
    title: getErrorTitle(err),
    status: getErrorStatus(err),
    detail: err.message,
    request_id: req.id,  // Always include!
    timestamp: new Date().toISOString()
  };
  
  // Log with request ID
  logger.error({
    request_id: req.id,
    error: err.message,
    stack: err.stack
  });
  
  res.status(error.status).json(error);
});
```

## Security: What Not to Expose

Error messages can leak information to attackers. Here's what to avoid:

### Don't Expose Internal Details

**Bad**:
```javascript
{
  "error": "Uncaught MongoError: connect ECONNREFUSED 127.0.0.1:27017"
}
```

This tells an attacker:
- You're using MongoDB
- The database is on localhost
- The database is down

**Good**:
```javascript
{
  "error": {
    "type": "https://api.example.com/errors/service-unavailable",
    "title": "Service Unavailable",
    "status": 503,
    "detail": "The service is temporarily unavailable",
    "request_id": "req_abc123"
  }
}
```

### Don't Expose Stack Traces in Production

**Bad**:
```javascript
{
  "error": "Error: user not found",
  "stack": "Error: user not found\n    at UserController.getUser (src/controllers/user.js:42:15)\n    at..."
}
```

**Good**:
```javascript
{
  "error": {
    "type": "https://api.example.com/errors/not-found",
    "title": "Not Found",
    "status": 404,
    "detail": "User not found",
    "request_id": "req_abc123"
  }
}
```

Log the stack trace server-side with the request ID. Support can look it up if needed.

### Be Careful with User Enumeration

**Bad**:
```javascript
POST /api/login
{"email": "alice@example.com", "password": "wrong"}

// Different messages for existing vs non-existing users
{
  "error": "Incorrect password"  // Tells attacker the email exists!
}
```

**Good**:
```javascript
POST /api/login
{"email": "alice@example.com", "password": "wrong"}

{
  "error": {
    "type": "https://api.example.com/errors/invalid-credentials",
    "title": "Invalid Credentials",
    "status": 401,
    "detail": "Invalid email or password"  // Doesn't reveal which
  }
}
```

## Error Handling Across the Stack

Errors can occur at multiple layers. Here's how to handle each:

### Application Layer

```javascript
// Domain-specific business logic errors
class InsufficientCreditError extends ApiError {
  constructor(required, available) {
    super({
      type: 'https://api.example.com/errors/insufficient-credit',
      title: 'Insufficient Credit',
      status: 403,
      detail: `Required: $${required}, Available: $${available}`,
      required,
      available
    });
  }
}

// Throw in business logic
if (account.balance < amount) {
  throw new InsufficientCreditError(amount, account.balance);
}
```

### Database Layer

```javascript
try {
  const user = await db.users.findOne({ email });
} catch (err) {
  if (err.code === 'ECONNREFUSED') {
    // Database is down
    throw new ApiError({
      type: 'https://api.example.com/errors/service-unavailable',
      title: 'Service Unavailable',
      status: 503,
      detail: 'Service temporarily unavailable'
    });
  }
  
  if (err.name === 'ValidationError') {
    // Mongoose validation error
    throw new ApiError({
      type: 'https://api.example.com/errors/validation-error',
      title: 'Validation Error',
      status: 422,
      detail: 'Invalid data provided',
      errors: formatMongooseErrors(err)
    });
  }
  
  // Unknown error
  throw err;
}
```

### External API Layer

```javascript
try {
  const response = await fetch('https://payment-provider.com/api/charge', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(chargeData)
  });
  
  if (!response.ok) {
    throw new ApiError({
      type: 'https://api.example.com/errors/payment-failed',
      title: 'Payment Failed',
      status: response.status,
      detail: 'Payment could not be processed',
      provider_error: await response.text(),
      request_id: req.id
    });
  }
} catch (err) {
  if (err.code === 'ETIMEDOUT') {
    throw new ApiError({
      type: 'https://api.example.com/errors/gateway-timeout',
      title: 'Gateway Timeout',
      status: 504,
      detail: 'Payment provider did not respond in time'
    });
  }
  
  throw err;
}
```

## Global Error Handler

Centralize error handling to ensure consistency:

```javascript
// Express global error handler
app.use((err, req, res, next) => {
  // Log all errors
  logger.error({
    request_id: req.id,
    error: err.message,
    stack: err.stack,
    url: req.url,
    method: req.method,
    user: req.user?.id
  });
  
  // Handle known error types
  if (err instanceof ApiError) {
    return res.status(err.status).json(err.toJSON());
  }
  
  // Handle validation errors
  if (err.name === 'ValidationError') {
    return res.status(422).json({
      type: 'https://api.example.com/errors/validation-error',
      title: 'Validation Error',
      status: 422,
      detail: 'Invalid data provided',
      errors: formatValidationErrors(err),
      request_id: req.id
    });
  }
  
  // Handle JWT errors
  if (err.name === 'JsonWebTokenError') {
    return res.status(401).json({
      type: 'https://api.example.com/errors/invalid-token',
      title: 'Invalid Token',
      status: 401,
      detail: 'Authentication token is invalid',
      request_id: req.id
    });
  }
  
  // Unknown error - don't expose details
  res.status(500).json({
    type: 'about:blank',
    title: 'Internal Server Error',
    status: 500,
    detail: 'An unexpected error occurred',
    request_id: req.id,
    timestamp: new Date().toISOString()
  });
});
```

## Testing Error Scenarios

Don't just test the happy path—test your error handling:

```javascript
describe('POST /api/users', () => {
  it('returns 422 for invalid email', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({ email: 'invalid', name: 'Alice' });
    
    expect(response.status).toBe(422);
    expect(response.body).toMatchObject({
      type: expect.stringContaining('validation-error'),
      title: 'Validation Error',
      status: 422,
      errors: expect.arrayContaining([
        expect.objectContaining({
          field: 'email',
          code: 'invalid_format'
        })
      ])
    });
  });
  
  it('returns 409 for duplicate email', async () => {
    // Create user
    await createUser({ email: 'alice@example.com' });
    
    // Try to create again
    const response = await request(app)
      .post('/api/users')
      .send({ email: 'alice@example.com', name: 'Alice' });
    
    expect(response.status).toBe(409);
    expect(response.body.type).toContain('duplicate-resource');
  });
  
  it('returns 503 when database is down', async () => {
    // Mock database failure
    jest.spyOn(db, 'users').mockRejectedValue(
      new Error('ECONNREFUSED')
    );
    
    const response = await request(app)
      .post('/api/users')
      .send({ email: 'alice@example.com', name: 'Alice' });
    
    expect(response.status).toBe(503);
    expect(response.body.request_id).toBeDefined();
  });
});
```

## Client-Side Error Handling

How should clients handle your errors?

### Parse by Status Code Category

```javascript
async function apiRequest(url, options) {
  const response = await fetch(url, options);
  
  if (response.ok) {
    return response.json();
  }
  
  const error = await response.json();
  
  if (response.status >= 400 && response.status < 500) {
    // Client error - don't retry
    if (response.status === 401) {
      // Redirect to login
      redirectToLogin();
    } else if (response.status === 422) {
      // Show validation errors to user
      showValidationErrors(error.errors);
    } else {
      // Show generic error
      showError(error.detail);
    }
    throw error;
  }
  
  if (response.status >= 500) {
    // Server error - might retry
    if (response.status === 503 && error.retry_after) {
      // Wait and retry
      await sleep(error.retry_after * 1000);
      return apiRequest(url, options);
    }
    
    // Log for debugging
    logError({
      request_id: error.request_id,
      url,
      status: response.status
    });
    
    showError('Service temporarily unavailable. Please try again.');
    throw error;
  }
}
```

### Display User-Friendly Messages

```javascript
function formatErrorForUser(error) {
  // Use custom messages for common errors
  const messages = {
    'VALIDATION_INVALID_EMAIL': 'Please enter a valid email address',
    'RATE_LIMIT_EXCEEDED': 'You\'re doing that too fast. Please slow down.',
    'INSUFFICIENT_CREDIT': 'You don\'t have enough credit for this operation'
  };
  
  return messages[error.code] || error.detail || 'Something went wrong';
}
```

## Summary: Error Handling Checklist

When implementing error handling:

- [ ] Use appropriate HTTP status codes (4xx for client errors, 5xx for server errors)
- [ ] Follow RFC 7807 for error response format
- [ ] Include machine-readable error codes
- [ ] Generate and include request IDs
- [ ] Never expose internal details in production
- [ ] Document all error codes and their meanings
- [ ] Handle validation errors with field-level detail
- [ ] Log errors server-side with full context
- [ ] Test error scenarios, not just happy paths
- [ ] Provide helpful error messages that explain how to fix issues
- [ ] Use consistent error format across all endpoints
- [ ] Include `Retry-After` headers for rate limits and 503 errors

Good error handling turns frustrating failures into guided recovery. Your users will thank you.

## Further Reading

- **RFC 7231**: HTTP Semantics (status codes)
- **RFC 7807**: Problem Details for HTTP APIs
- **RFC 8594**: Sunset HTTP Header
- **OWASP API Security**: Error handling best practices
- **JSON Schema**: For validating request bodies

---

**Next**: Learn how to secure your API in [Authentication](./authentication.md).

# API Versioning: Planning for Change

> **Your API will change. The question is: will you break your users when it does?**

Every API starts simple. You launch with a clean design, a handful of endpoints, and everything works beautifully. Then reality hits: you need to add a new field, change a response format, or—worse—fix a fundamental design flaw. You have thousands of apps in production depending on your current API. What do you do?

This is the versioning problem. Solve it well, and your API can evolve gracefully for years. Solve it poorly, and you'll either stagnate your API or break your users' applications. There's no middle ground.

## The Problem: Evolution vs. Stability

Here's the tension: your API needs to evolve to stay relevant, but your users need stability to avoid constant rewrites. A mobile app developer who integrated your API six months ago shouldn't wake up to find their app broken because you "improved" your API.

### The Real Cost of Breaking Changes

When you break an API, you don't just break code—you break trust:

- Mobile apps that can't update fast enough crash for users
- Third-party integrations fail at 3 AM, waking up operations teams
- Partners lose confidence and start looking for alternatives
- Your support team drowns in tickets from angry developers

Yet without changes, your API becomes a fossil, unable to support new features or fix design mistakes.

## Understanding Versioning Strategies

There are three main approaches to API versioning, each with different trade-offs. Let's explore them by examining how they handle a real scenario: you need to change how dates are formatted.

### Strategy 1: URL Versioning

**The Approach**: Put the version number in the URL path.

```
GET /api/v1/users
GET /api/v2/users
```

**Why It Works**: This is the most explicit and visible approach. Developers can see exactly which version they're using just by looking at the URL. Routing is straightforward—different URLs can point to completely different implementations.

**Real Example**:
```javascript
// Version 1: dates as Unix timestamps
GET /api/v1/orders/123
{
  "id": "123",
  "created": 1704724800,  // Unix timestamp
  "status": "shipped"
}

// Version 2: dates as ISO 8601 strings
GET /api/v2/orders/123
{
  "id": "123",
  "created": "2024-01-08T12:00:00Z",  // ISO 8601
  "status": "shipped"
}
```

**When to Use**: URL versioning works best when:
- You're building a public API that many developers will consume
- You want version selection to be obvious and hard to mess up
- You're okay with maintaining multiple codebases or routing logic
- Your infrastructure makes routing by URL path simple (like with API gateways)

**The Downside**: URLs change between versions, which means you're creating completely new resources from REST's perspective. You also need to maintain routing for each version.

**Standards Reference**: While HTTP doesn't mandate how you structure URLs (RFC 3986 just defines URI syntax), this approach aligns with the principle that different resources should have different identifiers.

### Strategy 2: Header Versioning

**The Approach**: Keep URLs the same, but specify version in HTTP headers.

```
GET /api/users
Accept: application/vnd.myapi.v2+json
```

Or with a custom header:
```
GET /api/users
API-Version: 2
```

**Why It Works**: This follows HTTP's content negotiation pattern (RFC 7231). The same resource can have different representations based on what the client requests. It's more "RESTful" because the resource identifier (URL) doesn't change—only the representation does.

**Real Example**:
```javascript
// Client requests version 1
GET /api/orders/123
Accept: application/vnd.myapi.v1+json

Response:
{
  "id": "123",
  "created": 1704724800,
  "status": "shipped"
}

// Client requests version 2
GET /api/orders/123
Accept: application/vnd.myapi.v2+json

Response:
{
  "id": "123",
  "created": "2024-01-08T12:00:00Z",
  "status": "shipped"
}
```

**When to Use**: Header versioning shines when:
- You want clean, stable URLs
- Your API design treats versions as different representations of the same resource
- You're comfortable with content negotiation patterns
- You have sophisticated clients who can manage headers

**The Downside**: Headers are invisible in browser address bars and harder to test with simple tools like cURL (though not impossible). Developers might forget to set them, leading to unexpected behavior.

**Standards Reference**: This approach leverages HTTP's built-in content negotiation (RFC 7231, Section 5.3). The `Accept` header was specifically designed for this kind of versioning.

### Strategy 3: Query Parameter Versioning

**The Approach**: Add version as a query parameter.

```
GET /api/users?version=2
```

**Why It Works**: It's simple to implement and understand. The version is visible in the URL (good) but doesn't change the resource path (also good?).

**Real Example**:
```javascript
// Version 1
GET /api/orders/123?version=1
{
  "id": "123",
  "created": 1704724800,
  "status": "shipped"
}

// Version 2
GET /api/orders/123?version=2
{
  "id": "123",
  "created": "2024-01-08T12:00:00Z",
  "status": "shipped"
}
```

**When to Use**: Query parameters work when:
- You want something simpler than header-based versioning
- You need versions to be visible in URLs
- You're okay with versions mixing with other query parameters

**The Downside**: Query parameters typically modify resource selection or representation, not the API version. This can lead to confusion: is `?version=2` selecting a different resource or requesting a different format? Also, these URLs become harder to cache effectively.

**Standards Reference**: RFC 3986 defines query strings for resource identification, but using them for versioning is more convention than standard.

## Semantic Versioning for APIs

Regardless of which strategy you choose, you need a system for deciding when to bump version numbers. This is where semantic versioning principles help.

### The Version Number Format

```
MAJOR.MINOR.PATCH
  2  .  3  .  1
```

**MAJOR**: Breaking changes that require client updates
- Removing an endpoint
- Removing or renaming a field
- Changing field types (string to number)
- Changing URL structures
- Changing authentication methods

**MINOR**: New features that don't break existing clients
- Adding new endpoints
- Adding new optional fields to requests
- Adding new fields to responses (clients should ignore unknown fields)
- Adding new optional query parameters

**PATCH**: Bug fixes and minor improvements
- Fixing incorrect behavior
- Performance improvements
- Documentation updates

**Critical Rule**: Most API versioning uses only the MAJOR version in the URL or headers (`v1`, `v2`, `v3`). The MINOR and PATCH numbers are tracked internally but don't require clients to change.

### Why? The Compatibility Contract

Here's the key insight: if you design your API right, MINOR and PATCH changes are invisible to existing clients:

```javascript
// Your API at v2.0.0
GET /api/v2/users/123
{
  "id": "123",
  "name": "Alice",
  "email": "alice@example.com"
}

// You add a new field (v2.1.0 - MINOR change)
GET /api/v2/users/123
{
  "id": "123",
  "name": "Alice",
  "email": "alice@example.com",
  "phone": "+1-555-1234"  // New field!
}

// Well-designed clients ignore unknown fields
// They still work perfectly!
```

This works because of two principles:

1. **Robustness Principle** (Postel's Law): "Be conservative in what you send, be liberal in what you accept"
2. **Forward Compatibility**: Clients should ignore fields they don't understand

### Version Lifecycle Management

Every version you release becomes a commitment. Here's a practical lifecycle:

**Phase 1: Active (0-12 months)**
- Full support, all new features
- Bug fixes and security updates
- Recommended for new integrations

**Phase 2: Deprecated (12-24 months)**
- Announced via headers: `Sunset: Sat, 31 Dec 2024 23:59:59 GMT` (RFC 8594)
- Security updates only
- Warning in documentation
- Migration guide published

**Phase 3: Sunset (after 24 months)**
- Version removed or returns errors
- Sufficient warning given

**Example Deprecation Header**:
```
HTTP/1.1 200 OK
Sunset: Sat, 31 Dec 2024 23:59:59 GMT
Deprecation: true
Link: <https://api.example.com/docs/migration/v2-to-v3>; rel="sunset"

{
  "data": "..."
}
```

## Implementation Patterns

Let's look at how to actually build versioned APIs.

### Pattern 1: Separate Codebases

**The Approach**: Maintain completely separate code for each version.

```
src/
  v1/
    controllers/
    models/
    routes.js
  v2/
    controllers/
    models/
    routes.js
  shared/
    utils/
    database/
```

**Pros**:
- Clean separation
- Can refactor v2 without touching v1
- Easy to remove old versions

**Cons**:
- Code duplication
- Bug fixes might need to be applied to multiple versions
- More code to maintain

**When to Use**: When versions are significantly different or when you want complete isolation.

### Pattern 2: Transformation Layer

**The Approach**: Keep one internal model, transform for each version.

```javascript
// Internal model
class Order {
  constructor(data) {
    this.id = data.id;
    this.createdAt = data.created_at; // Always Date object internally
    this.status = data.status;
  }
}

// Version 1 transformer
function toV1(order) {
  return {
    id: order.id,
    created: Math.floor(order.createdAt.getTime() / 1000), // Unix timestamp
    status: order.status
  };
}

// Version 2 transformer
function toV2(order) {
  return {
    id: order.id,
    created: order.createdAt.toISOString(), // ISO 8601
    status: order.status
  };
}

// Route handler
app.get('/api/:version/orders/:id', async (req, res) => {
  const order = await Order.findById(req.params.id);
  
  const transformer = req.params.version === 'v1' ? toV1 : toV2;
  res.json(transformer(order));
});
```

**Pros**:
- Single source of truth for business logic
- Bug fixes automatically apply to all versions
- Less code duplication

**Cons**:
- Transformers can get complex
- Hard to handle significantly different versions
- Testing needs to cover all transformations

**When to Use**: When versions differ mainly in representation, not logic.

### Pattern 3: Feature Flags

**The Approach**: Use runtime flags to enable/disable features per version.

```javascript
const features = {
  v1: {
    isoDateFormat: false,
    includeMetadata: false,
    expandedErrors: false
  },
  v2: {
    isoDateFormat: true,
    includeMetadata: true,
    expandedErrors: true
  }
};

function formatOrder(order, version) {
  const config = features[version];
  
  return {
    id: order.id,
    created: config.isoDateFormat 
      ? order.createdAt.toISOString() 
      : Math.floor(order.createdAt.getTime() / 1000),
    status: order.status,
    ...(config.includeMetadata && { 
      metadata: order.metadata 
    })
  };
}
```

**Pros**:
- Flexible
- Can gradually roll out features
- Easy to A/B test

**Cons**:
- Can lead to spaghetti code
- Hard to reason about all combinations
- Removing old versions is tricky

**When to Use**: For gradual rollouts or when versions have subtle differences.

## Making Version Changes Safe

When you need to introduce a breaking change, how do you do it safely?

### Step 1: Expand Phase

Add the new field/behavior alongside the old:

```javascript
// v2.0 - old way (created is Unix timestamp)
{
  "id": "123",
  "created": 1704724800,
  "status": "shipped"
}

// v2.1 - MINOR: add new field, keep old (EXPAND)
{
  "id": "123",
  "created": 1704724800,  // Keep old field
  "created_at": "2024-01-08T12:00:00Z",  // Add new field
  "status": "shipped"
}
```

### Step 2: Migrate Phase

Give clients time to migrate (6-12 months):
- Update documentation
- Send emails to registered developers
- Add deprecation headers
- Monitor usage of old field

### Step 3: Contract Phase

In next MAJOR version, remove the old field:

```javascript
// v3.0 - MAJOR: remove old field (CONTRACT)
{
  "id": "123",
  "created_at": "2024-01-08T12:00:00Z",  // Only new field
  "status": "shipped"
}
```

This is the "Expand-Contract" pattern. Never go straight from old to new in a breaking way.

## Real-World Example: Stripe's Versioning

Stripe has one of the most mature API versioning strategies. Let's see what we can learn:

**Version Format**: Date-based (e.g., `2024-01-08`)

```
GET /v1/charges
Stripe-Version: 2024-01-08
```

**Why Dates?**: Every change gets a dated version. Developers can upgrade at their own pace.

**Compatibility**: Stripe maintains old versions for *years*. They've never broken backward compatibility since launch.

**Account-Level Pinning**: Each API key is pinned to a version. You can upgrade when ready:

```javascript
// Your account created in 2023
// Your API calls use 2023-01-01 version by default

// When you're ready, override for specific calls
fetch('https://api.stripe.com/v1/charges', {
  headers: {
    'Stripe-Version': '2024-01-08'  // Opt into new version
  }
});
```

**What This Teaches Us**:
- Pin versions to accounts/API keys, not just requests
- Use dates for fine-grained version control
- Never force upgrades
- Provide upgrade tools and testing environments

## Common Pitfalls and How to Avoid Them

### Pitfall 1: Too Many Versions

**The Problem**: Maintaining v1, v2, v3, v4, v5 simultaneously becomes unsustainable.

**The Solution**: 
- Set clear deprecation timelines
- Only support 2-3 major versions at once
- Make minor/patch changes backward compatible

### Pitfall 2: Version Leakage

**The Problem**: Internal systems reference specific versions, creating tight coupling.

```javascript
// Bad: internal code knows about versions
class OrderService {
  formatForV1(order) { ... }
  formatForV2(order) { ... }
  formatForV3(order) { ... }
}
```

**The Solution**: Keep version logic at the API boundary:

```javascript
// Good: internal code version-agnostic
class OrderService {
  getOrder(id) {
    return this.database.findOne(id); // Returns domain model
  }
}

// Transformation happens in API layer
app.get('/api/:version/orders/:id', async (req, res) => {
  const order = await orderService.getOrder(req.params.id);
  const transformer = getTransformer(req.params.version);
  res.json(transformer(order));
});
```

### Pitfall 3: Incomplete Version Coverage

**The Problem**: You version endpoints but not error responses, headers, or webhooks.

**The Solution**: Version everything:
- API responses
- Error formats
- Webhook payloads
- WebSocket messages
- Rate limit headers

### Pitfall 4: No Migration Path

**The Problem**: You announce "v1 is deprecated" but give no guidance on migration.

**The Solution**: Provide:
- Detailed migration guides
- Code examples for common changes
- Automated migration tools where possible
- Testing environment with new version

## Decision Framework

Still not sure which versioning strategy to use? Use this decision tree:

**Question 1: Is this a public API for external developers?**
- Yes → Use URL versioning (`/api/v1/users`)
- No (internal only) → Header versioning might be fine

**Question 2: How often will you make breaking changes?**
- Rarely (once a year or less) → URL versioning
- Frequently (monthly) → Consider if you're making too many breaking changes!

**Question 3: How sophisticated are your clients?**
- Mix of skill levels → URL versioning (most visible)
- All experienced developers → Header versioning (more "REST-ful")

**Question 4: How long will you support old versions?**
- Years → URL versioning (easier to maintain separate)
- Months → Transformation layer might work

**For most teams**: Start with URL versioning in the format `/api/v{MAJOR}/resource`. It's explicit, easy to understand, and hard to mess up.

## Summary: The Versioning Checklist

When planning your versioning strategy:

- [ ] Choose a versioning scheme (URL, header, or query)
- [ ] Define what constitutes MAJOR vs. MINOR changes
- [ ] Set deprecation timeline (recommended: 12-24 months)
- [ ] Implement version detection in your routing
- [ ] Add deprecation headers to old versions
- [ ] Create migration guides for each major version
- [ ] Monitor usage of different versions
- [ ] Plan removal strategy for old versions
- [ ] Version not just responses but errors, webhooks, everything
- [ ] Test version transitions thoroughly

The goal isn't to never change your API—it's to change it without breaking trust.

## Further Reading

- **RFC 7231**: HTTP Semantics (covers content negotiation)
- **RFC 8594**: The Sunset HTTP Header (deprecation signaling)
- **Semantic Versioning**: https://semver.org
- **REST Dissertation**: Roy Fielding's original REST definition
- **API Evolution Patterns**: Martin Fowler's refactoring catalog

---

**Next**: Learn how to communicate failures gracefully in [Error Handling](./error-handling.md).

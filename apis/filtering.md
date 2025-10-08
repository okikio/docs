# API Filtering: Querying Exactly What You Need

> **Your users don't want all your data—they want *their* data.**

Imagine you're building a task management app. Your API has a `/tasks` endpoint that returns... everything. All 50,000 tasks from all users. Every single time. Your mobile app crashes because it ran out of memory. Your server buckles under the load. Your users delete your app.

This is what happens without filtering. Good filtering transforms your API from a data firehose into a surgical tool that delivers exactly what users need, nothing more, nothing less.

## The Problem: Data Overload

Without filtering, every API call becomes a liability:

**Performance Impact**:
- Massive response payloads (megabytes of JSON)
- Slow network transfers
- Client-side processing overhead
- Database queries scanning entire tables

**User Experience**:
- Long loading times
- App crashes from memory exhaustion
- Wasted bandwidth (critical on mobile)
- Battery drain from processing unused data

**Cost**:
- Higher server compute costs
- Increased bandwidth bills
- Database performance degradation
- Poor scalability

Filtering solves all of this by letting clients specify exactly what data they need.

## Query Parameter Basics

The standard way to filter REST APIs is through URL query parameters:

```
GET /api/tasks?status=open&priority=high
```

This reads naturally: "Get me tasks where status is 'open' AND priority is 'high'."

### Simple Equality Filters

**Single condition**:
```javascript
// Get all active users
GET /api/users?status=active

// Server implementation
app.get('/api/users', async (req, res) => {
  const { status } = req.query;
  
  const users = await db.users.find({ status });
  res.json({ users });
});
```

**Multiple conditions (AND logic)**:
```javascript
// Get high-priority open tasks
GET /api/tasks?status=open&priority=high

// Server implementation
app.get('/api/tasks', async (req, res) => {
  const filters = {};
  
  if (req.query.status) filters.status = req.query.status;
  if (req.query.priority) filters.priority = req.query.priority;
  
  const tasks = await db.tasks.find(filters);
  res.json({ tasks });
});
```

**Multiple values (OR logic)**:
```javascript
// Get tasks that are either 'open' or 'in_progress'
GET /api/tasks?status=open,in_progress

// Server implementation
app.get('/api/tasks', async (req, res) => {
  const { status } = req.query;
  
  const filters = {};
  if (status) {
    // Split comma-separated values
    filters.status = { $in: status.split(',') };
  }
  
  const tasks = await db.tasks.find(filters);
  res.json({ tasks });
});
```

## Advanced Filter Operators

For more complex queries, use operator-based filtering:

### Comparison Operators

```javascript
// Get users older than 18
GET /api/users?age[gte]=18

// Get products between $100 and $500
GET /api/products?price[gte]=100&price[lte]=500

// Get tasks created after a specific date
GET /api/tasks?created_at[gt]=2024-01-01T00:00:00Z
```

**Server implementation**:
```javascript
const OPERATORS = {
  'eq': '$eq',      // Equal
  'ne': '$ne',      // Not equal
  'gt': '$gt',      // Greater than
  'gte': '$gte',    // Greater than or equal
  'lt': '$lt',      // Less than
  'lte': '$lte',    // Less than or equal
  'in': '$in',      // In array
  'nin': '$nin'     // Not in array
};

function parseFilters(query) {
  const filters = {};
  
  for (const [key, value] of Object.entries(query)) {
    // Check if key contains an operator: field[operator]
    const match = key.match(/^(.+)\[(.+)\]$/);
    
    if (match) {
      const field = match[1];
      const operator = match[2];
      
      if (OPERATORS[operator]) {
        if (!filters[field]) filters[field] = {};
        
        // Convert value to appropriate type
        let parsedValue = value;
        if (operator === 'in' || operator === 'nin') {
          parsedValue = value.split(',');
        } else if (!isNaN(value)) {
          parsedValue = Number(value);
        }
        
        filters[field][OPERATORS[operator]] = parsedValue;
      }
    } else {
      // Simple equality
      filters[key] = value;
    }
  }
  
  return filters;
}

app.get('/api/products', async (req, res) => {
  const filters = parseFilters(req.query);
  
  const products = await db.products.find(filters);
  res.json({ products });
});
```

### Pattern Matching

```javascript
// Find users whose name starts with 'John'
GET /api/users?name[like]=John*

// Find users whose email contains '@gmail.com'
GET /api/users?email[contains]=@gmail.com

// Case-insensitive search
GET /api/users?name[ilike]=john

// Server implementation
function parseFilters(query) {
  const filters = {};
  
  for (const [key, value] of Object.entries(query)) {
    const match = key.match(/^(.+)\[(.+)\]$/);
    
    if (match) {
      const field = match[1];
      const operator = match[2];
      
      if (operator === 'like') {
        // Convert wildcard to regex
        const pattern = value.replace(/\*/g, '.*');
        filters[field] = { $regex: `^${pattern}$` };
      } else if (operator === 'ilike') {
        const pattern = value.replace(/\*/g, '.*');
        filters[field] = { $regex: `^${pattern}$`, $options: 'i' };
      } else if (operator === 'contains') {
        filters[field] = { $regex: value, $options: 'i' };
      }
    } else {
      filters[key] = value;
    }
  }
  
  return filters;
}
```

## Nested Field Filtering

For complex data structures, support filtering on nested fields:

```javascript
// Filter by nested field using dot notation
GET /api/orders?customer.country=US
GET /api/orders?shipping_address.city=Seattle

// Filter by array elements
GET /api/orders?items[0].product_id=prod_123

// Server implementation
function parseNestedFilters(query) {
  const filters = {};
  
  for (const [key, value] of Object.entries(query)) {
    // Support dot notation
    if (key.includes('.')) {
      filters[key] = value;
    } else {
      filters[key] = value;
    }
  }
  
  return filters;
}
```

## Field Selection (Sparse Fieldsets)

Let clients choose which fields to return:

```javascript
// Return only specific fields
GET /api/users?fields=id,name,email

// Exclude specific fields
GET /api/users?exclude=password_hash,secret_key

// Server implementation
app.get('/api/users', async (req, res) => {
  const { fields, exclude } = req.query;
  
  let projection = {};
  
  if (fields) {
    // Include only specified fields
    fields.split(',').forEach(field => {
      projection[field] = 1;
    });
  } else if (exclude) {
    // Exclude specified fields
    exclude.split(',').forEach(field => {
      projection[field] = 0;
    });
  }
  
  const users = await db.users.find({}, projection);
  res.json({ users });
});
```

**Benefits**:
- Reduces payload size
- Improves performance
- Saves bandwidth
- Increases security (don't leak sensitive fields)

## Full-Text Search

For searching across multiple text fields:

```javascript
// Search across all text fields
GET /api/products?q=wireless+headphones

// Search with filters
GET /api/products?q=headphones&category=electronics&price[lte]=100

// Server implementation (using MongoDB text index)
app.get('/api/products', async (req, res) => {
  const { q } = req.query;
  const filters = parseFilters(req.query);
  
  if (q) {
    filters.$text = { $search: q };
  }
  
  const products = await db.products.find(filters);
  
  // If using text search, include relevance score
  if (q) {
    const productsWithScore = products.map(product => ({
      ...product,
      score: product.score
    }));
    return res.json({ products: productsWithScore });
  }
  
  res.json({ products });
});
```

## Date and Time Filtering

Dates require special handling:

```javascript
// Tasks created on a specific date
GET /api/tasks?created_date=2024-01-15

// Tasks created in a date range
GET /api/tasks?created_at[gte]=2024-01-01&created_at[lt]=2024-02-01

// Relative dates (last 7 days)
GET /api/tasks?created_at[gte]=7d

// Server implementation
function parseDateFilter(value) {
  // Relative date: 7d, 30d, 1h, etc.
  const relativeMatch = value.match(/^(\d+)([dhm])$/);
  
  if (relativeMatch) {
    const amount = parseInt(relativeMatch[1]);
    const unit = relativeMatch[2];
    
    const now = new Date();
    let ms = 0;
    
    if (unit === 'd') ms = amount * 24 * 60 * 60 * 1000;
    if (unit === 'h') ms = amount * 60 * 60 * 1000;
    if (unit === 'm') ms = amount * 60 * 1000;
    
    return new Date(now.getTime() - ms);
  }
  
  // ISO 8601 date string
  return new Date(value);
}
```

## Sorting

Sorting is closely related to filtering:

```javascript
// Sort by single field (ascending)
GET /api/users?sort=name

// Sort descending
GET /api/users?sort=-created_at

// Multiple sort fields
GET /api/users?sort=-priority,created_at

// Server implementation
function parseSort(sortParam) {
  if (!sortParam) return {};
  
  const sort = {};
  
  sortParam.split(',').forEach(field => {
    if (field.startsWith('-')) {
      sort[field.slice(1)] = -1;  // Descending
    } else {
      sort[field] = 1;  // Ascending
    }
  });
  
  return sort;
}

app.get('/api/users', async (req, res) => {
  const filters = parseFilters(req.query);
  const sort = parseSort(req.query.sort);
  
  const users = await db.users.find(filters).sort(sort);
  res.json({ users });
});
```

## Security Considerations

Filtering opens potential security vulnerabilities:

### 1. SQL Injection

```javascript
// ❌ Vulnerable to SQL injection
app.get('/api/users', async (req, res) => {
  const { name } = req.query;
  
  // DON'T DO THIS!
  const query = `SELECT * FROM users WHERE name = '${name}'`;
  const users = await db.query(query);
});

// ✅ Use parameterized queries
app.get('/api/users', async (req, res) => {
  const { name } = req.query;
  
  const users = await db.query(
    'SELECT * FROM users WHERE name = ?',
    [name]
  );
});
```

### 2. NoSQL Injection

```javascript
// ❌ Vulnerable to NoSQL injection
app.get('/api/users', async (req, res) => {
  const filters = req.query;  // User can send { password: { $ne: null } }
  
  const users = await db.users.find(filters);
});

// ✅ Validate and sanitize
function sanitizeFilters(query) {
  const allowed = ['status', 'role', 'created_at'];
  const filters = {};
  
  for (const [key, value] of Object.entries(query)) {
    // Only allow specific fields
    if (allowed.includes(key)) {
      // Ensure value is not an object (prevents operator injection)
      if (typeof value === 'string' || typeof value === 'number') {
        filters[key] = value;
      }
    }
  }
  
  return filters;
}
```

### 3. Field Exposure

```javascript
// ❌ Allows querying sensitive fields
GET /api/users?password_hash=abc123

// ✅ Whitelist filterable fields
const FILTERABLE_FIELDS = {
  'users': ['status', 'role', 'created_at', 'country'],
  'tasks': ['status', 'priority', 'assignee', 'due_date']
};

function validateFilters(resource, query) {
  const allowed = FILTERABLE_FIELDS[resource] || [];
  const filters = {};
  
  for (const [key, value] of Object.entries(query)) {
    if (allowed.includes(key)) {
      filters[key] = value;
    }
  }
  
  return filters;
}
```

### 4. Performance Attacks

```javascript
// Attacker sends complex query to overload server
GET /api/users?age[gte]=1&age[lte]=1&status=active&role=user&...

// ✅ Limit query complexity
function validateComplexity(query) {
  const filterCount = Object.keys(query).length;
  
  if (filterCount > 10) {
    throw new Error('Too many filter parameters (max 10)');
  }
  
  // Limit array sizes
  for (const value of Object.values(query)) {
    if (typeof value === 'string' && value.includes(',')) {
      const items = value.split(',');
      if (items.length > 50) {
        throw new Error('Too many values in filter (max 50)');
      }
    }
  }
}
```

## Documentation Example

Document your filtering capabilities clearly:

```markdown
## Filtering

All list endpoints support filtering via query parameters.

### Basic Filters

```
GET /api/tasks?status=open
GET /api/tasks?status=open&priority=high
```

### Comparison Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `eq` | Equal | `?age[eq]=25` |
| `ne` | Not equal | `?status[ne]=deleted` |
| `gt` | Greater than | `?age[gt]=18` |
| `gte` | Greater than or equal | `?price[gte]=100` |
| `lt` | Less than | `?age[lt]=65` |
| `lte` | Less than or equal | `?price[lte]=1000` |
| `in` | In array | `?status[in]=open,pending` |

### Filterable Fields

**Tasks**:
- `status` (string): Task status
- `priority` (string): Task priority  
- `assignee` (string): User ID
- `due_date` (date): Due date (ISO 8601)
- `created_at` (date): Creation date (ISO 8601)

**Example**:
```
GET /api/tasks?status[in]=open,in_progress&priority=high&due_date[lte]=2024-12-31
```
```

## Best Practices

- [ ] Whitelist filterable fields
- [ ] Validate all filter values
- [ ] Use parameterized queries (prevent injection)
- [ ] Limit query complexity
- [ ] Index frequently filtered fields
- [ ] Document all filtering options
- [ ] Return clear errors for invalid filters
- [ ] Support common operators (eq, ne, gt, lt, in)
- [ ] Consider pagination with filtering
- [ ] Test filtering with various inputs

## Summary

Effective filtering transforms your API from returning everything to returning exactly what's needed. This improves performance, reduces bandwidth, and creates better user experiences.

The key principles:
1. **Let clients specify what they want** via query parameters
2. **Support common patterns** (equality, comparison, ranges)
3. **Secure against abuse** (validate, sanitize, limit complexity)
4. **Document clearly** what's filterable and how

Well-implemented filtering is the difference between an API that scales and one that collapses under its own data.

## Further Reading

- **JSON API Specification** - Filtering recommendations
- **OData** - Advanced query syntax
- **GraphQL** - Alternative with built-in filtering
- **Elasticsearch** - Full-text search capabilities

---

**Next**: Learn how to handle large datasets efficiently in [Pagination](./pagination.md).

# Pagination: Managing Large Datasets

> **Don't send 100,000 records when the user can only see 20 at a time.**

Your API just returned 2 million user records in a single response. The JSON payload is 500MB. The mobile app crashed. The web browser tab froze. Your server ran out of memory. All because someone called `GET /api/users` without pagination.

Pagination isn't optional—it's essential for any API that returns collections. It's how you make large datasets manageable, performant, and actually usable.

## The Problem Without Pagination

**Memory Exhaustion**:
- Server loads entire dataset into memory
- Client receives massive payloads
- Both may crash or freeze

**Performance**:
- Slow database queries (full table scans)
- Large network transfers
- JSON parsing overhead on client

**User Experience**:
- Long wait times
- App unresponsiveness
- Wasted bandwidth

Pagination solves this by breaking large result sets into manageable chunks.

## Three Pagination Strategies

There are three main approaches, each with different trade-offs.

### 1. Offset-Based Pagination

**The Pattern**: Skip N records, return M records.

```javascript
// Page 1: First 20 items
GET /api/users?limit=20&offset=0

// Page 2: Next 20 items
GET /api/users?limit=20&offset=20

// Page 3: Next 20 items
GET /api/users?limit=20&offset=40
```

**How It Works**:
```javascript
app.get('/api/users', async (req, res) => {
  const limit = parseInt(req.query.limit) || 20;
  const offset = parseInt(req.query.offset) || 0;
  
  // Limit maximum page size
  const maxLimit = 100;
  const safeLimit = Math.min(limit, maxLimit);
  
  const users = await db.users
    .find()
    .skip(offset)
    .limit(safeLimit);
  
  const total = await db.users.countDocuments();
  
  res.json({
    data: users,
    pagination: {
      limit: safeLimit,
      offset,
      total,
      total_pages: Math.ceil(total / safeLimit)
    }
  });
});
```

**Response Format**:
```json
{
  "data": [
    { "id": "21", "name": "User 21" },
    { "id": "22", "name": "User 22" }
  ],
  "pagination": {
    "limit": 20,
    "offset": 20,
    "total": 150,
    "total_pages": 8
  },
  "links": {
    "first": "/api/users?limit=20&offset=0",
    "prev": "/api/users?limit=20&offset=0",
    "next": "/api/users?limit=20&offset=40",
    "last": "/api/users?limit=20&offset=140"
  }
}
```

**Pros**:
- Simple to implement
- Can jump to any page directly
- Shows total count and pages
- Familiar pattern

**Cons**:
- Performance degrades with large offsets (database still scans skipped rows)
- Inconsistent results if data changes between requests
- Inefficient for large datasets

**When to Use**: Small to medium datasets where users need random access to pages.

### 2. Page-Based Pagination

**The Pattern**: Request by page number instead of offset.

```javascript
// Page 1
GET /api/users?page=1&per_page=20

// Page 2
GET /api/users?page=2&per_page=20
```

**Implementation**:
```javascript
app.get('/api/users', async (req, res) => {
  const perPage = parseInt(req.query.per_page) || 20;
  const page = parseInt(req.query.page) || 1;
  
  const maxPerPage = 100;
  const safePerPage = Math.min(perPage, maxPerPage);
  
  // Calculate offset from page number
  const offset = (page - 1) * safePerPage;
  
  const users = await db.users
    .find()
    .skip(offset)
    .limit(safePerPage);
  
  const total = await db.users.countDocuments();
  const totalPages = Math.ceil(total / safePerPage);
  
  res.json({
    data: users,
    pagination: {
      page,
      per_page: safePerPage,
      total,
      total_pages: totalPages
    },
    links: {
      first: `/api/users?page=1&per_page=${safePerPage}`,
      prev: page > 1 ? `/api/users?page=${page - 1}&per_page=${safePerPage}` : null,
      next: page < totalPages ? `/api/users?page=${page + 1}&per_page=${safePerPage}` : null,
      last: `/api/users?page=${totalPages}&per_page=${safePerPage}`
    }
  });
});
```

**Pros**:
- More intuitive than offsets
- Same benefits as offset-based

**Cons**:
- Same performance issues as offset-based
- Just syntactic sugar over offsets

**When to Use**: Same as offset-based, but when "page" is more natural for your API.

### 3. Cursor-Based Pagination (Recommended)

**The Pattern**: Use a pointer (cursor) to mark position in the dataset.

```javascript
// First page
GET /api/users?limit=20

// Next page (cursor from previous response)
GET /api/users?limit=20&cursor=eyJpZCI6IjIwIn0=
```

**How It Works**:
```javascript
app.get('/api/users', async (req, res) => {
  const limit = parseInt(req.query.limit) || 20;
  const cursor = req.query.cursor;
  
  const maxLimit = 100;
  const safeLimit = Math.min(limit, maxLimit);
  
  const query = {};
  
  if (cursor) {
    // Decode cursor (Base64 encoded JSON)
    const decoded = JSON.parse(
      Buffer.from(cursor, 'base64').toString('utf-8')
    );
    
    // Cursor contains the ID of the last item from previous page
    query._id = { $gt: decoded.id };
  }
  
  const users = await db.users
    .find(query)
    .sort({ _id: 1 })  // Must sort by cursor field
    .limit(safeLimit + 1);  // Fetch one extra to check if more exist
  
  const hasMore = users.length > safeLimit;
  const results = users.slice(0, safeLimit);
  
  // Create cursor for next page
  let nextCursor = null;
  if (hasMore) {
    const lastItem = results[results.length - 1];
    nextCursor = Buffer.from(
      JSON.stringify({ id: lastItem._id })
    ).toString('base64');
  }
  
  res.json({
    data: results,
    pagination: {
      limit: safeLimit,
      next_cursor: nextCursor,
      has_more: hasMore
    },
    links: {
      next: nextCursor ? `/api/users?limit=${safeLimit}&cursor=${nextCursor}` : null
    }
  });
});
```

**Response Format**:
```json
{
  "data": [
    { "id": "21", "name": "User 21" },
    { "id": "22", "name": "User 22" }
  ],
  "pagination": {
    "limit": 20,
    "next_cursor": "eyJpZCI6IjQwIn0=",
    "has_more": true
  },
  "links": {
    "next": "/api/users?limit=20&cursor=eyJpZCI6IjQwIn0="
  }
}
```

**Pros**:
- Consistent results even if data changes
- Excellent performance (uses indexed fields)
- Scales to massive datasets
- No duplicate or skipped items

**Cons**:
- Can't jump to arbitrary pages
- No total count (calculating it is expensive)
- Slightly more complex to implement
- Can't go backwards easily

**When to Use**: Large datasets, real-time data, infinite scroll, feeds.

**Bi-directional Cursors**:
```javascript
// Support both forward and backward navigation
app.get('/api/users', async (req, res) => {
  const limit = parseInt(req.query.limit) || 20;
  const { cursor, direction = 'next' } = req.query;
  
  const query = {};
  let sort = { _id: 1 };
  
  if (cursor) {
    const decoded = JSON.parse(
      Buffer.from(cursor, 'base64').toString('utf-8')
    );
    
    if (direction === 'next') {
      query._id = { $gt: decoded.id };
      sort = { _id: 1 };
    } else {
      query._id = { $lt: decoded.id };
      sort = { _id: -1 };
    }
  }
  
  const users = await db.users
    .find(query)
    .sort(sort)
    .limit(limit + 1);
  
  if (direction === 'prev') {
    users.reverse();  // Reverse results for backward pagination
  }
  
  // Create both prev and next cursors
  const prevCursor = users.length > 0 ? 
    Buffer.from(JSON.stringify({ id: users[0]._id })).toString('base64') : null;
  
  const nextCursor = users.length > limit ?
    Buffer.from(JSON.stringify({ id: users[limit - 1]._id })).toString('base64') : null;
  
  res.json({
    data: users.slice(0, limit),
    pagination: {
      prev_cursor: prevCursor,
      next_cursor: nextCursor
    }
  });
});
```

## HTTP Link Headers

Include pagination links in HTTP headers per RFC 8288:

```javascript
app.get('/api/users', async (req, res) => {
  const users = await getUsers(req.query);
  const { prevUrl, nextUrl, firstUrl, lastUrl } = buildPaginationUrls(req);
  
  // Link header
  const links = [
    `<${firstUrl}>; rel="first"`,
    `<${prevUrl}>; rel="prev"`,
    `<${nextUrl}>; rel="next"`,
    `<${lastUrl}>; rel="last"`
  ].filter(Boolean).join(', ');
  
  res.set('Link', links);
  
  // Also include total count header
  res.set('X-Total-Count', users.total);
  
  res.json({ data: users.data });
});
```

**Example Response Headers**:
```
Link: <https://api.example.com/users?page=1>; rel="first",
      <https://api.example.com/users?page=2>; rel="prev",
      <https://api.example.com/users?page=4>; rel="next",
      <https://api.example.com/users?page=10>; rel="last"
X-Total-Count: 200
```

## Combining Filters and Pagination

Pagination should work seamlessly with filtering:

```javascript
// Filtered and paginated
GET /api/tasks?status=open&priority=high&limit=20&cursor=abc123

app.get('/api/tasks', async (req, res) => {
  const { status, priority, limit, cursor } = req.query;
  
  // Build filter query
  const query = {};
  if (status) query.status = status;
  if (priority) query.priority = priority;
  
  // Add cursor condition
  if (cursor) {
    const decoded = JSON.parse(
      Buffer.from(cursor, 'base64').toString('utf-8')
    );
    query._id = { $gt: decoded.id };
  }
  
  const tasks = await db.tasks
    .find(query)
    .sort({ _id: 1 })
    .limit(parseInt(limit) || 20);
  
  res.json({ data: tasks });
});
```

## Performance Optimization

### 1. Index Cursor Fields

```javascript
// MongoDB: Index the field used for cursors
db.users.createIndex({ _id: 1 });
db.tasks.createIndex({ created_at: 1, _id: 1 });

// PostgreSQL
CREATE INDEX idx_users_id ON users(id);
CREATE INDEX idx_tasks_created_at_id ON tasks(created_at, id);
```

### 2. Avoid COUNT() for Large Tables

```javascript
// ❌ Expensive on large tables
const total = await db.users.count();

// ✅ Use approximate count
const total = await db.users.estimatedDocumentCount();

// ✅ Or don't return total at all (cursor-based)
res.json({
  data: users,
  pagination: {
    has_more: true  // Just indicate if more exists
  }
});
```

### 3. Limit Maximum Page Size

```javascript
const DEFAULT_LIMIT = 20;
const MAX_LIMIT = 100;

function getLimit(requestedLimit) {
  const limit = parseInt(requestedLimit) || DEFAULT_LIMIT;
  return Math.min(Math.max(limit, 1), MAX_LIMIT);
}
```

## Error Handling

```javascript
app.get('/api/users', async (req, res) => {
  try {
    const limit = getLimit(req.query.limit);
    const offset = parseInt(req.query.offset) || 0;
    
    // Validate offset isn't too large
    if (offset > 10000) {
      return res.status(400).json({
        error: {
          type: 'invalid_offset',
          message: 'Offset too large. Use cursor-based pagination for deep pagination.',
          max_offset: 10000
        }
      });
    }
    
    const users = await db.users.find().skip(offset).limit(limit);
    res.json({ data: users });
  } catch (err) {
    res.status(500).json({
      error: {
        type: 'pagination_error',
        message: 'Failed to paginate results'
      }
    });
  }
});
```

## Best Practices

- [ ] Set reasonable default page sizes (20-50 items)
- [ ] Enforce maximum page sizes (100-200 items)
- [ ] Use cursor-based pagination for large datasets
- [ ] Include pagination metadata in responses
- [ ] Provide navigation links (first, prev, next, last)
- [ ] Index fields used for cursors and sorting
- [ ] Handle edge cases (empty results, invalid cursors)
- [ ] Document pagination parameters clearly
- [ ] Consider infinite scroll vs. page numbers UI
- [ ] Test with large datasets

## Summary

Pagination is essential for APIs that return collections. The three main approaches are:

1. **Offset-based**: Simple, familiar, but performance issues with large offsets
2. **Page-based**: Syntactic sugar over offsets
3. **Cursor-based**: Best performance and consistency, but no random page access

For most modern APIs, especially those with large or real-time data, cursor-based pagination is the right choice.

## Further Reading

- **RFC 8288**: Web Linking (Link header)
- **JSON API**: Pagination spec
- **GraphQL**: Cursor-based pagination patterns
- **Database Performance**: Index optimization

---

**Next**: Protect your API from abuse with [Rate Limiting](./rate-limiting.md).

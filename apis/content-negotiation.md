# Content Negotiation: Supporting Multiple Formats

> **Your API should speak the language your clients understand.**

Your API returns JSON. Perfect—until a client needs XML for their legacy system. Or CSV for Excel imports. Or Protocol Buffers for performance. You could create separate endpoints for each format (`/api/users.json`, `/api/users.xml`), but that's duplication and maintenance hell.

Content negotiation solves this elegantly: one endpoint, multiple formats, client chooses what they need.

## The HTTP Foundation

Content negotiation is built into HTTP via the `Accept` header (RFC 7231):

```javascript
// Client requests JSON
GET /api/users
Accept: application/json

// Client requests XML
GET /api/users
Accept: application/xml

// Client requests CSV
GET /api/users
Accept: text/csv
```

The server examines the `Accept` header and returns the appropriate format.

## Common Media Types

**JSON** (Default for most APIs)
```
Content-Type: application/json
```

**XML**
```
Content-Type: application/xml
```

**CSV**
```
Content-Type: text/csv
```

**HTML**
```
Content-Type: text/html
```

**Protocol Buffers**
```
Content-Type: application/x-protobuf
```

**MessagePack**
```
Content-Type: application/x-msgpack
```

## Implementation

### Basic Content Negotiation

```javascript
app.get('/api/users', async (req, res) => {
  const users = await db.users.find();
  
  const acceptHeader = req.headers.accept || 'application/json';
  
  if (acceptHeader.includes('application/json')) {
    res.type('application/json');
    res.json({ users });
  } else if (acceptHeader.includes('application/xml')) {
    res.type('application/xml');
    const xml = convertToXML(users);
    res.send(xml);
  } else if (acceptHeader.includes('text/csv')) {
    res.type('text/csv');
    const csv = convertToCSV(users);
    res.send(csv);
  } else {
    res.status(406).json({
      error: {
        type: 'unsupported_media_type',
        message: 'Requested format not supported',
        supported: ['application/json', 'application/xml', 'text/csv']
      }
    });
  }
});
```

### Using Express Middleware

```javascript
const accepts = require('accepts');

app.get('/api/users', async (req, res) => {
  const users = await db.users.find();
  
  const accept = accepts(req);
  const format = accept.type(['json', 'xml', 'csv']);
  
  switch (format) {
    case 'json':
      res.json({ users });
      break;
    case 'xml':
      res.type('xml').send(toXML(users));
      break;
    case 'csv':
      res.type('csv').send(toCSV(users));
      break;
    default:
      res.status(406).json({
        error: 'Unsupported format'
      });
  }
});
```

## Format Converters

### JSON to XML

```javascript
const js2xmlparser = require('js2xmlparser');

function toXML(data) {
  return js2xmlparser.parse('users', { user: data });
}

// Example output
/*
<?xml version="1.0"?>
<users>
  <user>
    <id>1</id>
    <name>Alice</name>
  </user>
  <user>
    <id>2</id>
    <name>Bob</name>
  </user>
</users>
*/
```

### JSON to CSV

```javascript
const { parse } = require('json2csv');

function toCSV(data) {
  const fields = ['id', 'name', 'email', 'created_at'];
  return parse(data, { fields });
}

// Example output
/*
id,name,email,created_at
1,Alice,alice@example.com,2024-01-01
2,Bob,bob@example.com,2024-01-02
*/
```

## Quality Values

Clients can specify preferences with quality values (q):

```
Accept: application/json;q=1.0, application/xml;q=0.8, text/*;q=0.5
```

This means: "I prefer JSON (q=1.0), but I'll accept XML (q=0.8) or any text format (q=0.5)."

```javascript
const accepts = require('accepts');

app.get('/api/users', async (req, res) => {
  const users = await db.users.find();
  
  // Automatically handles quality values
  const format = accepts(req).type(['json', 'xml', 'csv']);
  
  // Respond based on best match
  res.format({
    'application/json': () => {
      res.json({ users });
    },
    'application/xml': () => {
      res.type('xml').send(toXML(users));
    },
    'text/csv': () => {
      res.type('csv').send(toCSV(users));
    },
    default: () => {
      res.status(406).send('Not Acceptable');
    }
  });
});
```

## Compression Negotiation

Along with format, negotiate compression:

```
Accept-Encoding: gzip, deflate, br
```

```javascript
const compression = require('compression');

app.use(compression({
  filter: (req, res) => {
    if (req.headers['x-no-compression']) {
      return false;
    }
    return compression.filter(req, res);
  },
  threshold: 1024  // Only compress responses > 1KB
}));
```

## Language Negotiation

For internationalized APIs:

```
Accept-Language: en-US, en;q=0.9, es;q=0.8
```

```javascript
app.get('/api/messages', (req, res) => {
  const acceptLanguage = req.headers['accept-language'];
  const language = parseLanguage(acceptLanguage); // en-US, en, or es
  
  const messages = getMessages(language);
  res.json({ messages });
});
```

## API Versioning via Media Types

Combine content negotiation with versioning:

```
Accept: application/vnd.myapi.v2+json
```

```javascript
app.get('/api/users', async (req, res) => {
  const accept = req.headers.accept || '';
  
  let version = 1;
  if (accept.includes('vnd.myapi.v2')) {
    version = 2;
  }
  
  const users = await db.users.find();
  
  if (version === 2) {
    res.json({ users: transformV2(users) });
  } else {
    res.json({ users: transformV1(users) });
  }
});
```

## Best Practices

- [ ] Default to JSON if no Accept header specified
- [ ] Support at least JSON (required for modern APIs)
- [ ] Return 406 Not Acceptable for unsupported formats
- [ ] Include Content-Type in all responses
- [ ] Document supported formats clearly
- [ ] Use standard media types (IANA registered)
- [ ] Consider compression for large responses
- [ ] Cache responses per format
- [ ] Test all supported formats
- [ ] Handle format conversion errors gracefully

## Performance Considerations

```javascript
// Cache converted formats
const cache = new Map();

app.get('/api/users', async (req, res) => {
  const format = accepts(req).type(['json', 'xml', 'csv']);
  const cacheKey = `users:${format}`;
  
  let data = cache.get(cacheKey);
  
  if (!data) {
    const users = await db.users.find();
    
    switch (format) {
      case 'json':
        data = JSON.stringify({ users });
        break;
      case 'xml':
        data = toXML(users);
        break;
      case 'csv':
        data = toCSV(users);
        break;
    }
    
    cache.set(cacheKey, data);
    setTimeout(() => cache.delete(cacheKey), 60000); // Cache for 1 minute
  }
  
  res.type(format).send(data);
});
```

## Summary

Content negotiation allows one endpoint to serve multiple formats. Key points:

- Use `Accept` header for format selection
- Support JSON as minimum (industry standard)
- Add XML, CSV, etc. based on client needs
- Return 406 for unsupported formats
- Consider versioning via media types
- Cache converted formats for performance

This makes your API more flexible without endpoint proliferation.

## Further Reading

- **RFC 7231**: HTTP Semantics (Content Negotiation)
- **IANA Media Types**: Official media type registry
- **Accept Header**: Quality values and wildcards

---

**Next**: Make operations safe to retry with [Idempotency](./idempotency.md).

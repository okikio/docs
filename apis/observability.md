# Observability: Logging, Monitoring & Health Checks

> **If you can't see what's happening, you can't fix it.**

It's 3 AM. Your API is down. Users are complaining on Twitter. You log into your server and... nothing. No logs. No metrics. No idea what went wrong or when it started. You're debugging blind, customers are losing money, and every minute of downtime costs you.

This is life without observability. With proper logging, monitoring, and health checks, you would have seen the problem coming, been alerted automatically, and had the data to fix it in minutes instead of hours.

## The Three Pillars of Observability

**Logs**: What happened and when
**Metrics**: How the system is performing
**Traces**: How requests flow through the system

Together, they give you complete visibility into your API.

## Structured Logging

Traditional logging:
```
User logged in
Error in database
Request took 523ms
```

This is useless. What user? Which database? Which request?

Structured logging:
```json
{
  "timestamp": "2024-01-08T14:30:00Z",
  "level": "info",
  "message": "User logged in",
  "user_id": "user_123",
  "ip": "192.168.1.1",
  "request_id": "req_abc123"
}
```

This is searchable, filterable, and actually useful.

**Implementation**:
```javascript
const winston = require('winston');

const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'app.log' })
  ]
});

// Log with context
logger.info('User logged in', {
  user_id: user.id,
  email: user.email,
  ip: req.ip,
  request_id: req.id
});

logger.error('Database query failed', {
  error: err.message,
  stack: err.stack,
  query: sqlQuery,
  request_id: req.id
});
```

## Log Levels

Use appropriate severity levels:

**ERROR** - Something failed
```javascript
logger.error('Payment processing failed', {
  error: err.message,
  user_id: req.user.id,
  amount: payment.amount,
  request_id: req.id
});
```

**WARN** - Something suspicious but not broken
```javascript
logger.warn('Rate limit approaching', {
  user_id: req.user.id,
  requests: currentCount,
  limit: maxRequests
});
```

**INFO** - Normal business events
```javascript
logger.info('Order created', {
  order_id: order.id,
  user_id: req.user.id,
  total: order.total
});
```

**DEBUG** - Detailed diagnostic info
```javascript
logger.debug('Database query executed', {
  query: sql,
  duration_ms: queryTime,
  rows_returned: results.length
});
```

## Request Logging Middleware

Log every API request:

```javascript
app.use((req, res, next) => {
  const start = Date.now();
  
  // Log request
  logger.info('Request started', {
    method: req.method,
    path: req.path,
    ip: req.ip,
    user_agent: req.headers['user-agent'],
    request_id: req.id
  });
  
  // Capture response
  res.on('finish', () => {
    const duration = Date.now() - start;
    
    logger.info('Request completed', {
      method: req.method,
      path: req.path,
      status_code: res.statusCode,
      duration_ms: duration,
      request_id: req.id,
      user_id: req.user?.id
    });
  });
  
  next();
});
```

## Error Logging

Capture all errors with full context:

```javascript
app.use((err, req, res, next) => {
  logger.error('Unhandled error', {
    error: err.message,
    stack: err.stack,
    method: req.method,
    path: req.path,
    body: req.body,
    user_id: req.user?.id,
    request_id: req.id
  });
  
  res.status(500).json({
    error: {
      type: 'internal_error',
      message: 'An error occurred',
      request_id: req.id
    }
  });
});
```

## Metrics Collection

Track key performance indicators:

```javascript
const promClient = require('prom-client');

// Create a Registry
const register = new promClient.Registry();

// HTTP request duration
const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.1, 0.5, 1, 2, 5]
});

// HTTP request count
const httpRequestCount = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code']
});

// Active connections
const activeConnections = new promClient.Gauge({
  name: 'active_connections',
  help: 'Number of active connections'
});

register.registerMetric(httpRequestDuration);
register.registerMetric(httpRequestCount);
register.registerMetric(activeConnections);

// Middleware to track metrics
app.use((req, res, next) => {
  const start = Date.now();
  
  activeConnections.inc();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    
    httpRequestDuration.labels(
      req.method,
      req.route?.path || req.path,
      res.statusCode
    ).observe(duration);
    
    httpRequestCount.labels(
      req.method,
      req.route?.path || req.path,
      res.statusCode
    ).inc();
    
    activeConnections.dec();
  });
  
  next();
});

// Expose metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

## Health Checks

Essential endpoints to monitor system health:

### Liveness Probe

"Is the service running?"

```javascript
app.get('/health/live', (req, res) => {
  res.status(200).json({
    status: 'ok',
    timestamp: new Date().toISOString()
  });
});
```

### Readiness Probe

"Is the service ready to handle requests?"

```javascript
app.get('/health/ready', async (req, res) => {
  const checks = {
    database: false,
    redis: false,
    external_api: false
  };
  
  try {
    // Check database
    await db.ping();
    checks.database = true;
    
    // Check Redis
    await redis.ping();
    checks.redis = true;
    
    // Check external API
    const apiStatus = await checkExternalAPI();
    checks.external_api = apiStatus;
    
    const healthy = Object.values(checks).every(v => v === true);
    
    res.status(healthy ? 200 : 503).json({
      status: healthy ? 'ready' : 'not_ready',
      checks,
      timestamp: new Date().toISOString()
    });
  } catch (err) {
    res.status(503).json({
      status: 'not_ready',
      checks,
      error: err.message,
      timestamp: new Date().toISOString()
    });
  }
});
```

### Detailed Health Check

```javascript
app.get('/health', async (req, res) => {
  const health = {
    status: 'healthy',
    version: process.env.VERSION || 'unknown',
    uptime: process.uptime(),
    timestamp: new Date().toISOString(),
    checks: {}
  };
  
  // Database check
  try {
    const dbStart = Date.now();
    await db.ping();
    health.checks.database = {
      status: 'healthy',
      response_time_ms: Date.now() - dbStart
    };
  } catch (err) {
    health.status = 'unhealthy';
    health.checks.database = {
      status: 'unhealthy',
      error: err.message
    };
  }
  
  // Redis check
  try {
    const redisStart = Date.now();
    await redis.ping();
    health.checks.redis = {
      status: 'healthy',
      response_time_ms: Date.now() - redisStart
    };
  } catch (err) {
    health.status = 'degraded';
    health.checks.redis = {
      status: 'unhealthy',
      error: err.message
    };
  }
  
  // Memory check
  const memUsage = process.memoryUsage();
  health.checks.memory = {
    status: memUsage.heapUsed < 1024 * 1024 * 1024 ? 'healthy' : 'warning',
    heap_used_mb: Math.round(memUsage.heapUsed / 1024 / 1024),
    heap_total_mb: Math.round(memUsage.heapTotal / 1024 / 1024)
  };
  
  const statusCode = health.status === 'healthy' ? 200 :
                      health.status === 'degraded' ? 200 : 503;
  
  res.status(statusCode).json(health);
});
```

## Distributed Tracing

Track requests across multiple services:

```javascript
const { trace, context } = require('@opentelemetry/api');
const tracer = trace.getTracer('my-api');

app.get('/api/orders/:id', async (req, res) => {
  // Create span for this request
  const span = tracer.startSpan('get_order');
  
  try {
    span.setAttribute('order.id', req.params.id);
    span.setAttribute('user.id', req.user?.id);
    
    // Database call (child span)
    const dbSpan = tracer.startSpan('db.query', { parent: span });
    const order = await db.orders.findById(req.params.id);
    dbSpan.end();
    
    // External API call (child span)
    const apiSpan = tracer.startSpan('external.api', { parent: span });
    const shipping = await getShippingStatus(order.tracking_number);
    apiSpan.end();
    
    span.setStatus({ code: 0 });  // Success
    res.json({ order, shipping });
  } catch (err) {
    span.setStatus({ code: 2, message: err.message });  // Error
    throw err;
  } finally {
    span.end();
  }
});
```

## Alerting

Set up alerts for critical issues:

```javascript
// Alert on high error rate
const errorRate = new promClient.Gauge({
  name: 'error_rate',
  help: 'Percentage of requests that resulted in errors'
});

// Calculate every minute
setInterval(async () => {
  const stats = await getRequestStats();
  const rate = stats.errors / stats.total;
  
  errorRate.set(rate);
  
  // Alert if error rate > 5%
  if (rate > 0.05) {
    await sendAlert({
      severity: 'critical',
      message: `Error rate at ${(rate * 100).toFixed(2)}%`,
      details: stats
    });
  }
}, 60000);
```

## Log Aggregation

Send logs to centralized service:

```javascript
// Using Winston with transport
const winston = require('winston');
const { Loggly } = require('winston-loggly-bulk');

const logger = winston.createLogger({
  transports: [
    new winston.transports.Console(),
    new Loggly({
      token: process.env.LOGGLY_TOKEN,
      subdomain: process.env.LOGGLY_SUBDOMAIN,
      tags: ['api', 'production'],
      json: true
    })
  ]
});
```

## Best Practices

- [ ] Use structured logging (JSON format)
- [ ] Include request IDs in all logs
- [ ] Log at appropriate levels (ERROR, WARN, INFO, DEBUG)
- [ ] Implement health check endpoints
- [ ] Track key metrics (latency, error rate, throughput)
- [ ] Set up alerting for critical issues
- [ ] Use log aggregation service
- [ ] Implement distributed tracing
- [ ] Monitor resource usage (CPU, memory, disk)
- [ ] Test health checks regularly
- [ ] Set up dashboards (Grafana, Datadog)
- [ ] Rotate log files to prevent disk filling

## Monitoring Checklist

**Application Metrics**:
- Request rate (requests/second)
- Error rate (percentage)
- Response time (p50, p95, p99)
- Active connections
- Queue depth

**System Metrics**:
- CPU usage
- Memory usage
- Disk I/O
- Network I/O

**Business Metrics**:
- Orders created
- Revenue generated
- User signups
- Failed payments

## Summary

Observability is essential for production APIs:

- **Logs**: Structured, searchable, with full context
- **Metrics**: Track performance and health
- **Traces**: Understand request flows
- **Health Checks**: Enable automated monitoring
- **Alerts**: Get notified of issues

Without observability, you're flying blind. With it, you can:
- Debug issues quickly
- Identify performance bottlenecks
- Prevent outages
- Understand user behavior
- Make data-driven decisions

Invest in observability from day one.

## Further Reading

- **The Twelve-Factor App**: Logs as event streams
- **Prometheus**: Metrics and alerting
- **OpenTelemetry**: Tracing standard
- **ELK Stack**: Elasticsearch, Logstash, Kibana
- **Datadog**: Application monitoring

---

**Congratulations!** You've completed all REST API best practices guides.

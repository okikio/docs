# HTTP API design

An HTTP API gives clients a way to request data or change system state. Its contract
includes inputs, authorization, response formats, failure behavior, and retry rules.
Use these guides to define that contract and trace a request from the client to
its observable result.

This section contains ten topic guides. Their examples illustrate decisions to
review for your own service; they are not a complete production implementation.

## What You'll Learn

Choose a guide for the behavior you need to define:

### Core Topics

1. **[Versioning](./versioning.md)** - How to evolve your API without breaking existing clients
2. **[Error Handling](./error-handling.md)** - Standardized approaches to communicating failures
3. **[Authentication](./authentication.md)** - Securing your API with modern authentication methods
4. **[Filtering](./filtering.md)** - Enabling clients to query exactly the data they need
5. **[Pagination](./pagination.md)** - Efficiently handling large datasets
6. **[Rate Limiting](./rate-limiting.md)** - Protecting your infrastructure from abuse
7. **[Content Negotiation](./content-negotiation.md)** - Supporting multiple data formats
8. **[Idempotency](./idempotency.md)** - Making operations safe to retry
9. **[Caching](./caching.md)** - Leveraging HTTP caching for performance
10. **[Observability](./observability.md)** - Logging, monitoring, and debugging in production

## REST Fundamentals

REST (Representational State Transfer) is an architectural style described in [Roy Fielding's dissertation](https://roy.gbiv.com/pubs/dissertation/rest_arch_style.htm). HTTP is a protocol often used to implement it. [RFC 9110](https://www.rfc-editor.org/info/rfc9110/) defines HTTP semantics, including methods, status codes, and representation metadata; it does not define REST.

### The Six Constraints

Roy Fielding's original dissertation defined REST with six architectural constraints:

1. **Client-Server** - Separation of concerns between UI and data storage
2. **Stateless** - Each request contains all information needed to understand it
3. **Cacheable** - Responses must define themselves as cacheable or not
4. **Uniform Interface** - Consistent way to interact with resources
5. **Layered System** - Client can't tell if connected directly to server
6. **Code on Demand** (optional) - Servers can extend client functionality

## Getting Started

If you're new to API design, we recommend starting with these guides in order:

1. Start with **[Error Handling](./error-handling.md)** to understand how to communicate problems
2. Move to **[Authentication](./authentication.md)** to learn about securing your API
3. Then **[Versioning](./versioning.md)** to plan for evolution
4. Finally **[Caching](./caching.md)** to make it fast

For specific challenges:
- **Building a search API?** → Read [Filtering](./filtering.md) and [Pagination](./pagination.md)
- **Payment or transaction API?** → Focus on [Idempotency](./idempotency.md)
- **Public API with many users?** → Start with [Rate Limiting](./rate-limiting.md)
- **Need to debug issues?** → Jump to [Observability](./observability.md)

## Standards and Specifications

These guides reference and build upon established web standards:

- **HTTP semantics** — [RFC 9110](https://www.rfc-editor.org/info/rfc9110/)
- **HTTP caching** — [RFC 9111](https://www.rfc-editor.org/info/rfc9111/)
- **HTTP/1.1 message syntax** — [RFC 9112](https://www.rfc-editor.org/info/rfc9112/)
- **HTTP/2** — [RFC 9113](https://www.rfc-editor.org/info/rfc9113/)
- **HTTP/3** - RFC 9114 (QUIC-based HTTP)
- **URI** - RFC 3986 (Uniform Resource Identifiers)
- **JSON** - RFC 8259 (JavaScript Object Notation)
- **OAuth 2.0** - RFC 6749 (Authorization framework)
- **JWT** - RFC 7519 (JSON Web Tokens)
- **Problem Details** — [RFC 9457](https://www.rfc-editor.org/info/rfc9457/), which replaces RFC 7807

## Beyond This Guide

API design is a vast topic. After mastering these fundamentals, consider exploring:

- **GraphQL** for query flexibility
- **gRPC** for high-performance RPC
- **WebSockets** for real-time bidirectional communication
- **Server-Sent Events** for server-push updates
- **Webhooks** for event-driven integrations

Choose a protocol from the interaction your clients need, then document its connection, authorization, retry, and failure behavior.

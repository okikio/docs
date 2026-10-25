# Software design guides

Use these guides to make and explain decisions about HTTP APIs, browser applications,
and reusable JavaScript packages. Each guide starts with a problem, describes the
relevant contract, and gives examples or questions to check against your own system.

## Find a guide

| Task | Start here | What is available |
| --- | --- | --- |
| Design an HTTP API | [API guide](apis/README.md) | An overview and ten topic guides |
| Build a browser interface | [Frontend guide](frontend/README.md) | One overview of user flows, browser behavior, performance, and accessibility |
| Publish reusable code | [Library guide](libraries/README.md) | One overview of API contracts, package formats, documentation, and releases |

### API topics

- [Versioning](apis/versioning.md): change an API while supporting existing clients.
- [Error handling](apis/error-handling.md): explain what failed and what the client can do next.
- [Authentication](apis/authentication.md): establish identity and enforce access rules.
- [Filtering](apis/filtering.md) and [pagination](apis/pagination.md): select and traverse results.
- [Rate limiting](apis/rate-limiting.md): bound request volume and communicate retry conditions.
- [Content negotiation](apis/content-negotiation.md): select a response representation.
- [Idempotency](apis/idempotency.md): define what happens when a request is repeated.
- [Caching](apis/caching.md): reuse responses under explicit freshness and validation rules.
- [Observability](apis/observability.md): connect a failed request to diagnostic evidence.

## Apply a guide to a real change

1. State the user's goal and the result the system must provide.
2. Identify the current entrypoint, responsible component, stored state, and external dependencies.
3. Read the guide for the decision you need to make. Follow its primary-source links when a standard determines the behavior.
4. Record the chosen behavior, its limits, and the alternatives that matter for this system.
5. Verify one normal flow and the failures that could violate the contract. Keep the commands and results with the change.

These documents are design guidance. They do not certify an implementation or
replace its supported-version policy, security review, or tests. Frontend and
library topic pages beyond the overviews have not been added yet.

## Contribute

A useful addition lets another maintainer apply the idea without asking its author
to explain it again. Give the reader a concrete problem, define necessary terms,
and connect each recommendation to its reason. Distinguish a standard's requirement
from a project choice. Link to the relevant primary source and include prerequisites
and expected results for runnable examples.

Before submitting, follow the links, check the examples, and explain any checks
you could not run. Preserve exact API names, commands, and protocol fields when
simplifying the surrounding English.

## License

MIT

# Design and publish a reusable library

A library exposes behavior that other projects depend on. Its users need to know
which inputs it accepts, what it returns, what can fail, and which environments it
supports. Maintainers need to understand how those guarantees connect to the
implementation and release process.

This page is an overview. Separate package-architecture, API-design, and publishing
guides have not been added to this repository yet.

## Define the public contract

Start with one concrete use case and a complete example. Name the operation from
the caller's task. Document required input, defaults, returned values, errors,
side effects, and resource ownership. Use the same term for the same concept in
types, examples, error messages, and release notes.

For an asynchronous operation, distinguish requesting cancellation from stopping
the underlying work and releasing resources. State whether an input is reusable
or consumed once. A caller should not have to infer these rules from implementation
details.

Keep implementation helpers private unless a consumer needs a supported contract
for them. Test documented imports as a consumer would use them.

## Choose package formats from supported consumers

Do not add output formats only because another package publishes them. Record the
runtimes, versions, and module systems you intend to support, then test the built
package in those environments.

| Consumer requirement | Packaging decision to evaluate |
| --- | --- |
| ECMAScript module imports | ESM entrypoints and the corresponding type declarations |
| CommonJS `require()` | A compatible CommonJS entrypoint, if this consumer is supported |
| A browser script without a module loader | A separate browser distribution, if this is a supported use case |

For Node.js, [`package.json` and file extensions determine module interpretation](https://nodejs.org/api/packages.html).
The `exports` field defines public entrypoints. A dual ESM/CommonJS package needs
consumer checks for both paths; adding a second build also adds maintenance and
compatibility work. Use the project's actual package requirements to choose.

For example, this export map describes an ESM entrypoint and its declarations.
It is a packaging example, not a complete publishable package:

```json
{
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    }
  }
}
```

Confirm that every declared file exists in the packed artifact. Source-checkout
tests alone cannot prove that consumers can install and import the release.

## Make documentation usable

Give the reader installation instructions, one complete normal example, expected
output, and the failure or cleanup behavior that affects that example. Declare
its runtime and prerequisites. Keep examples close to tests when that lets you
check that they still run.

Types help callers discover inputs and outputs, but they do not explain the whole
runtime contract. Document defaults, timing, mutation, limits, and ownership where
the type system cannot express them. Test declarations against supported consumers;
handwritten or generated declarations can drift from runtime behavior.

## Keep dependencies and size measurable

For each dependency, record the behavior it provides and why it belongs at that
layer. Consider its supported runtimes, license, release history, and maintenance
cost. A dependency can reduce risk when it replaces code the project would otherwise
have to maintain; it still needs a clear owner.

Set a size budget from the intended consumer. Measure the installed entrypoint
with representative imports and the same build and compression settings over time.
Keep optional features separate where that meaningfully reduces what consumers
load. A universal size limit such as 10 KB is not useful for every kind of library.

## Verify the promises you publish

Use focused tests for functions and state transitions, integration tests for
component boundaries, and consumer tests for installation and public imports.
Select tests from the contract and failure risks rather than a fixed ratio of
unit to end-to-end tests.

A release review should answer these questions:

- Does the packed artifact contain every public entrypoint and required asset?
- Can supported consumers import it and run a representative documented flow?
- Do errors, cancellation, and cleanup match the documented behavior?
- Are changes compatible with existing callers, or is migration guidance required?
- Which checks ran, on which version and environment, and which remain unrun?

## Version and release changes deliberately

[Semantic Versioning](https://semver.org/) relates versions to the declared public
API. After 1.0.0, incompatible API changes require a major release, compatible
functionality a minor release, and compatible fixes a patch release. Versions
before 1.0.0 have different stability expectations; state the project's policy.

Write release notes around caller-visible changes. Explain what changed, who is
affected, and how to migrate. Preserve reasons and limitations. Generated lists
of changed packages are useful, but they do not replace an explanation of a
shared change.

Build and test the artifact before publishing it. Record the source revision and
artifact used for the release. Keep publication, tagging, and deployment steps
explicit so a successful test is not mistaken for a successful publication.

## Set maintenance expectations

Document the license, supported versions, reporting process, contribution checks,
and maintenance status. Mark experimental APIs explicitly. Choose the release
schedule from readiness and support capacity, rather than promising a fixed number
of weeks from prototype to 1.0.

[Return to the guide index](../README.md)

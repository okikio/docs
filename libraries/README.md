# Library & Package Development Best Practices

> **Creating reusable code that developers love to use**

Libraries are the building blocks of modern software. When you publish a library, you're not just sharing code—you're making a promise to other developers. A promise that your library will work, that it will be documented, that it will be maintained, and that it won't break their applications.

Great libraries get adopted widely and stand the test of time. Poor libraries frustrate developers and get abandoned. The difference comes down to thoughtful design, clear documentation, and attention to the developer experience.

This section covers everything you need to create production-ready libraries that developers will trust and recommend.

## What You'll Learn

Each guide takes you from basic concepts to professional-grade implementation:

### Core Topics

1. **Package Architecture** - Structuring your library for maintainability
2. **API Design** - Creating intuitive, flexible interfaces
3. **Documentation** - Writing guides that actually help
4. **TypeScript & Types** - Providing excellent type safety
5. **Testing** - Ensuring reliability and correctness
6. **Versioning** - Managing changes without breaking users
7. **Performance** - Keeping your library fast and lightweight
8. **Bundle Size** - Minimizing the impact on applications
9. **Backwards Compatibility** - Evolving without breaking
10. **Security** - Protecting users from vulnerabilities
11. **Developer Experience** - Making your library pleasant to use
12. **Publishing** - Distributing your library effectively

## The Library Mindset

Building a library is different from building an application. The mindset shift is crucial:

### Applications vs. Libraries

**Applications**:
- You control the entire environment
- You know the use cases
- You can change anything anytime
- Users interact through UI

**Libraries**:
- You run in unknown environments
- You can't predict all use cases
- Breaking changes affect thousands of projects
- Developers interact through code

This difference impacts every decision you make.

### The Responsibility of Publishing

When you publish a library, you're taking on responsibilities:

**Stability**: Applications depend on your code not breaking
**Security**: Vulnerabilities in your library affect all users
**Performance**: Your code runs in production applications
**Support**: Developers will ask questions and report bugs
**Maintenance**: Updates, bug fixes, and compatibility

If you're not ready for these responsibilities, consider keeping your library private or clearly marking it as experimental.

## What Makes a Great Library

Great libraries share common characteristics:

### 1. Clear Purpose

**Good**: "A library for parsing and manipulating dates"
**Bad**: "A utility library with various helpful functions"

The best libraries do one thing well. If your library tries to do everything, it will excel at nothing.

### 2. Intuitive API

```javascript
// Good: Clear, predictable
const user = await db.users.findById(123);
await db.users.update(123, { name: 'Alice' });

// Bad: Confusing, inconsistent
const user = await db.getUserByTheId(123);
await db.updateUser({ userId: 123, data: { name: 'Alice' } });
```

APIs should feel natural. Developers should guess the right method name on the first try.

### 3. Excellent Documentation

Documentation is not optional. If developers can't figure out how to use your library in 5 minutes, they'll find an alternative.

### 4. Minimal Dependencies

Every dependency is a liability:
- Increases bundle size
- Introduces security vulnerabilities
- Creates compatibility issues
- Adds maintenance burden

Only add dependencies when they provide significant value.

### 5. Predictable Behavior

No surprises. No magic. No hidden side effects.

```javascript
// Good: Explicit, predictable
const result = transform(data, { sortBy: 'name' });

// Bad: Magic, unpredictable
const result = transform(data); // Sorts by... what? How do I know?
```

## The Module System Landscape

JavaScript has evolved through several module systems. Modern libraries need to support multiple formats.

### CommonJS (CJS)

The original Node.js module system:

```javascript
// Export
module.exports = { myFunction };

// Import
const { myFunction } = require('my-library');
```

**Still used by**: Older Node.js projects, some build tools
**File extension**: `.js` or `.cjs`

### ES Modules (ESM)

The standard JavaScript module system:

```javascript
// Export
export { myFunction };

// Import
import { myFunction } from 'my-library';
```

**Used by**: Modern applications, browsers, Node.js 12+
**File extension**: `.mjs` or `.js` (with `"type": "module"`)

### Universal Module Definition (UMD)

Works in browsers, Node.js, and AMD loaders:

```javascript
(function (root, factory) {
  if (typeof define === 'function' && define.amd) {
    define([], factory); // AMD
  } else if (typeof module === 'object' && module.exports) {
    module.exports = factory(); // CommonJS
  } else {
    root.MyLibrary = factory(); // Browser global
  }
}(typeof self !== 'undefined' ? self : this, function () {
  return { myFunction };
}));
```

**Used by**: Browser scripts via CDN, legacy projects
**File extension**: `.js`

### What Should You Publish?

**Recommendation**: Publish both ESM and CJS builds. Most modern libraries follow this pattern:

```
dist/
  index.js      # CommonJS
  index.mjs     # ES Modules
  index.d.ts    # TypeScript declarations
  index.umd.js  # UMD for browsers (optional)
```

Configure in `package.json`:

```json
{
  "main": "./dist/index.js",
  "module": "./dist/index.mjs",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.js"
    }
  }
}
```

## TypeScript: Not Just for Type Safety

Even if your library is written in JavaScript, you should provide TypeScript definitions. Why?

**Developer Experience**: IntelliSense, autocomplete, inline documentation
**Error Prevention**: Catch mistakes before runtime
**Self-Documentation**: Types serve as always-up-to-date API documentation
**Growing Adoption**: Over 60% of npm packages now use TypeScript

You don't need to write TypeScript to provide types. You can use JSDoc:

```javascript
/**
 * Fetches user data from the API
 * @param {string} userId - The user's unique identifier
 * @param {Object} options - Optional configuration
 * @param {boolean} [options.includeProfile=true] - Include profile data
 * @returns {Promise<User>} The user object
 */
export async function getUser(userId, options = {}) {
  // Implementation
}
```

## Semantic Versioning: The Contract

Semantic versioning (SemVer) is how you communicate the impact of changes:

```
MAJOR.MINOR.PATCH
  2  .  3  .  1
```

**MAJOR** (2): Breaking changes
- Removed or renamed functions
- Changed function signatures
- Changed return types
- Changed behavior in incompatible ways

**MINOR** (3): New features (backwards compatible)
- Added new functions
- Added optional parameters
- Added new functionality

**PATCH** (1): Bug fixes (backwards compatible)
- Fixed incorrect behavior
- Performance improvements
- Documentation updates

**Rule**: Never break backwards compatibility in MINOR or PATCH releases.

## The Testing Pyramid

Libraries need comprehensive testing:

```
         E2E Tests (Few)
            /\
           /  \
          /    \
         /      \
    Integration  \
       Tests     /\
      (Some)    /  \
               /    \
              /      \
             /________\
          Unit Tests (Many)
```

**Unit Tests**: Test individual functions in isolation
**Integration Tests**: Test how components work together
**End-to-End Tests**: Test in real application scenarios

For libraries, focus heavily on unit tests with good integration coverage.

## Bundle Size Matters

Your library's size directly impacts:
- Download time for users
- Parse/compile time in bundlers
- Application bundle size

**Guidelines**:
- Keep core library < 10KB (minified + gzipped)
- Make large features optional
- Use tree-shaking to eliminate unused code
- Avoid heavy dependencies

**Tools to measure**:
- bundlephobia.com - Shows package size
- size-limit - Fails CI if size grows too much

## Security: Your Responsibility

Security vulnerabilities in your library affect every application that uses it.

### Supply Chain Attacks

In 2021, the `colors` and `faker` packages were sabotaged by their own maintainer, breaking thousands of applications.

**Protections**:
- Use 2FA on npm account
- Audit dependencies regularly
- Use lock files
- Review all PRs carefully
- Have multiple maintainers

### Input Validation

Always validate and sanitize inputs:

```javascript
// Bad: Trusts input
export function execute(code) {
  eval(code); // Never do this!
}

// Good: Validates and limits
export function calculate(expression) {
  if (typeof expression !== 'string') {
    throw new TypeError('Expression must be a string');
  }
  
  if (expression.length > 100) {
    throw new Error('Expression too long');
  }
  
  // Use safe parser instead of eval
  return safelyParse(expression);
}
```

## Open Source Considerations

If you're publishing open source:

**Choose a License**:
- MIT: Permissive, most popular
- Apache 2.0: Includes patent grant
- GPL: Copyleft, requires derivatives to be open source

**Set Expectations**:
- Document how to contribute
- Define supported versions
- Set response time expectations
- Be clear about maintenance commitment

**Community Management**:
- Respond to issues promptly (even if just to acknowledge)
- Be respectful and professional
- Merge PRs from contributors
- Give credit where due

## The Path to Production

Creating a great library takes time:

**Phase 1: Prototype** (Weeks 1-2)
- Build core functionality
- Test with small projects
- Iterate on API design

**Phase 2: Polish** (Weeks 3-4)
- Add comprehensive tests
- Write documentation
- Set up build process
- Configure TypeScript

**Phase 3: Pre-release** (Weeks 5-6)
- Publish beta versions
- Get feedback from early users
- Fix bugs and improve API
- Finalize documentation

**Phase 4: Launch** (Week 7)
- Publish 1.0.0
- Write announcement post
- Submit to registries
- Monitor for issues

**Phase 5: Maintenance** (Ongoing)
- Fix bugs
- Add features
- Keep dependencies updated
- Support users

## Getting Started

Start with these guides in order:

1. **Package Architecture** - Set up your project structure
2. **API Design** - Design your public interface
3. **Testing** - Write tests before implementing features
4. **Documentation** - Document as you build
5. **TypeScript** - Add type definitions
6. **Publishing** - Get your library out there

Remember: It's better to start small and focused than to try building everything at once.

## Further Reading

- **npm docs** - Package publishing and management
- **SemVer** - Semantic versioning specification
- **TypeScript Handbook** - Type system guide
- **Jest Documentation** - Testing framework
- **Rollup** - Module bundler for libraries
- **API Design Guide** - Best practices for API design

---

**Next**: Learn how to structure your library in [Package Architecture](./package-architecture.md).

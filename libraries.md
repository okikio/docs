# Library Best Practices

A comprehensive guide to creating reusable, maintainable, and developer-friendly libraries and packages.

## Table of Contents

1. [Package Setup](#package-setup)
2. [API Design](#api-design)
3. [Documentation](#documentation)
4. [TypeScript Support](#typescript-support)
5. [Testing](#testing)
6. [Versioning & Releases](#versioning--releases)
7. [Performance](#performance)
8. [Bundle Size](#bundle-size)
9. [Backwards Compatibility](#backwards-compatibility)
10. [Security](#security)
11. [Developer Experience](#developer-experience)
12. [Publishing & Distribution](#publishing--distribution)

---

## Package Setup

Structure your library for maintainability and ease of use.

### Directory Structure

```
my-library/
├── src/                    # Source code
│   ├── index.ts           # Main entry point
│   ├── core/              # Core functionality
│   ├── utils/             # Utility functions
│   └── types/             # TypeScript types
├── dist/                  # Compiled output (gitignored)
│   ├── index.js           # CommonJS
│   ├── index.mjs          # ES Modules
│   ├── index.d.ts         # TypeScript declarations
│   └── index.umd.js       # UMD (browser)
├── tests/                 # Test files
├── docs/                  # Documentation
├── examples/              # Usage examples
├── .github/               # GitHub actions and templates
├── .gitignore
├── .npmignore
├── package.json
├── tsconfig.json
├── README.md
├── LICENSE
└── CHANGELOG.md
```

### package.json Configuration

```json
{
  "name": "@scope/my-library",
  "version": "1.0.0",
  "description": "A brief description of your library",
  "keywords": ["keyword1", "keyword2", "keyword3"],
  "author": "Your Name <email@example.com>",
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": "https://github.com/username/my-library.git"
  },
  "bugs": {
    "url": "https://github.com/username/my-library/issues"
  },
  "homepage": "https://github.com/username/my-library#readme",
  "main": "./dist/index.js",
  "module": "./dist/index.mjs",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.js"
    },
    "./package.json": "./package.json"
  },
  "files": [
    "dist",
    "README.md",
    "LICENSE"
  ],
  "scripts": {
    "build": "tsup src/index.ts --format cjs,esm --dts",
    "test": "jest",
    "lint": "eslint src --ext .ts",
    "prepublishOnly": "npm run build && npm test"
  },
  "peerDependencies": {
    "react": "^18.0.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "typescript": "^5.0.0",
    "tsup": "^7.0.0",
    "jest": "^29.0.0"
  },
  "engines": {
    "node": ">=16.0.0"
  }
}
```

### .npmignore

```
src/
tests/
examples/
.github/
*.test.ts
*.spec.ts
tsconfig.json
.eslintrc
.prettierrc
jest.config.js
```

### Best Practices

- Use scoped packages (@scope/package-name) for organization
- Provide multiple module formats (CJS, ESM, UMD)
- Include only necessary files in npm package
- Specify peer dependencies clearly
- Set minimum Node.js version
- Use semantic versioning
- Include LICENSE file
- Add comprehensive README

---

## API Design

Create intuitive and flexible APIs.

### Principle of Least Surprise

Design APIs that work as users expect:

```javascript
// Good: Intuitive naming
library.createUser({ name: 'John', email: 'john@example.com' });
library.deleteUser(userId);

// Bad: Confusing naming
library.makeUser({ name: 'John', email: 'john@example.com' });
library.removeUser(userId);
```

### Consistent Naming Conventions

```javascript
// Use consistent verb patterns
get(), set(), has(), is(), create(), update(), delete()

// Use consistent parameter order
function transform(data, options) { }
function format(value, options) { }
function validate(input, rules) { }
```

### Options Object Pattern

```javascript
// Good: Options object for flexibility
function createChart(data, options = {}) {
  const {
    width = 600,
    height = 400,
    type = 'bar',
    theme = 'light',
    ...rest
  } = options;
  
  // Implementation
}

createChart(data, { width: 800, type: 'line' });

// Bad: Too many parameters
function createChart(data, width, height, type, theme) { }
```

### Builder Pattern

```javascript
class QueryBuilder {
  constructor() {
    this.query = {};
  }
  
  where(field, value) {
    this.query.where = { ...this.query.where, [field]: value };
    return this;
  }
  
  orderBy(field, direction = 'asc') {
    this.query.orderBy = { field, direction };
    return this;
  }
  
  limit(count) {
    this.query.limit = count;
    return this;
  }
  
  build() {
    return this.query;
  }
}

// Usage
const query = new QueryBuilder()
  .where('status', 'active')
  .orderBy('createdAt', 'desc')
  .limit(10)
  .build();
```

### Plugin/Extension System

```javascript
class Library {
  constructor() {
    this.plugins = [];
  }
  
  use(plugin, options = {}) {
    if (typeof plugin === 'function') {
      plugin(this, options);
    } else if (plugin.install) {
      plugin.install(this, options);
    }
    this.plugins.push(plugin);
    return this;
  }
}

// Plugin
const loggerPlugin = {
  install(library, options) {
    library.log = (message) => {
      if (options.enabled) {
        console.log(`[${options.prefix}] ${message}`);
      }
    };
  }
};

// Usage
const lib = new Library();
lib.use(loggerPlugin, { enabled: true, prefix: 'MyLib' });
```

### Functional vs OOP APIs

**Functional Style**
```javascript
// Pure functions, composable
import { map, filter, reduce } from 'my-library';

const result = reduce(
  filter(
    map(data, x => x * 2),
    x => x > 10
  ),
  (acc, x) => acc + x,
  0
);
```

**Object-Oriented Style**
```javascript
// Stateful, method chaining
import { Collection } from 'my-library';

const result = new Collection(data)
  .map(x => x * 2)
  .filter(x => x > 10)
  .reduce((acc, x) => acc + x, 0);
```

### Error Handling

```javascript
// Custom error classes
class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = 'ValidationError';
    this.field = field;
  }
}

class NetworkError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.name = 'NetworkError';
    this.statusCode = statusCode;
  }
}

// Throw descriptive errors
function validateEmail(email) {
  if (!email) {
    throw new ValidationError('Email is required', 'email');
  }
  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
    throw new ValidationError('Invalid email format', 'email');
  }
}
```

### Best Practices

- Keep the API surface small and focused
- Make simple things simple, complex things possible
- Use method chaining where appropriate
- Provide sensible defaults
- Return consistent types
- Don't break backwards compatibility
- Support both promises and callbacks (if needed)
- Make the API tree-shakeable

---

## Documentation

Provide comprehensive and accessible documentation.

### README Structure

```markdown
# Library Name

Brief description of what the library does.

## Features

- Feature 1
- Feature 2
- Feature 3

## Installation

```bash
npm install my-library
# or
yarn add my-library
```

## Quick Start

```javascript
import { createInstance } from 'my-library';

const instance = createInstance({
  option1: 'value1',
  option2: 'value2'
});

instance.doSomething();
```

## API Reference

### createInstance(options)

Creates a new instance with the given options.

**Parameters:**
- `options` (Object) - Configuration options
  - `option1` (string) - Description of option1
  - `option2` (number) - Description of option2

**Returns:** Instance

**Example:**
```javascript
const instance = createInstance({ option1: 'value' });
```

## Advanced Usage

[More detailed examples]

## Browser Support

- Chrome (latest)
- Firefox (latest)
- Safari (latest)
- Edge (latest)

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md)

## License

MIT © [Your Name]
```

### JSDoc Comments

```typescript
/**
 * Fetches user data from the API
 * 
 * @param {string} userId - The unique identifier of the user
 * @param {Object} options - Optional configuration
 * @param {boolean} [options.includeProfile=true] - Include user profile data
 * @param {boolean} [options.includeStats=false] - Include user statistics
 * @returns {Promise<User>} A promise that resolves to the user object
 * @throws {NetworkError} If the network request fails
 * @throws {NotFoundError} If the user doesn't exist
 * 
 * @example
 * const user = await fetchUser('123', { includeStats: true });
 * console.log(user.name);
 */
export async function fetchUser(
  userId: string,
  options: FetchUserOptions = {}
): Promise<User> {
  // Implementation
}
```

### API Documentation Site

```
docs/
├── index.md              # Home page
├── getting-started.md    # Installation and quick start
├── api/
│   ├── core.md          # Core API reference
│   ├── utilities.md     # Utility functions
│   └── types.md         # Type definitions
├── guides/
│   ├── basic-usage.md
│   ├── advanced-usage.md
│   └── migration.md
└── examples/
    ├── example1.md
    └── example2.md
```

### Changelog

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- New feature X

### Changed
- Improved performance of Y

### Deprecated
- Method Z will be removed in v3.0.0

### Fixed
- Bug in feature A

## [2.1.0] - 2024-01-15

### Added
- Support for custom plugins
- New `validate()` method

### Fixed
- Memory leak in event listeners

## [2.0.0] - 2024-01-01

### Breaking Changes
- Removed deprecated `oldMethod()`
- Changed return type of `getData()`

### Added
- Complete TypeScript rewrite
- New configuration options

### Migration Guide
[Link to migration guide]
```

### Best Practices

- Write clear, concise documentation
- Provide runnable examples
- Document all public APIs
- Include migration guides for breaking changes
- Keep documentation in sync with code
- Use tools like TypeDoc or JSDoc for auto-generation
- Provide examples for common use cases
- Include troubleshooting section

---

## TypeScript Support

Provide excellent TypeScript experience.

### Type Definitions

```typescript
// src/types/index.ts

export interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user' | 'guest';
}

export interface CreateUserOptions {
  name: string;
  email: string;
  role?: User['role'];
  metadata?: Record<string, unknown>;
}

export type UserUpdatePayload = Partial<Omit<User, 'id'>>;

export interface LibraryConfig {
  apiKey: string;
  baseURL?: string;
  timeout?: number;
  retries?: number;
}

// Generic types
export interface ApiResponse<T> {
  data: T;
  status: number;
  message?: string;
}

export type AsyncResult<T, E = Error> = Promise<
  | { success: true; data: T }
  | { success: false; error: E }
>;
```

### Exported Types

```typescript
// src/index.ts

export { User, CreateUserOptions, UserUpdatePayload } from './types';

export class Library {
  constructor(config: LibraryConfig) {
    // Implementation
  }
  
  createUser(options: CreateUserOptions): Promise<User> {
    // Implementation
  }
  
  getUser(id: string): Promise<User | null> {
    // Implementation
  }
}

// Export everything
export * from './types';
export * from './utils';
```

### Generic Functions

```typescript
// Properly typed generic utilities
export function pick<T, K extends keyof T>(
  obj: T,
  keys: K[]
): Pick<T, K> {
  const result = {} as Pick<T, K>;
  keys.forEach(key => {
    result[key] = obj[key];
  });
  return result;
}

export function groupBy<T, K extends string | number>(
  array: T[],
  keyFn: (item: T) => K
): Record<K, T[]> {
  return array.reduce((acc, item) => {
    const key = keyFn(item);
    if (!acc[key]) acc[key] = [];
    acc[key].push(item);
    return acc;
  }, {} as Record<K, T[]>);
}
```

### Type Guards

```typescript
export function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'name' in value &&
    'email' in value
  );
}

export function isError(value: unknown): value is Error {
  return value instanceof Error;
}
```

### Declaration Files

```typescript
// dist/index.d.ts (auto-generated)
export interface User {
  id: string;
  name: string;
  email: string;
}

export declare class Library {
  constructor(config: LibraryConfig);
  createUser(options: CreateUserOptions): Promise<User>;
  getUser(id: string): Promise<User | null>;
}

export * from './types';
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "lib": ["ES2020"],
    "declaration": true,
    "declarationMap": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "isolatedModules": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
```

### Best Practices

- Export all public types
- Use strict TypeScript configuration
- Provide type definitions for all public APIs
- Use generics for reusable components
- Include type guards for runtime checks
- Generate declaration files automatically
- Test type definitions
- Support TypeScript 4.5+

---

## Testing

Ensure reliability and correctness.

### Unit Tests

```typescript
// tests/library.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { Library } from '../src';

describe('Library', () => {
  let library: Library;
  
  beforeEach(() => {
    library = new Library({ apiKey: 'test-key' });
  });
  
  describe('createUser', () => {
    it('creates a user with valid data', async () => {
      const user = await library.createUser({
        name: 'John Doe',
        email: 'john@example.com'
      });
      
      expect(user).toHaveProperty('id');
      expect(user.name).toBe('John Doe');
      expect(user.email).toBe('john@example.com');
    });
    
    it('throws error for invalid email', async () => {
      await expect(
        library.createUser({
          name: 'John',
          email: 'invalid-email'
        })
      ).rejects.toThrow('Invalid email format');
    });
  });
  
  describe('getUser', () => {
    it('returns user when found', async () => {
      const user = await library.getUser('123');
      expect(user).toBeTruthy();
    });
    
    it('returns null when not found', async () => {
      const user = await library.getUser('nonexistent');
      expect(user).toBeNull();
    });
  });
});
```

### Integration Tests

```typescript
// tests/integration/api.test.ts
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { setupTestServer } from './helpers/server';
import { Library } from '../../src';

describe('API Integration', () => {
  let server: TestServer;
  let library: Library;
  
  beforeAll(async () => {
    server = await setupTestServer();
    library = new Library({
      apiKey: 'test-key',
      baseURL: server.url
    });
  });
  
  afterAll(async () => {
    await server.close();
  });
  
  it('handles full user lifecycle', async () => {
    // Create
    const user = await library.createUser({
      name: 'Test User',
      email: 'test@example.com'
    });
    expect(user.id).toBeDefined();
    
    // Read
    const fetched = await library.getUser(user.id);
    expect(fetched).toEqual(user);
    
    // Update
    const updated = await library.updateUser(user.id, {
      name: 'Updated Name'
    });
    expect(updated.name).toBe('Updated Name');
    
    // Delete
    await library.deleteUser(user.id);
    const deleted = await library.getUser(user.id);
    expect(deleted).toBeNull();
  });
});
```

### Type Tests

```typescript
// tests/types.test.ts
import { expectType, expectError } from 'tsd';
import { Library, User, CreateUserOptions } from '../src';

// Test that types are correct
expectType<User>({
  id: '123',
  name: 'John',
  email: 'john@example.com',
  role: 'user'
});

// Test that invalid types are rejected
expectError<User>({
  id: '123',
  name: 'John'
  // Missing email
});

// Test generic functions
const lib = new Library({ apiKey: 'key' });
expectType<Promise<User>>(lib.createUser({ name: 'John', email: 'john@example.com' }));
```

### Test Coverage

```json
{
  "jest": {
    "collectCoverageFrom": [
      "src/**/*.ts",
      "!src/**/*.test.ts",
      "!src/**/*.spec.ts"
    ],
    "coverageThreshold": {
      "global": {
        "branches": 90,
        "functions": 90,
        "lines": 90,
        "statements": 90
      }
    }
  }
}
```

### Best Practices

- Aim for >90% code coverage
- Test public API thoroughly
- Test edge cases and error conditions
- Write integration tests for critical paths
- Test TypeScript types
- Use test fixtures for complex data
- Mock external dependencies
- Run tests in CI/CD pipeline
- Keep tests fast and isolated

---

## Versioning & Releases

Manage versions and releases effectively.

### Semantic Versioning

Follow [SemVer](https://semver.org/):

**MAJOR.MINOR.PATCH** (e.g., 2.3.1)

- **MAJOR**: Breaking changes
- **MINOR**: New features (backwards compatible)
- **PATCH**: Bug fixes (backwards compatible)

### Version Bumping

```bash
# Patch release (1.0.0 -> 1.0.1)
npm version patch

# Minor release (1.0.0 -> 1.1.0)
npm version minor

# Major release (1.0.0 -> 2.0.0)
npm version major

# Pre-release (1.0.0 -> 1.0.1-beta.0)
npm version prerelease --preid=beta
```

### Release Process

```bash
# 1. Update version
npm version minor

# 2. Update CHANGELOG.md
# [Manually edit changelog]

# 3. Commit changes
git add .
git commit -m "chore: release v2.1.0"

# 4. Create git tag
git tag v2.1.0

# 5. Push to remote
git push origin main --tags

# 6. Publish to npm
npm publish
```

### Automated Releases (GitHub Actions)

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          registry-url: 'https://registry.npmjs.org'
      
      - run: npm ci
      - run: npm test
      - run: npm run build
      
      - name: Publish to npm
        run: npm publish --access public
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
      
      - name: Create GitHub Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref }}
          release_name: Release ${{ github.ref }}
          draft: false
          prerelease: false
```

### Pre-releases

```bash
# Alpha release
npm version prerelease --preid=alpha
# 1.0.0 -> 1.0.1-alpha.0

# Beta release
npm version prerelease --preid=beta
# 1.0.1-alpha.0 -> 1.0.1-beta.0

# Release candidate
npm version prerelease --preid=rc
# 1.0.1-beta.0 -> 1.0.1-rc.0

# Publish pre-release
npm publish --tag beta
```

### Deprecation

```bash
# Deprecate a version
npm deprecate my-library@1.0.0 "Critical security vulnerability, use 1.0.1+"

# Deprecate all versions
npm deprecate my-library "Package no longer maintained"
```

### Best Practices

- Follow semantic versioning strictly
- Tag releases in git
- Update CHANGELOG.md for every release
- Automate releases with CI/CD
- Use pre-releases for testing
- Provide migration guides for breaking changes
- Never delete published versions
- Use `npm publish --dry-run` to test

---

## Performance

Optimize library performance.

### Lazy Loading

```javascript
// Lazy load heavy dependencies
export async function processImage(image) {
  const { default: sharp } = await import('sharp');
  return sharp(image).resize(800).toBuffer();
}

// Tree-shakeable exports
export { lightFeature } from './light';
export { heavyFeature } from './heavy'; // Only loaded if used
```

### Memoization

```javascript
// Memoize expensive computations
const cache = new Map();

export function expensiveOperation(input) {
  if (cache.has(input)) {
    return cache.get(input);
  }
  
  const result = /* expensive computation */;
  cache.set(input, result);
  return result;
}

// With size limit
class LRUCache {
  constructor(maxSize = 100) {
    this.cache = new Map();
    this.maxSize = maxSize;
  }
  
  get(key) {
    if (!this.cache.has(key)) return undefined;
    const value = this.cache.get(key);
    // Move to end (most recently used)
    this.cache.delete(key);
    this.cache.set(key, value);
    return value;
  }
  
  set(key, value) {
    if (this.cache.has(key)) {
      this.cache.delete(key);
    } else if (this.cache.size >= this.maxSize) {
      // Remove least recently used
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    this.cache.set(key, value);
  }
}
```

### Debouncing & Throttling

```javascript
// Debounce function
export function debounce(fn, delay) {
  let timeoutId;
  return function (...args) {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn.apply(this, args), delay);
  };
}

// Throttle function
export function throttle(fn, limit) {
  let inThrottle;
  return function (...args) {
    if (!inThrottle) {
      fn.apply(this, args);
      inThrottle = true;
      setTimeout(() => inThrottle = false, limit);
    }
  };
}
```

### Avoid Synchronous Operations

```javascript
// Bad: Synchronous file reading
import fs from 'fs';
const data = fs.readFileSync('large-file.txt', 'utf8');

// Good: Asynchronous
import fs from 'fs/promises';
const data = await fs.readFile('large-file.txt', 'utf8');
```

### Benchmarking

```javascript
// tests/benchmarks/performance.bench.ts
import { bench, describe } from 'vitest';
import { myFunction, optimizedFunction } from '../src';

describe('Performance comparison', () => {
  bench('original implementation', () => {
    myFunction(testData);
  });
  
  bench('optimized implementation', () => {
    optimizedFunction(testData);
  });
});
```

### Best Practices

- Profile before optimizing
- Lazy load heavy dependencies
- Implement caching where appropriate
- Avoid blocking operations
- Use Web Workers for CPU-intensive tasks
- Minimize object allocations in hot paths
- Use efficient data structures
- Benchmark performance changes

---

## Bundle Size

Keep your library lightweight.

### Analyze Bundle

```bash
# Using bundlephobia
npm install -g bundlephobia
bundlephobia my-library

# Using webpack-bundle-analyzer
npm install --save-dev webpack-bundle-analyzer
```

### Tree Shaking

```javascript
// package.json
{
  "sideEffects": false // Enable tree shaking
}

// Or specify side effects
{
  "sideEffects": [
    "*.css",
    "*.scss"
  ]
}

// Use ES modules
// src/index.ts
export { featureA } from './featureA';
export { featureB } from './featureB';
export { featureC } from './featureC';

// Users can import only what they need
import { featureA } from 'my-library'; // Only featureA is bundled
```

### Peer Dependencies

```json
{
  "peerDependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0"
  },
  "peerDependenciesMeta": {
    "react-dom": {
      "optional": true
    }
  }
}
```

### External Dependencies

```javascript
// rollup.config.js
export default {
  external: [
    'react',
    'react-dom',
    'lodash'
  ],
  output: {
    globals: {
      'react': 'React',
      'react-dom': 'ReactDOM'
    }
  }
};
```

### Bundle Size Badge

```markdown
[![npm bundle size](https://img.shields.io/bundlephobia/minzip/my-library)](https://bundlephobia.com/package/my-library)
```

### Size Limits

```json
// package.json
{
  "size-limit": [
    {
      "path": "dist/index.js",
      "limit": "10 KB"
    }
  ]
}
```

```yaml
# .github/workflows/size-limit.yml
name: Size Limit

on: [pull_request]

jobs:
  size:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: andresz1/size-limit-action@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

### Best Practices

- Keep bundle size < 10KB (minified + gzipped) when possible
- Use peer dependencies for common libraries
- Enable tree shaking
- Monitor bundle size in CI
- Use code splitting for large features
- Minimize dependencies
- Use lightweight alternatives when available

---

## Backwards Compatibility

Maintain compatibility across versions.

### Deprecation Warnings

```javascript
// Deprecate old methods gracefully
export function oldMethod() {
  console.warn(
    'oldMethod() is deprecated and will be removed in v3.0.0. ' +
    'Use newMethod() instead.'
  );
  return newMethod();
}

// Or with custom warning
let hasWarned = false;
export function deprecatedFeature() {
  if (!hasWarned) {
    console.warn('deprecatedFeature() is deprecated');
    hasWarned = true;
  }
  // Implementation
}
```

### Version Detection

```javascript
// Support multiple versions of dependencies
const reactVersion = React.version;

if (reactVersion.startsWith('16.')) {
  // Use React 16 API
} else if (reactVersion.startsWith('17.') || reactVersion.startsWith('18.')) {
  // Use React 17/18 API
}
```

### Feature Detection

```javascript
// Check for feature support
export function setupObserver(element, callback) {
  if ('IntersectionObserver' in window) {
    const observer = new IntersectionObserver(callback);
    observer.observe(element);
    return observer;
  } else {
    // Fallback implementation
    console.warn('IntersectionObserver not supported, using fallback');
    return setupFallbackObserver(element, callback);
  }
}
```

### Migration Guides

```markdown
# Migration Guide: v2 to v3

## Breaking Changes

### 1. Method Renaming

**Before (v2):**
```javascript
library.getData();
```

**After (v3):**
```javascript
library.fetchData();
```

### 2. Configuration Changes

**Before (v2):**
```javascript
new Library({ apiKey: 'key', debug: true });
```

**After (v3):**
```javascript
new Library({
  auth: { apiKey: 'key' },
  logging: { enabled: true }
});
```

### 3. Return Type Changes

**Before (v2):**
```javascript
const data = await library.fetch(); // Returns array
```

**After (v3):**
```javascript
const { data } = await library.fetch(); // Returns object with data property
```

## New Features

- Feature A
- Feature B

## Deprecations

- `oldMethod()` will be removed in v4.0.0
```

### Best Practices

- Never break backwards compatibility in minor/patch releases
- Provide deprecation warnings before removing features
- Maintain deprecated features for at least one major version
- Document all breaking changes
- Provide migration guides
- Use feature detection over version detection
- Consider polyfills for older environments

---

## Security

Protect users from vulnerabilities.

### Input Validation

```javascript
export function sanitizeInput(input) {
  if (typeof input !== 'string') {
    throw new TypeError('Input must be a string');
  }
  
  // Remove potentially dangerous characters
  return input
    .replace(/[<>]/g, '')
    .trim()
    .slice(0, 1000); // Limit length
}

export function validateUrl(url) {
  try {
    const parsed = new URL(url);
    // Only allow http and https
    if (!['http:', 'https:'].includes(parsed.protocol)) {
      throw new Error('Invalid protocol');
    }
    return parsed.href;
  } catch {
    throw new Error('Invalid URL');
  }
}
```

### Avoid eval()

```javascript
// Bad: Using eval
const result = eval(userInput); // Never do this!

// Good: Use JSON.parse for data
const data = JSON.parse(userInput);

// Good: Use Function constructor if absolutely necessary
const fn = new Function('x', 'return x * 2');
```

### Dependency Audits

```bash
# Check for vulnerabilities
npm audit

# Fix automatically
npm audit fix

# Update dependencies
npm update

# Check for outdated packages
npm outdated
```

### Security Workflow

```yaml
# .github/workflows/security.yml
name: Security Audit

on:
  schedule:
    - cron: '0 0 * * 0' # Weekly
  pull_request:

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      
      - name: Run npm audit
        run: npm audit --audit-level=moderate
      
      - name: Check for outdated dependencies
        run: npm outdated
```

### Secrets Management

```javascript
// Bad: Hardcoded secrets
const API_KEY = 'sk_live_abc123'; // Never do this!

// Good: Use environment variables
const API_KEY = process.env.API_KEY;

if (!API_KEY) {
  throw new Error('API_KEY environment variable is required');
}
```

### Rate Limiting

```javascript
class RateLimiter {
  constructor(maxRequests, windowMs) {
    this.maxRequests = maxRequests;
    this.windowMs = windowMs;
    this.requests = new Map();
  }
  
  checkLimit(key) {
    const now = Date.now();
    const userRequests = this.requests.get(key) || [];
    
    // Remove old requests
    const validRequests = userRequests.filter(
      timestamp => now - timestamp < this.windowMs
    );
    
    if (validRequests.length >= this.maxRequests) {
      throw new Error('Rate limit exceeded');
    }
    
    validRequests.push(now);
    this.requests.set(key, validRequests);
  }
}
```

### Best Practices

- Validate and sanitize all inputs
- Keep dependencies updated
- Run security audits regularly
- Never expose secrets in code
- Use HTTPS for all network requests
- Implement rate limiting
- Follow principle of least privilege
- Report security issues responsibly
- Have a security policy (SECURITY.md)

---

## Developer Experience

Make your library enjoyable to use.

### TypeScript Autocompletion

```typescript
// Provide excellent IDE support
export interface Config {
  /** API key for authentication */
  apiKey: string;
  
  /** Base URL for API requests (default: https://api.example.com) */
  baseURL?: string;
  
  /** Request timeout in milliseconds (default: 30000) */
  timeout?: number;
}

export class Library {
  /**
   * Creates a new Library instance
   * @example
   * const lib = new Library({ apiKey: 'your-key' });
   */
  constructor(config: Config) {
    // Implementation
  }
}
```

### Error Messages

```javascript
// Helpful error messages
class ValidationError extends Error {
  constructor(field, value, expected) {
    super(
      `Invalid value for "${field}": received ${JSON.stringify(value)}, ` +
      `expected ${expected}.\n\n` +
      `See documentation: https://docs.example.com/errors#validation`
    );
    this.name = 'ValidationError';
    this.field = field;
  }
}
```

### Debug Mode

```javascript
export class Library {
  constructor(config) {
    this.debug = config.debug || false;
    this.logger = config.logger || console;
  }
  
  log(message, ...args) {
    if (this.debug) {
      this.logger.log(`[Library] ${message}`, ...args);
    }
  }
  
  async fetchData() {
    this.log('Fetching data...');
    const data = await fetch(this.url);
    this.log('Data received:', data);
    return data;
  }
}
```

### Helpful Warnings

```javascript
export function createInstance(config) {
  if (!config.apiKey) {
    console.warn(
      '[Library] No API key provided. Some features may not work.\n' +
      'Get your API key at: https://example.com/api-keys'
    );
  }
  
  if (config.deprecated Option) {
    console.warn(
      '[Library] "deprecatedOption" is deprecated and will be removed in v3.0.0.\n' +
      'Use "newOption" instead.'
    );
  }
  
  return new Library(config);
}
```

### Code Examples

```javascript
/**
 * Fetches user data
 * 
 * @example
 * Basic usage:
 * ```javascript
 * const user = await library.getUser('123');
 * console.log(user.name);
 * ```
 * 
 * @example
 * With options:
 * ```javascript
 * const user = await library.getUser('123', {
 *   includeProfile: true,
 *   includeStats: true
 * });
 * ```
 */
export async function getUser(id, options) {
  // Implementation
}
```

### Playground/REPL

Provide an online playground:
- CodeSandbox template
- StackBlitz example
- Runkit notebook

### Best Practices

- Provide TypeScript definitions
- Write clear error messages
- Include debug mode
- Provide helpful warnings
- Document with examples
- Create online playgrounds
- Respond to issues promptly
- Accept community contributions

---

## Publishing & Distribution

Make your library accessible.

### npm Publishing

```bash
# Login to npm
npm login

# Publish public package
npm publish --access public

# Publish scoped package
npm publish --access public

# Publish with tag
npm publish --tag beta
```

### Multiple Registries

```json
// .npmrc
registry=https://registry.npmjs.org/
@myorg:registry=https://npm.pkg.github.com/
```

### Package Preparation

```json
{
  "scripts": {
    "prepublishOnly": "npm run lint && npm test && npm run build",
    "prepack": "npm run build"
  }
}
```

### CDN Distribution

```html
<!-- unpkg -->
<script src="https://unpkg.com/my-library@1.0.0/dist/index.umd.js"></script>

<!-- jsDelivr -->
<script src="https://cdn.jsdelivr.net/npm/my-library@1.0.0/dist/index.umd.js"></script>
```

### GitHub Releases

Create releases on GitHub:
- Tag with version number
- Include changelog
- Attach build artifacts
- Mark breaking changes

### Badges

```markdown
[![npm version](https://img.shields.io/npm/v/my-library.svg)](https://www.npmjs.com/package/my-library)
[![npm downloads](https://img.shields.io/npm/dm/my-library.svg)](https://www.npmjs.com/package/my-library)
[![bundle size](https://img.shields.io/bundlephobia/minzip/my-library)](https://bundlephobia.com/package/my-library)
[![license](https://img.shields.io/npm/l/my-library.svg)](https://github.com/username/my-library/blob/main/LICENSE)
[![build status](https://img.shields.io/github/workflow/status/username/my-library/CI)](https://github.com/username/my-library/actions)
```

### Best Practices

- Test package before publishing (`npm pack`)
- Use `.npmignore` to exclude unnecessary files
- Publish to npm registry for discoverability
- Create GitHub releases for each version
- Make package available via CDN
- Add badges to README
- Set up automated publishing
- Monitor download statistics

---

## Summary

Following these library best practices will help you create high-quality, maintainable packages:

1. **Structure your package** with clear organization and configuration
2. **Design intuitive APIs** that are easy to learn and use
3. **Document thoroughly** with examples and API references
4. **Support TypeScript** for excellent developer experience
5. **Test comprehensively** with unit, integration, and type tests
6. **Version semantically** and automate releases
7. **Optimize performance** through lazy loading and caching
8. **Minimize bundle size** with tree shaking and peer dependencies
9. **Maintain compatibility** with deprecation warnings and migration guides
10. **Secure your library** against common vulnerabilities
11. **Enhance developer experience** with helpful errors and debugging
12. **Distribute widely** via npm, CDN, and GitHub

These standards create a foundation for libraries that are reliable, performant, and a joy to use.

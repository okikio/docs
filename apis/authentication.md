# Authentication: Securing Your API

> **If you don't know who's making the request, how can you trust what they're asking for?**

It's 3 AM. Your monitoring dashboard lights up red. Someone just accessed your production database through your API and deleted 10,000 customer records. Your API had no authentication—anyone could call any endpoint. The damage is catastrophic, and it's entirely preventable.

Authentication isn't a feature you add later. It's the foundation of API security. Without it, you're leaving your data, your users, and your business completely exposed to anyone with internet access.

## The Stakes: Why Authentication Matters

Here's what's at risk without proper authentication:

**Data Breaches**:
- Sensitive user data exposed to attackers
- Regulatory violations (GDPR, HIPAA, PCI-DSS)
- Massive fines and legal liability

**Service Abuse**:
- Attackers consuming your resources
- Denial of service from malicious traffic
- Infrastructure costs skyrocketing

**Reputation Damage**:
- Loss of customer trust
- Negative press coverage
- Competitive disadvantage

**Business Impact**:
- Direct financial losses
- Customer churn
- Potential business closure

Authentication is your first line of defense. Get it right, and you sleep soundly. Get it wrong, and you're one breach away from disaster.

## Authentication vs. Authorization

Before we dive in, let's clarify a critical distinction that trips up many developers:

**Authentication**: "Who are you?"
- Proves the identity of the caller
- Answers: "Is this request from Alice or Bob?"
- Example: Login with username and password

**Authorization**: "What are you allowed to do?"
- Determines permissions for authenticated users
- Answers: "Can Alice delete this resource?"
- Example: Role-based access control

This guide focuses on **authentication**—proving identity. Authorization is covered in a separate guide.

## The Authentication Spectrum

There are multiple ways to authenticate API requests, each with different trade-offs. Let's explore them from simplest to most sophisticated.

### 1. API Keys: The Starting Point

**The Approach**: A long random string that identifies the caller.

```javascript
GET /api/users
Authorization: ApiKey sk_live_abc123xyz789
```

Or sometimes in a custom header:
```javascript
GET /api/users
X-API-Key: sk_live_abc123xyz789
```

**How It Works**:
1. User creates an account on your platform
2. You generate a unique API key for them
3. They include this key in every request
4. You validate the key and identify the user

**Real Example**:
```javascript
// Server-side validation
app.use(async (req, res, next) => {
  const apiKey = req.headers['x-api-key'];
  
  if (!apiKey) {
    return res.status(401).json({
      error: {
        type: 'authentication_required',
        message: 'API key is required'
      }
    });
  }
  
  // Look up API key in database
  const user = await db.users.findOne({ apiKey });
  
  if (!user) {
    return res.status(401).json({
      error: {
        type: 'invalid_credentials',
        message: 'Invalid API key'
      }
    });
  }
  
  // Attach user to request
  req.user = user;
  next();
});
```

**When to Use**:
- Server-to-server communication
- Internal microservices
- Webhook callbacks
- Simple APIs with trusted clients

**Strengths**:
- Simple to implement and understand
- No expiration (until revoked)
- Easy to regenerate if compromised
- Works well for automation and scripts

**Weaknesses**:
- Can't revoke individual sessions (key is long-lived)
- No built-in expiration
- If leaked, valid until manually revoked
- Not suitable for user-facing applications
- Difficult to implement fine-grained permissions

**Security Best Practices**:
```javascript
// Generate cryptographically secure API keys
const crypto = require('crypto');

function generateApiKey() {
  // 32 bytes = 256 bits of entropy
  const key = crypto.randomBytes(32).toString('base64url');
  return `sk_live_${key}`;
}

// Hash API keys before storing
const bcrypt = require('bcrypt');

async function storeApiKey(userId, apiKey) {
  const hashedKey = await bcrypt.hash(apiKey, 10);
  
  await db.apiKeys.create({
    userId,
    keyHash: hashedKey,
    prefix: apiKey.substring(0, 12), // Store prefix for identification
    createdAt: new Date()
  });
}

// Validate by comparing hashes
async function validateApiKey(providedKey) {
  const prefix = providedKey.substring(0, 12);
  
  // Find key by prefix for faster lookup
  const keys = await db.apiKeys.find({ prefix });
  
  for (const key of keys) {
    const isValid = await bcrypt.compare(providedKey, key.keyHash);
    if (isValid) {
      return await db.users.findById(key.userId);
    }
  }
  
  return null;
}
```

**Standards Reference**: While there's no formal RFC for API keys, the pattern follows HTTP authentication framework (RFC 7235).

### 2. Basic Authentication: The HTTP Standard

**The Approach**: Send username and password encoded in Base64 with every request.

```javascript
GET /api/users
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
```

The string after "Basic" is `base64(username:password)`. For example:
```javascript
const credentials = Buffer.from('alice:secret123').toString('base64');
// Results in: YWxpY2U6c2VjcmV0MTIz

fetch('/api/users', {
  headers: {
    'Authorization': `Basic ${credentials}`
  }
});
```

**How It Works**:
1. Client encodes `username:password` in Base64
2. Sends encoded string in Authorization header
3. Server decodes and validates credentials
4. Server returns response

**Server-Side Validation**:
```javascript
app.use((req, res, next) => {
  const authHeader = req.headers.authorization;
  
  if (!authHeader || !authHeader.startsWith('Basic ')) {
    res.setHeader('WWW-Authenticate', 'Basic realm="API"');
    return res.status(401).json({
      error: {
        type: 'authentication_required',
        message: 'Basic authentication required'
      }
    });
  }
  
  // Extract and decode credentials
  const base64Credentials = authHeader.split(' ')[1];
  const credentials = Buffer.from(base64Credentials, 'base64').toString('utf-8');
  const [username, password] = credentials.split(':');
  
  // Validate credentials
  const user = await validateUser(username, password);
  
  if (!user) {
    return res.status(401).json({
      error: {
        type: 'invalid_credentials',
        message: 'Invalid username or password'
      }
    });
  }
  
  req.user = user;
  next();
});
```

**When to Use**:
- Quick prototypes and development
- Internal tools
- Simple webhook endpoints
- When combined with HTTPS

**Strengths**:
- Built into HTTP specification (RFC 7617)
- Supported by all HTTP clients natively
- Simple to implement

**Weaknesses**:
- Sends credentials with every request
- Base64 is encoding, not encryption (easily reversible)
- **Must use HTTPS** or credentials are sent in plain text
- No built-in session management
- Can't revoke individual sessions
- Browser may cache credentials

**Critical Security Warning**:
```javascript
// ❌ NEVER do this without HTTPS
GET http://api.example.com/users
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=

// ✅ ALWAYS use HTTPS
GET https://api.example.com/users
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
```

**Standards Reference**: RFC 7617 - The 'Basic' HTTP Authentication Scheme

### 3. Bearer Tokens (JWT): Modern Stateless Authentication

**The Approach**: Issue a cryptographically signed token that contains user information.

```javascript
GET /api/users
Authorization: ******
```

**How It Works**:
1. User logs in with credentials
2. Server generates a JWT containing user info
3. Server signs the JWT with a secret key
4. Client stores the JWT
5. Client sends JWT with each request
6. Server validates signature and extracts user info

**JWT Structure**:
A JWT has three parts separated by dots:
```
header.payload.signature
```

```javascript
// Header (algorithm and token type)
{
  "alg": "HS256",
  "typ": "JWT"
}

// Payload (user data and metadata)
{
  "sub": "user_123",           // Subject (user ID)
  "email": "alice@example.com",
  "role": "admin",
  "iat": 1704724800,           // Issued at (timestamp)
  "exp": 1704728400            // Expires at (timestamp)
}

// Signature (proves token wasn't tampered with)
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret
)
```

**Implementation Example**:
```javascript
const jwt = require('jsonwebtoken');

// Login endpoint - issues JWT
app.post('/api/auth/login', async (req, res) => {
  const { email, password } = req.body;
  
  // Validate credentials
  const user = await db.users.findOne({ email });
  
  if (!user || !await bcrypt.compare(password, user.passwordHash)) {
    return res.status(401).json({
      error: {
        type: 'invalid_credentials',
        message: 'Invalid email or password'
      }
    });
  }
  
  // Generate JWT
  const token = jwt.sign(
    {
      sub: user.id,
      email: user.email,
      role: user.role
    },
    process.env.JWT_SECRET,
    {
      expiresIn: '1h',           // Token expires in 1 hour
      issuer: 'api.example.com',
      audience: 'api.example.com'
    }
  );
  
  res.json({
    access_token: token,
    token_type: '******,
    expires_in: 3600
  });
});

// Middleware to validate JWT
function authenticateToken(req, res, next) {
  const authHeader = req.headers.authorization;
  const token = authHeader && authHeader.split(' ')[1]; // Bearer TOKEN
  
  if (!token) {
    return res.status(401).json({
      error: {
        type: 'authentication_required',
        message: 'Access token is required'
      }
    });
  }
  
  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET, {
      issuer: 'api.example.com',
      audience: 'api.example.com'
    });
    
    req.user = payload;
    next();
  } catch (err) {
    if (err.name === 'TokenExpiredError') {
      return res.status(401).json({
        error: {
          type: 'token_expired',
          message: 'Access token has expired',
          expired_at: err.expiredAt
        }
      });
    }
    
    return res.status(401).json({
      error: {
        type: 'invalid_token',
        message: 'Invalid access token'
      }
    });
  }
}

// Protected endpoint
app.get('/api/users/me', authenticateToken, (req, res) => {
  res.json({
    id: req.user.sub,
    email: req.user.email,
    role: req.user.role
  });
});
```

**When to Use**:
- Modern web and mobile applications
- Microservices architecture
- Single-page applications (SPAs)
- APIs that need to scale horizontally

**Strengths**:
- Stateless (server doesn't need to store sessions)
- Self-contained (includes user information)
- Scales horizontally easily
- Can set expiration times
- Works across domains
- Industry standard (RFC 7519)

**Weaknesses**:
- Can't revoke individual tokens before expiration
- Token size larger than simple keys
- If secret is compromised, all tokens are invalid
- Payload is visible (Base64 encoded, not encrypted)

**Security Best Practices**:
```javascript
// Use strong secrets
const JWT_SECRET = crypto.randomBytes(64).toString('hex');

// Short expiration times
const token = jwt.sign(payload, secret, {
  expiresIn: '15m'  // 15 minutes, not hours or days
});

// Rotate secrets periodically
// Validate all claims
jwt.verify(token, secret, {
  issuer: 'expected-issuer',
  audience: 'expected-audience',
  algorithms: ['HS256']  // Prevent algorithm confusion attacks
});

// Don't store sensitive data in payload
// ❌ Bad
{ "sub": "123", "creditCard": "4111-1111-1111-1111" }

// ✅ Good
{ "sub": "123", "email": "user@example.com" }
```

**Token Refresh Pattern**:
```javascript
// Issue both access and refresh tokens
app.post('/api/auth/login', async (req, res) => {
  const user = await validateUser(req.body);
  
  // Short-lived access token
  const accessToken = jwt.sign(
    { sub: user.id, email: user.email },
    process.env.JWT_SECRET,
    { expiresIn: '15m' }
  );
  
  // Long-lived refresh token
  const refreshToken = jwt.sign(
    { sub: user.id, type: 'refresh' },
    process.env.REFRESH_SECRET,
    { expiresIn: '7d' }
  );
  
  // Store refresh token hash in database
  await db.refreshTokens.create({
    userId: user.id,
    tokenHash: await bcrypt.hash(refreshToken, 10),
    expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
  });
  
  res.json({
    access_token: accessToken,
    refresh_token: refreshToken,
    expires_in: 900
  });
});

// Refresh endpoint
app.post('/api/auth/refresh', async (req, res) => {
  const { refresh_token } = req.body;
  
  try {
    const payload = jwt.verify(refresh_token, process.env.REFRESH_SECRET);
    
    if (payload.type !== 'refresh') {
      throw new Error('Invalid token type');
    }
    
    // Verify refresh token is in database
    const tokens = await db.refreshTokens.find({ userId: payload.sub });
    
    let isValid = false;
    for (const token of tokens) {
      if (await bcrypt.compare(refresh_token, token.tokenHash)) {
        isValid = true;
        break;
      }
    }
    
    if (!isValid) {
      throw new Error('Token revoked');
    }
    
    // Issue new access token
    const newAccessToken = jwt.sign(
      { sub: payload.sub, email: payload.email },
      process.env.JWT_SECRET,
      { expiresIn: '15m' }
    );
    
    res.json({
      access_token: newAccessToken,
      expires_in: 900
    });
  } catch (err) {
    return res.status(401).json({
      error: {
        type: 'invalid_refresh_token',
        message: 'Invalid or expired refresh token'
      }
    });
  }
});
```

**Standards Reference**: RFC 7519 - JSON Web Token (JWT)

### 4. OAuth 2.0: Delegated Authorization

**The Approach**: Let users authorize your application to access their data on another service without sharing passwords.

**The Scenario**: You're building an app that needs to access a user's Google Calendar. Instead of asking for their Google password (which they should never share), you use OAuth 2.0.

**How It Works** (Authorization Code Flow):

```
1. User clicks "Sign in with Google"
2. Your app redirects to Google's authorization server
3. User logs in to Google and approves your app
4. Google redirects back to your app with an authorization code
5. Your app exchanges the code for an access token
6. Your app uses the access token to call Google APIs
```

**Visual Flow**:
```
┌─────────┐                                        ┌──────────┐
│  User   │                                        │  Google  │
│ (Alice) │                                        │  (OAuth  │
└─────────┘                                        │ Provider)│
     │                                             └──────────┘
     │  1. Click "Sign in with Google"                  │
     │─────────────────────────────────────────────────>│
     │                                                   │
     │  2. Redirected to Google login                   │
     │<─────────────────────────────────────────────────│
     │                                                   │
     │  3. Logs in and approves                         │
     │─────────────────────────────────────────────────>│
     │                                                   │
     │  4. Redirected back with auth code               │
     │<─────────────────────────────────────────────────│
     │                                                   │
┌─────────────┐                                         │
│  Your App   │                                         │
└─────────────┘                                         │
     │  5. Exchange code for access token               │
     │─────────────────────────────────────────────────>│
     │                                                   │
     │  6. Receive access token                         │
     │<─────────────────────────────────────────────────│
     │                                                   │
     │  7. Call Google APIs with token                  │
     │─────────────────────────────────────────────────>│
```

**Implementation Example** (as OAuth client):
```javascript
const express = require('express');
const axios = require('axios');

// Step 1: Redirect to authorization server
app.get('/auth/google', (req, res) => {
  const authUrl = 'https://accounts.google.com/o/oauth2/v2/auth';
  const params = new URLSearchParams({
    client_id: process.env.GOOGLE_CLIENT_ID,
    redirect_uri: 'https://yourapp.com/auth/callback',
    response_type: 'code',
    scope: 'openid email profile',
    state: generateRandomState() // CSRF protection
  });
  
  res.redirect(`${authUrl}?${params}`);
});

// Step 2: Handle callback with authorization code
app.get('/auth/callback', async (req, res) => {
  const { code, state } = req.query;
  
  // Verify state to prevent CSRF attacks
  if (!verifyState(state)) {
    return res.status(400).json({ error: 'Invalid state parameter' });
  }
  
  try {
    // Step 3: Exchange code for access token
    const tokenResponse = await axios.post(
      'https://oauth2.googleapis.com/token',
      {
        code,
        client_id: process.env.GOOGLE_CLIENT_ID,
        client_secret: process.env.GOOGLE_CLIENT_SECRET,
        redirect_uri: 'https://yourapp.com/auth/callback',
        grant_type: 'authorization_code'
      }
    );
    
    const { access_token, refresh_token, id_token } = tokenResponse.data;
    
    // Step 4: Use access token to get user info
    const userResponse = await axios.get(
      'https://www.googleapis.com/oauth2/v2/userinfo',
      {
        headers: { Authorization: `******}
      }
    );
    
    const user = userResponse.data;
    
    // Create or update user in your database
    const localUser = await db.users.upsert({
      googleId: user.id,
      email: user.email,
      name: user.name,
      picture: user.picture
    });
    
    // Create session for user
    const sessionToken = await createSession(localUser);
    
    res.cookie('session', sessionToken, {
      httpOnly: true,
      secure: true,
      sameSite: 'lax'
    });
    
    res.redirect('/dashboard');
  } catch (err) {
    res.status(500).json({
      error: {
        type: 'oauth_error',
        message: 'Failed to complete OAuth flow'
      }
    });
  }
});
```

**OAuth 2.0 Grant Types**:

**1. Authorization Code** (most secure, for web apps)
- User authorizes, gets code
- App exchanges code for token server-side
- Includes PKCE extension for added security

**2. Client Credentials** (for machine-to-machine)
- App authenticates directly
- No user interaction
- Used for service accounts

**3. Refresh Token** (for getting new access tokens)
- Exchange refresh token for new access token
- Avoids repeated logins

**4. ~~Implicit~~ and ~~Password~~** (deprecated, don't use)
- Implicit: Returns token directly (insecure)
- Password: App handles user password (defeats OAuth purpose)

**When to Use**:
- Social login ("Sign in with Google/Facebook/GitHub")
- Third-party integrations (accessing user data on other platforms)
- Federated identity
- When you need limited, scoped access

**Strengths**:
- Users never share passwords with your app
- Granular permissions (scopes)
- Token can be revoked independently
- Industry standard for delegated access
- Supports multiple grant types

**Weaknesses**:
- Complex to implement correctly
- Requires HTTPS
- Multiple round-trips
- Need to handle token refresh
- State management required

**Security Best Practices**:
```javascript
// Always use PKCE (Proof Key for Code Exchange)
// Generates a random code verifier and challenge

function generateCodeVerifier() {
  return crypto.randomBytes(32).toString('base64url');
}

function generateCodeChallenge(verifier) {
  return crypto
    .createHash('sha256')
    .update(verifier)
    .digest('base64url');
}

// Step 1: Generate and store verifier
const codeVerifier = generateCodeVerifier();
const codeChallenge = generateCodeChallenge(codeVerifier);

// Store verifier in session
req.session.codeVerifier = codeVerifier;

// Step 2: Include challenge in authorization request
const authUrl = `https://oauth.provider.com/authorize?` +
  `client_id=${clientId}&` +
  `code_challenge=${codeChallenge}&` +
  `code_challenge_method=S256`;

// Step 3: Include verifier when exchanging code
const token = await exchangeCode(code, codeVerifier);
```

**Standards Reference**: 
- RFC 6749 - OAuth 2.0 Framework
- RFC 7636 - Proof Key for Code Exchange (PKCE)
- RFC 8252 - OAuth for Native Apps

### Comparing Authentication Methods

| Method | Complexity | Security | Scalability | Use Case |
|--------|-----------|----------|-------------|----------|
| API Keys | Low | Medium | High | Server-to-server, internal APIs |
| Basic Auth | Low | Low (needs HTTPS) | High | Simple APIs, prototypes |
| JWT/****** | Medium | High | Very High | Modern web/mobile apps |
| OAuth 2.0 | High | Very High | High | Social login, third-party access |

## Implementing Secure Password Storage

If you're building your own authentication system (not using OAuth), you must store passwords securely.

**Never Do This**:
```javascript
// ❌ NEVER store passwords in plain text
await db.users.create({
  email: 'alice@example.com',
  password: 'MyPassword123'  // Anyone with DB access can see this!
});

// ❌ NEVER use simple hashing
const hash = crypto.createHash('md5').update(password).digest('hex');
// MD5 is broken, can be cracked in seconds
```

**Always Do This**:
```javascript
const bcrypt = require('bcrypt');

// Hash password with salt
async function hashPassword(password) {
  const saltRounds = 10;  // Higher = more secure but slower
  return await bcrypt.hash(password, saltRounds);
}

// Store hashed password
app.post('/api/auth/register', async (req, res) => {
  const { email, password } = req.body;
  
  // Validate password strength
  if (password.length < 8) {
    return res.status(400).json({
      error: {
        type: 'weak_password',
        message: 'Password must be at least 8 characters'
      }
    });
  }
  
  const passwordHash = await hashPassword(password);
  
  await db.users.create({
    email,
    passwordHash  // Store hash, not password
  });
  
  res.status(201).json({ message: 'User created' });
});

// Validate password
app.post('/api/auth/login', async (req, res) => {
  const { email, password } = req.body;
  
  const user = await db.users.findOne({ email });
  
  if (!user) {
    // Don't reveal whether email exists
    return res.status(401).json({
      error: {
        type: 'invalid_credentials',
        message: 'Invalid email or password'
      }
    });
  }
  
  const isValid = await bcrypt.compare(password, user.passwordHash);
  
  if (!isValid) {
    return res.status(401).json({
      error: {
        type: 'invalid_credentials',
        message: 'Invalid email or password'
      }
    });
  }
  
  // Issue token...
});
```

**Password Requirements**:
```javascript
function validatePassword(password) {
  const errors = [];
  
  if (password.length < 12) {
    errors.push('Must be at least 12 characters');
  }
  
  if (!/[a-z]/.test(password)) {
    errors.push('Must contain lowercase letter');
  }
  
  if (!/[A-Z]/.test(password)) {
    errors.push('Must contain uppercase letter');
  }
  
  if (!/[0-9]/.test(password)) {
    errors.push('Must contain number');
  }
  
  if (!/[^a-zA-Z0-9]/.test(password)) {
    errors.push('Must contain special character');
  }
  
  // Check against common passwords
  if (COMMON_PASSWORDS.includes(password.toLowerCase())) {
    errors.push('Password is too common');
  }
  
  return errors;
}
```

## Rate Limiting for Authentication

Authentication endpoints are prime targets for attacks. Always implement rate limiting.

```javascript
const rateLimit = require('express-rate-limit');

// Strict rate limit for login
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // 5 attempts per window
  message: {
    error: {
      type: 'rate_limit_exceeded',
      message: 'Too many login attempts, please try again later'
    }
  },
  standardHeaders: true,
  legacyHeaders: false,
});

app.post('/api/auth/login', loginLimiter, async (req, res) => {
  // Login logic...
});

// Account lockout after repeated failures
const loginAttempts = new Map();

async function checkLoginAttempts(email) {
  const attempts = loginAttempts.get(email) || 0;
  
  if (attempts >= 10) {
    // Lock account for 1 hour
    const lockUntil = Date.now() + 60 * 60 * 1000;
    await db.users.update({ email }, { lockedUntil: lockUntil });
    
    return false;
  }
  
  return true;
}

async function recordFailedLogin(email) {
  const attempts = (loginAttempts.get(email) || 0) + 1;
  loginAttempts.set(email, attempts);
  
  // Clear after 1 hour
  setTimeout(() => loginAttempts.delete(email), 60 * 60 * 1000);
}
```

## Multi-Factor Authentication (MFA)

Add a second layer of security beyond passwords.

**Common MFA Methods**:
1. **TOTP** (Time-based One-Time Password) - Google Authenticator, Authy
2. **SMS** - Text message codes (less secure, but convenient)
3. **Email** - Email codes
4. **Hardware tokens** - YubiKey, etc.

**TOTP Implementation**:
```javascript
const speakeasy = require('speakeasy');
const QRCode = require('qrcode');

// Generate MFA secret for user
app.post('/api/auth/mfa/setup', authenticateToken, async (req, res) => {
  const secret = speakeasy.generateSecret({
    name: `YourApp (${req.user.email})`
  });
  
  // Store secret in database
  await db.users.update(
    { id: req.user.sub },
    { mfaSecret: secret.base32, mfaEnabled: false }
  );
  
  // Generate QR code for user to scan
  const qrCodeUrl = await QRCode.toDataURL(secret.otpauth_url);
  
  res.json({
    secret: secret.base32,
    qr_code: qrCodeUrl
  });
});

// Verify and enable MFA
app.post('/api/auth/mfa/verify', authenticateToken, async (req, res) => {
  const { code } = req.body;
  
  const user = await db.users.findById(req.user.sub);
  
  const isValid = speakeasy.totp.verify({
    secret: user.mfaSecret,
    encoding: 'base32',
    token: code,
    window: 2 // Allow 2 time steps of clock skew
  });
  
  if (!isValid) {
    return res.status(400).json({
      error: {
        type: 'invalid_code',
        message: 'Invalid verification code'
      }
    });
  }
  
  // Enable MFA
  await db.users.update(
    { id: req.user.sub },
    { mfaEnabled: true }
  );
  
  res.json({ message: 'MFA enabled successfully' });
});

// Login with MFA
app.post('/api/auth/login', async (req, res) => {
  const { email, password, mfa_code } = req.body;
  
  const user = await validateCredentials(email, password);
  
  if (!user) {
    return res.status(401).json({
      error: {
        type: 'invalid_credentials',
        message: 'Invalid email or password'
      }
    });
  }
  
  // Check if MFA is enabled
  if (user.mfaEnabled) {
    if (!mfa_code) {
      return res.status(401).json({
        error: {
          type: 'mfa_required',
          message: 'MFA code is required'
        }
      });
    }
    
    const isValid = speakeasy.totp.verify({
      secret: user.mfaSecret,
      encoding: 'base32',
      token: mfa_code,
      window: 2
    });
    
    if (!isValid) {
      return res.status(401).json({
        error: {
          type: 'invalid_mfa_code',
          message: 'Invalid MFA code'
        }
      });
    }
  }
  
  // Issue tokens...
  const token = generateJWT(user);
  res.json({ access_token: token });
});
```

## Session Management

For traditional web applications, session-based authentication is still relevant.

```javascript
const session = require('express-session');
const RedisStore = require('connect-redis')(session);
const redis = require('redis');

const redisClient = redis.createClient();

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: true,        // HTTPS only
    httpOnly: true,      // Not accessible via JavaScript
    maxAge: 1000 * 60 * 60 * 24,  // 24 hours
    sameSite: 'lax'      // CSRF protection
  }
}));

// Login creates session
app.post('/api/auth/login', async (req, res) => {
  const user = await validateCredentials(req.body);
  
  if (!user) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  
  // Store user in session
  req.session.userId = user.id;
  req.session.email = user.email;
  
  res.json({ message: 'Logged in successfully' });
});

// Middleware to check session
function requireAuth(req, res, next) {
  if (!req.session.userId) {
    return res.status(401).json({
      error: {
        type: 'authentication_required',
        message: 'Please log in'
      }
    });
  }
  
  next();
}

// Logout destroys session
app.post('/api/auth/logout', (req, res) => {
  req.session.destroy((err) => {
    if (err) {
      return res.status(500).json({ error: 'Logout failed' });
    }
    
    res.clearCookie('connect.sid');
    res.json({ message: 'Logged out successfully' });
  });
});
```

## Security Checklist

When implementing authentication:

- [ ] Always use HTTPS in production
- [ ] Hash passwords with bcrypt (salt rounds >= 10)
- [ ] Implement rate limiting on auth endpoints
- [ ] Use secure, httpOnly cookies for sessions
- [ ] Set short expiration on access tokens (15-30 minutes)
- [ ] Implement token refresh mechanism
- [ ] Add MFA for sensitive operations
- [ ] Log all authentication events
- [ ] Monitor for suspicious patterns
- [ ] Implement account lockout after failed attempts
- [ ] Use CSRF tokens for session-based auth
- [ ] Validate all inputs rigorously
- [ ] Don't reveal whether email/username exists
- [ ] Use timing-safe comparison for tokens
- [ ] Rotate secrets periodically
- [ ] Have a plan for handling compromised credentials

## Common Security Vulnerabilities

### 1. Timing Attacks

```javascript
// ❌ Vulnerable to timing attack
function compareTokens(userToken, validToken) {
  if (userToken.length !== validToken.length) {
    return false;
  }
  
  for (let i = 0; i < userToken.length; i++) {
    if (userToken[i] !== validToken[i]) {
      return false;  // Returns early, leaks information through timing
    }
  }
  
  return true;
}

// ✅ Timing-safe comparison
const crypto = require('crypto');

function compareTokensSafe(userToken, validToken) {
  const userBuffer = Buffer.from(userToken);
  const validBuffer = Buffer.from(validToken);
  
  if (userBuffer.length !== validBuffer.length) {
    return false;
  }
  
  return crypto.timingSafeEqual(userBuffer, validBuffer);
}
```

### 2. Session Fixation

```javascript
// ✅ Regenerate session ID after login
app.post('/api/auth/login', async (req, res) => {
  const user = await validateCredentials(req.body);
  
  // Regenerate session ID
  req.session.regenerate((err) => {
    if (err) {
      return res.status(500).json({ error: 'Login failed' });
    }
    
    req.session.userId = user.id;
    res.json({ message: 'Logged in' });
  });
});
```

### 3. JWT Algorithm Confusion

```javascript
// ❌ Accepts any algorithm
jwt.verify(token, secret);

// ✅ Specify allowed algorithms
jwt.verify(token, secret, {
  algorithms: ['HS256']  // Only allow HS256
});
```

## Testing Authentication

```javascript
const request = require('supertest');
const app = require('../app');

describe('Authentication', () => {
  describe('POST /api/auth/login', () => {
    it('returns token for valid credentials', async () => {
      const response = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'test@example.com',
          password: 'ValidPassword123!'
        });
      
      expect(response.status).toBe(200);
      expect(response.body).toHaveProperty('access_token');
      expect(response.body.token_type).toBe('******);
    });
    
    it('returns 401 for invalid credentials', async () => {
      const response = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'test@example.com',
          password: 'WrongPassword'
        });
      
      expect(response.status).toBe(401);
      expect(response.body.error.type).toBe('invalid_credentials');
    });
    
    it('enforces rate limiting', async () => {
      const attempts = [];
      
      for (let i = 0; i < 6; i++) {
        attempts.push(
          request(app)
            .post('/api/auth/login')
            .send({ email: 'test@example.com', password: 'wrong' })
        );
      }
      
      const responses = await Promise.all(attempts);
      const lastResponse = responses[responses.length - 1];
      
      expect(lastResponse.status).toBe(429);
    });
  });
  
  describe('Protected endpoints', () => {
    it('returns 401 without token', async () => {
      const response = await request(app)
        .get('/api/users/me');
      
      expect(response.status).toBe(401);
    });
    
    it('returns 401 with invalid token', async () => {
      const response = await request(app)
        .get('/api/users/me')
        .set('Authorization', '******invalid-token');
      
      expect(response.status).toBe(401);
    });
    
    it('allows access with valid token', async () => {
      const loginResponse = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'test@example.com',
          password: 'ValidPassword123!'
        });
      
      const token = loginResponse.body.access_token;
      
      const response = await request(app)
        .get('/api/users/me')
        .set('Authorization', `******{token}`);
      
      expect(response.status).toBe(200);
      expect(response.body).toHaveProperty('email');
    });
  });
});
```

## Summary: Choosing Your Authentication Strategy

**For simple internal APIs**:
- Start with API keys
- Add rate limiting
- Monitor usage

**For modern web/mobile apps**:
- Use JWT with short expiration
- Implement refresh tokens
- Add MFA for sensitive operations

**For third-party integrations**:
- Implement OAuth 2.0
- Support standard scopes
- Provide clear developer documentation

**For maximum security**:
- Use OAuth 2.0 with PKCE
- Require MFA
- Implement hardware token support
- Add behavioral analysis

Authentication is not optional. It's the foundation of API security. Invest the time to get it right from the start.

## Further Reading

- **RFC 7235**: HTTP Authentication Framework
- **RFC 7617**: Basic HTTP Authentication
- **RFC 7519**: JSON Web Token (JWT)
- **RFC 6749**: OAuth 2.0 Framework
- **RFC 7636**: Proof Key for Code Exchange (PKCE)
- **OWASP Authentication Cheat Sheet**
- **NIST Digital Identity Guidelines**

---

**Next**: Learn how to filter data effectively in [Filtering](./filtering.md).

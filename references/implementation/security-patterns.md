# Security Patterns for JS/TS Applications

Comprehensive security reference organized around OWASP Top 10 (2021). Load this when
building any application with HTTP endpoints, user-facing input, authentication, or
sensitive data handling. Each section explains the *threat*, the *pattern to follow*,
and *code examples*.

---

## Table of Contents

1. [Authentication & Session Management](#1-authentication--session-management)
2. [Authorization (RBAC / ABAC)](#2-authorization-rbac--abac)
3. [Input Validation & Output Encoding](#3-input-validation--output-encoding)
4. [XSS Prevention](#4-xss-prevention)
5. [CSRF Protection](#5-csrf-protection)
6. [SSRF Prevention](#6-ssrf-prevention)
7. [Security Headers](#7-security-headers)
8. [Rate Limiting](#8-rate-limiting)
9. [Prototype Pollution](#9-prototype-pollution)
10. [Supply Chain Security](#10-supply-chain-security)
11. [Cryptography & Secrets](#11-cryptography--secrets)
12. [Logging & Error Disclosure](#12-logging--error-disclosure)

---

## 1. Authentication & Session Management

**Threat:** Broken authentication (OWASP A07:2021) — credential stuffing, session
hijacking, weak password storage, token theft.

### Password Hashing — Always bcrypt or argon2

```typescript
import bcrypt from 'bcrypt';

const SALT_ROUNDS = 12; // minimum 10, recommended 12+

async function hashPassword(plaintext: string): Promise<string> {
  return bcrypt.hash(plaintext, SALT_ROUNDS);
}

async function verifyPassword(plaintext: string, hash: string): Promise<boolean> {
  return bcrypt.compare(plaintext, hash);
}
```

**Never use:** MD5, SHA1, SHA256 (without key stretching), plain text. These are fast
hashes — attackers can try billions per second. bcrypt/argon2 are deliberately slow.

### JWT Best Practices

```typescript
import jwt from 'jsonwebtoken';

interface TokenPayload {
  sub: string;       // user ID
  role: string;
  iat: number;
  exp: number;
}

function createToken(userId: string, role: string, secret: string): string {
  return jwt.sign(
    { sub: userId, role },
    secret,
    {
      algorithm: 'HS256',   // explicit algorithm — prevent "none" attack
      expiresIn: '15m',     // short-lived access tokens
      issuer: 'your-app',
      audience: 'your-app',
    }
  );
}

function verifyToken(token: string, secret: string): TokenPayload {
  return jwt.verify(token, secret, {
    algorithms: ['HS256'],  // allowlist algorithms — prevent algorithm switching
    issuer: 'your-app',
    audience: 'your-app',
    clockTolerance: 30,     // 30s tolerance for clock skew
  }) as TokenPayload;
}
```

**Key rules:**
- Always set `algorithms` allowlist in `verify()` — prevents algorithm confusion attacks
- Short-lived access tokens (5-15 min) + longer refresh tokens (7-30 days)
- Store refresh tokens server-side (database), not just in JWT
- Rotate refresh tokens on use (one-time use)
- Never store JWTs in localStorage (XSS-accessible) — use HttpOnly cookies

### Session Cookies

```typescript
// Express / Fastify session cookie settings
const sessionConfig = {
  cookie: {
    httpOnly: true,      // not accessible via document.cookie (XSS protection)
    secure: true,        // only sent over HTTPS
    sameSite: 'lax',     // CSRF protection (use 'strict' for sensitive apps)
    maxAge: 15 * 60 * 1000, // 15 minutes
    path: '/',
    domain: '.yourdomain.com',
  },
  name: '__Host-session', // __Host- prefix enforces Secure + no Domain
};
```

---

## 2. Authorization (RBAC / ABAC)

**Threat:** Broken access control (OWASP A01:2021) — the #1 web vulnerability.
Users accessing resources/actions they shouldn't.

### Role-Based Access Control (RBAC)

```typescript
// Define permissions per role
const PERMISSIONS = {
  admin: ['read', 'write', 'delete', 'manage-users'] as const,
  editor: ['read', 'write'] as const,
  viewer: ['read'] as const,
} as const;

type Role = keyof typeof PERMISSIONS;
type Permission = typeof PERMISSIONS[Role][number];

function hasPermission(role: Role, permission: Permission): boolean {
  return (PERMISSIONS[role] as readonly string[]).includes(permission);
}

// Middleware pattern
function requirePermission(permission: Permission) {
  return (req: Request, res: Response, next: NextFunction) => {
    const user = req.user; // set by auth middleware
    if (!user || !hasPermission(user.role, permission)) {
      return res.status(403).json({ error: 'FORBIDDEN' });
    }
    next();
  };
}

// Usage
app.delete('/api/users/:id', requirePermission('manage-users'), deleteUserHandler);
```

### Object-Level Authorization (IDOR Prevention)

```typescript
// ❌ Bad — no ownership check (IDOR vulnerability)
async function getDocument(req: Request) {
  const doc = await db.document.findUnique({ where: { id: req.params.id } });
  return doc; // any user can access any document by guessing IDs
}

// ✅ Good — verify the requesting user owns/can access the resource
async function getDocument(req: Request) {
  const doc = await db.document.findUnique({
    where: {
      id: req.params.id,
      OR: [
        { ownerId: req.user.id },              // owner
        { shares: { some: { userId: req.user.id } } }, // shared with user
      ],
    },
  });

  if (!doc) {
    // Return 404, not 403 — don't reveal that the resource exists
    throw new NotFoundError('Document not found');
  }

  return doc;
}
```

**Key rules:**
- Check authorization on EVERY endpoint, not just the frontend
- Use 404 (not 403) when a user can't access a resource — prevents enumeration
- Never trust client-side role/permission claims
- Apply principle of least privilege — default deny

---

## 3. Input Validation & Output Encoding

**Threat:** Injection (OWASP A03:2021) — SQL injection, NoSQL injection, command
injection, header injection, template injection.

### Validation at the Boundary

```typescript
import { z } from 'zod';

// Validate ALL external input at the entry point
const CreatePostSchema = z.object({
  title: z.string()
    .min(1, 'Title is required')
    .max(200, 'Title too long')
    .trim(),
  body: z.string()
    .min(1)
    .max(50_000),
  tags: z.array(z.string().max(50))
    .max(10)
    .default([]),
  // Reject unexpected fields — prevent mass assignment
}).strict();

// Validate URL parameters — they're strings, not numbers
const IdParamSchema = z.object({
  id: z.string().uuid('Invalid ID format'),
});

// Validate query parameters
const PaginationSchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  sort: z.enum(['created_at', 'updated_at', 'title']).default('created_at'),
  order: z.enum(['asc', 'desc']).default('desc'),
});
```

**Key rules:**
- Validate at system boundaries (HTTP handlers, message consumers, file readers)
- Use allowlists (enum of valid values) over denylists (blocking "bad" values)
- Use `.strict()` in Zod to reject unexpected fields (mass assignment prevention)
- Validate URL params, query params, headers, AND body — not just body
- Coerce types explicitly — query params and URL params are always strings

### Output Encoding

```typescript
// ❌ Bad — user content rendered directly into HTML
const html = `<div>${userComment}</div>`;

// ✅ Good — encode before rendering
import { encode } from 'html-entities';
const html = `<div>${encode(userComment)}</div>`;

// ✅ Better — use a templating engine with auto-escaping
// EJS, Handlebars, React JSX all auto-escape by default

// For JSON responses — safe by default (JSON.stringify escapes strings)
res.json({ comment: userComment }); // ✅ Safe
```

### Command Injection Prevention

```typescript
import { execFile } from 'node:child_process';

// ❌ NEVER use exec/spawn with shell and user input
import { exec } from 'node:child_process';
exec(`convert ${userFilename} output.png`); // shell injection

// ✅ Use execFile — no shell, arguments are passed as array
execFile('convert', [userFilename, 'output.png'], (error, stdout) => {
  // arguments are not interpreted by a shell
});

// ✅ Or use execa with explicit arguments
import { execa } from 'execa';
await execa('convert', [userFilename, 'output.png']);
```

---

## 4. XSS Prevention

**Threat:** Cross-Site Scripting (part of OWASP A03:2021) — attacker injects scripts
into pages viewed by other users.

### React / JSX — Mostly Safe by Default

```typescript
// ✅ Safe — JSX auto-escapes expressions
function Comment({ text }: { text: string }) {
  return <p>{text}</p>; // XSS-safe: React escapes the text
}

// ❌ DANGEROUS — dangerouslySetInnerHTML bypasses escaping
function Comment({ html }: { html: string }) {
  return <p dangerouslySetInnerHTML={{ __html: html }} />; // XSS vulnerability
}

// ✅ If you must render HTML, sanitize first
import DOMPurify from 'dompurify';
function Comment({ html }: { html: string }) {
  const clean = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br'],
    ALLOWED_ATTR: ['href'],
  });
  return <p dangerouslySetInnerHTML={{ __html: clean }} />;
}
```

### URL-Based XSS

```typescript
// ❌ Dangerous — javascript: protocol in href
function Link({ url }: { url: string }) {
  return <a href={url}>Click</a>; // If url = "javascript:alert(1)" → XSS
}

// ✅ Safe — validate URL protocol
function sanitizeUrl(url: string): string {
  try {
    const parsed = new URL(url);
    if (!['http:', 'https:', 'mailto:'].includes(parsed.protocol)) {
      return '#'; // reject non-http protocols
    }
    return parsed.toString();
  } catch {
    return '#'; // invalid URL
  }
}
```

### Server-Side XSS

```typescript
// ❌ Dangerous — string interpolation in HTML response
app.get('/search', (req, res) => {
  res.send(`<p>Results for: ${req.query.q}</p>`); // reflected XSS
});

// ✅ Safe — use a template engine with auto-escaping, or encode manually
import { encode } from 'html-entities';
app.get('/search', (req, res) => {
  res.send(`<p>Results for: ${encode(String(req.query.q))}</p>`);
});
```

---

## 5. CSRF Protection

**Threat:** Cross-Site Request Forgery — attacker tricks authenticated user into
making unintended requests.

### Token-Based CSRF Protection

```typescript
import crypto from 'node:crypto';

// Generate CSRF token per session
function generateCsrfToken(): string {
  return crypto.randomBytes(32).toString('hex');
}

// Middleware: set token in cookie + validate on mutations
function csrfProtection(req: Request, res: Response, next: NextFunction) {
  if (['GET', 'HEAD', 'OPTIONS'].includes(req.method)) {
    // Safe methods — set/refresh the token
    const token = generateCsrfToken();
    req.session.csrfToken = token;
    res.cookie('csrf-token', token, { httpOnly: false }); // JS-readable
    return next();
  }

  // Mutation methods — validate the token
  const headerToken = req.headers['x-csrf-token'];
  const sessionToken = req.session.csrfToken;

  if (!headerToken || !sessionToken || headerToken !== sessionToken) {
    return res.status(403).json({ error: 'CSRF_VALIDATION_FAILED' });
  }
  next();
}
```

### SameSite Cookies (Modern Alternative)

```typescript
// SameSite=Lax prevents most CSRF without tokens
// SameSite=Strict is stronger but breaks legitimate cross-site navigation
const cookieOptions = {
  sameSite: 'lax' as const,  // prevents CSRF from cross-site POST
  httpOnly: true,
  secure: true,
};
```

**Key rules:**
- SameSite=Lax is the minimum — covers most CSRF vectors
- For sensitive operations (money transfer, account deletion), use explicit CSRF tokens
- APIs using Bearer tokens (not cookies) are inherently CSRF-safe

---

## 6. SSRF Prevention

**Threat:** Server-Side Request Forgery (OWASP A10:2021) — attacker makes the server
fetch internal resources (metadata endpoints, internal services, localhost).

```typescript
import { URL } from 'node:url';
import dns from 'node:dns/promises';
import { isPrivate } from 'ip'; // or implement manually

// ❌ Dangerous — fetch any URL the user provides
async function fetchUrl(userUrl: string) {
  return fetch(userUrl); // can access http://169.254.169.254/metadata, localhost, etc.
}

// ✅ Safe — validate URL and resolved IP before fetching
const BLOCKED_HOSTS = new Set(['localhost', '127.0.0.1', '0.0.0.0', '::1']);

async function safeFetch(userUrl: string): Promise<Response> {
  // 1. Parse and validate URL
  let parsed: URL;
  try {
    parsed = new URL(userUrl);
  } catch {
    throw new ValidationError('Invalid URL');
  }

  // 2. Only allow http/https
  if (!['http:', 'https:'].includes(parsed.protocol)) {
    throw new ValidationError('Only HTTP(S) URLs allowed');
  }

  // 3. Block known internal hostnames
  if (BLOCKED_HOSTS.has(parsed.hostname)) {
    throw new ValidationError('Internal hosts not allowed');
  }

  // 4. Resolve DNS and check for private IPs
  const addresses = await dns.resolve4(parsed.hostname);
  for (const addr of addresses) {
    if (isPrivateIP(addr)) {
      throw new ValidationError('Private IP addresses not allowed');
    }
  }

  // 5. Fetch with timeout
  return fetch(userUrl, { signal: AbortSignal.timeout(10_000) });
}

function isPrivateIP(ip: string): boolean {
  const parts = ip.split('.').map(Number);
  return (
    parts[0] === 10 ||                               // 10.0.0.0/8
    (parts[0] === 172 && parts[1] >= 16 && parts[1] <= 31) || // 172.16.0.0/12
    (parts[0] === 192 && parts[1] === 168) ||         // 192.168.0.0/16
    (parts[0] === 127) ||                             // 127.0.0.0/8
    (parts[0] === 169 && parts[1] === 254)            // 169.254.0.0/16 (link-local/cloud metadata)
  );
}
```

**Key rules:**
- Always validate user-supplied URLs before server-side fetching
- Block private/internal IP ranges (10.x, 172.16-31.x, 192.168.x, 169.254.x, 127.x)
- Block cloud metadata endpoints (169.254.169.254)
- Use an allowlist of domains when possible
- Set timeouts on all outbound requests

---

## 7. Security Headers

**Threat:** Security misconfiguration (OWASP A05:2021) — missing headers that enable
clickjacking, MIME sniffing, XSS, and other attacks.

### Helmet.js (Express / Fastify)

```typescript
import helmet from 'helmet';

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],        // no 'unsafe-inline' or 'unsafe-eval'
      styleSrc: ["'self'", "'unsafe-inline'"], // inline styles often needed
      imgSrc: ["'self'", 'data:', 'https:'],
      connectSrc: ["'self'"],
      fontSrc: ["'self'"],
      objectSrc: ["'none'"],        // block Flash, Java applets
      frameAncestors: ["'none'"],   // prevent clickjacking
      upgradeInsecureRequests: [],
    },
  },
  crossOriginEmbedderPolicy: true,
  crossOriginOpenerPolicy: true,
  crossOriginResourcePolicy: { policy: 'same-origin' },
  hsts: {
    maxAge: 31536000,               // 1 year
    includeSubDomains: true,
    preload: true,
  },
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
}));
```

### Manual Headers (for non-Express frameworks)

```typescript
function securityHeaders(res: Response): void {
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader('X-XSS-Protection', '0');        // disable — CSP is better
  res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
  res.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin');
  res.setHeader('Permissions-Policy', 'camera=(), microphone=(), geolocation=()');
}
```

### CORS Configuration

```typescript
// ❌ Bad — overly permissive
app.use(cors({ origin: '*', credentials: true })); // credentials + wildcard = vulnerability

// ✅ Good — explicit allowlist
const ALLOWED_ORIGINS = new Set([
  'https://yourdomain.com',
  'https://app.yourdomain.com',
]);

app.use(cors({
  origin: (origin, callback) => {
    if (!origin || ALLOWED_ORIGINS.has(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-CSRF-Token'],
  maxAge: 86400, // cache preflight for 24h
}));
```

---

## 8. Rate Limiting

**Threat:** Brute force, credential stuffing, DoS, resource exhaustion.

### Express Rate Limiting

```typescript
import rateLimit from 'express-rate-limit';

// General API rate limit
const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 100,                    // 100 requests per window
  standardHeaders: true,       // RateLimit-* headers
  legacyHeaders: false,
  message: { error: 'RATE_LIMIT_EXCEEDED', retryAfter: 900 },
});

// Strict limit for auth endpoints
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,                     // 5 attempts per 15 minutes
  skipSuccessfulRequests: true, // don't count successful logins
  message: { error: 'TOO_MANY_LOGIN_ATTEMPTS', retryAfter: 900 },
});

app.use('/api/', apiLimiter);
app.use('/api/auth/login', authLimiter);
app.use('/api/auth/signup', authLimiter);
```

### Redis-Based Rate Limiting (Distributed)

```typescript
import { RateLimiterRedis } from 'rate-limiter-flexible';

const rateLimiter = new RateLimiterRedis({
  storeClient: redisClient,
  keyPrefix: 'rl',
  points: 100,           // requests
  duration: 900,          // per 15 minutes
  blockDuration: 900,     // block for 15 min after exceeding
});

async function rateLimitMiddleware(req: Request, res: Response, next: NextFunction) {
  try {
    const key = req.ip ?? 'unknown';
    await rateLimiter.consume(key);
    next();
  } catch (rateLimiterRes) {
    res.setHeader('Retry-After', String(Math.ceil(rateLimiterRes.msBeforeNext / 1000)));
    res.status(429).json({ error: 'RATE_LIMIT_EXCEEDED' });
  }
}
```

**Key rules:**
- Rate limit ALL public endpoints (not just auth)
- Stricter limits on auth endpoints (5-10 attempts per 15 min)
- Use Redis for distributed rate limiting (multi-instance deployments)
- Return `Retry-After` header with 429 responses
- Consider per-user AND per-IP limiting

---

## 9. Prototype Pollution

**Threat:** Attacker modifies `Object.prototype` via `__proto__`, `constructor`, or
`prototype` properties in user input, affecting all objects in the process.

```typescript
// ❌ Vulnerable — recursive merge without protection
function deepMerge(target: any, source: any) {
  for (const key of Object.keys(source)) {
    if (typeof source[key] === 'object') {
      target[key] = deepMerge(target[key] || {}, source[key]);
    } else {
      target[key] = source[key];
    }
  }
  return target;
}
// Attacker sends: { "__proto__": { "isAdmin": true } }
// Now every object has isAdmin === true

// ✅ Safe — reject dangerous keys
const DANGEROUS_KEYS = new Set(['__proto__', 'constructor', 'prototype']);

function safeMerge(target: Record<string, unknown>, source: Record<string, unknown>): Record<string, unknown> {
  for (const key of Object.keys(source)) {
    if (DANGEROUS_KEYS.has(key)) continue; // skip prototype pollution vectors

    const sourceVal = source[key];
    const targetVal = target[key];

    if (isPlainObject(sourceVal) && isPlainObject(targetVal)) {
      target[key] = safeMerge(
        targetVal as Record<string, unknown>,
        sourceVal as Record<string, unknown>,
      );
    } else {
      target[key] = sourceVal;
    }
  }
  return target;
}

// ✅ Better — use Object.create(null) for user-controlled objects
const userConfig = Object.create(null); // no prototype chain
```

**Key rules:**
- Never use recursive merge/deep-clone on user input without filtering keys
- Use Zod `.strict()` to reject unexpected fields (prevents `__proto__` in parsed input)
- Use `Object.create(null)` for dictionaries built from user input
- Use `Map` instead of plain objects for key-value stores with user-controlled keys
- Avoid `lodash.merge`, `lodash.defaultsDeep` on untrusted input (historically vulnerable)

---

## 10. Supply Chain Security

**Threat:** Vulnerable dependencies (OWASP A06:2021) — packages with known CVEs,
typosquatted packages, hallucinated packages that don't exist.

### Dependency Verification Protocol

Before installing ANY package:

```bash
# 1. Verify the package exists and is legitimate
npm info {package-name}

# Check for:
# - Downloads/week (< 100 is suspicious for a well-known package)
# - Last publish date (abandoned packages are risky)
# - Maintainer count (single-maintainer packages are higher risk)
# - Repository link (should point to a real, active repo)

# 2. Check for known vulnerabilities
npm audit

# 3. Check for typosquatting — verify the exact name
# Common patterns: lodash vs 1odash, express vs expres, chalk vs cha1k
```

### Lockfile & Pinning

```jsonc
// package.json — pin exact versions for critical dependencies
{
  "dependencies": {
    "express": "4.18.2",       // exact version, not ^4.18.2
    "jsonwebtoken": "9.0.2"
  },
  "overrides": {
    // Force resolution of transitive dependency with known CVE
    "vulnerable-package": ">=2.0.1"
  }
}
```

**Key rules:**
- Always commit lockfiles (package-lock.json, yarn.lock, pnpm-lock.yaml)
- Run `npm audit` after every install and before every deployment
- Verify package legitimacy before first install (check npm info, GitHub repo)
- Watch for hallucinated packages — if a package name doesn't exist on npm, DO NOT
  create it or install it. Flag it as a potential hallucination.
- Use `npm audit fix` for automated patches, review breaking changes manually
- Consider using `Socket.dev` or `Snyk` for deeper supply chain analysis

---

## 11. Cryptography & Secrets

**Threat:** Cryptographic failures (OWASP A02:2021) — weak encryption, exposed secrets,
broken random number generation.

### Secure Random Values

```typescript
import crypto from 'node:crypto';

// ✅ Cryptographically secure random values
const token = crypto.randomBytes(32).toString('hex');
const uuid = crypto.randomUUID();

// ❌ NEVER use Math.random() for security-sensitive values
const insecureToken = Math.random().toString(36); // predictable!
```

### Timing-Safe Comparison

```typescript
import crypto from 'node:crypto';

// ❌ Vulnerable to timing attacks
function verifyToken(provided: string, expected: string): boolean {
  return provided === expected; // early-exit reveals string length and content
}

// ✅ Constant-time comparison
function verifyToken(provided: string, expected: string): boolean {
  if (provided.length !== expected.length) return false;
  return crypto.timingSafeEqual(
    Buffer.from(provided),
    Buffer.from(expected),
  );
}
```

### Environment-Based Secret Management

```typescript
// ❌ NEVER — hardcoded secrets
const SECRET = 'my-secret-key-123';

// ❌ NEVER — secrets in source code, even in "config" files
export const config = { dbPassword: 'password123' };

// ✅ Always load from environment, validate at startup
import { z } from 'zod';

const SecretsSchema = z.object({
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32, 'JWT secret must be at least 32 characters'),
  API_KEY: z.string().min(1),
});

const secrets = SecretsSchema.parse(process.env);
```

---

## 12. Logging & Error Disclosure

**Threat:** Security logging and monitoring failures (OWASP A09:2021), sensitive data
exposure through error messages.

### Safe Error Responses

```typescript
// ❌ Bad — leaks internal details
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  res.status(500).json({
    error: err.message,    // may contain SQL, file paths, stack traces
    stack: err.stack,      // reveals internal code structure
  });
});

// ✅ Good — generic response to client, detailed logging internally
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  const requestId = crypto.randomUUID();

  // Log full details internally (with request correlation)
  logger.error({
    requestId,
    error: err.message,
    stack: err.stack,
    path: req.path,
    method: req.method,
    userId: req.user?.id,
  });

  // Return generic message to client
  if (err instanceof AppError) {
    res.status(err.statusCode).json({
      error: err.code,        // e.g., 'VALIDATION_ERROR', 'NOT_FOUND'
      message: err.userMessage, // safe, user-facing message
      requestId,               // for support correlation
    });
  } else {
    res.status(500).json({
      error: 'INTERNAL_ERROR',
      message: 'An unexpected error occurred',
      requestId,
    });
  }
});
```

### Safe Logging

```typescript
// ❌ Bad — sensitive data in logs
logger.info('User login', { email, password, token });
logger.debug('Request body', req.body); // may contain passwords, tokens

// ✅ Good — redact sensitive fields
const SENSITIVE_FIELDS = new Set([
  'password', 'token', 'secret', 'authorization',
  'cookie', 'creditCard', 'ssn', 'apiKey',
]);

function redact(obj: Record<string, unknown>): Record<string, unknown> {
  return Object.fromEntries(
    Object.entries(obj).map(([key, value]) =>
      SENSITIVE_FIELDS.has(key.toLowerCase())
        ? [key, '[REDACTED]']
        : [key, value]
    )
  );
}

// ✅ Log security events for monitoring
logger.warn('Failed login attempt', { email, ip: req.ip, attempt: count });
logger.warn('Authorization denied', { userId, resource, action });
logger.info('Password changed', { userId }); // not the password itself
```

**Key rules:**
- Never expose stack traces, SQL errors, or file paths to clients
- Use request IDs to correlate client errors with server logs
- Redact passwords, tokens, API keys, PII from all logs
- Log security events (failed logins, auth failures, rate limits) for monitoring
- Log at appropriate levels: security events at WARN or ERROR

---

## Security Decision Checklist

When implementing any endpoint or feature, verify:

| Check | Question |
|-------|----------|
| **Input** | Is all external input validated with a schema? |
| **Output** | Is user content encoded/escaped before rendering in HTML? |
| **Auth** | Does this endpoint require authentication? Is it enforced? |
| **Authz** | Does this endpoint check that the user can access THIS specific resource? |
| **Injection** | Are queries parameterized? Is user input kept out of commands/templates? |
| **Secrets** | Are credentials loaded from environment, not hardcoded? |
| **Headers** | Are security headers set (CSP, HSTS, X-Content-Type-Options)? |
| **Rate limit** | Is this endpoint rate-limited? Are auth endpoints strictly limited? |
| **Errors** | Do error responses hide internal details? |
| **Logging** | Are security events logged? Is sensitive data redacted? |
| **Deps** | Were new dependencies verified for legitimacy and known CVEs? |
| **CSRF** | If using cookies for auth, is CSRF protection in place? |
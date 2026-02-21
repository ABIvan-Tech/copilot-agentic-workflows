# Security Best Practices

## Overview
Comprehensive security guidelines for building secure applications, protecting against common vulnerabilities, and following defense-in-depth principles.

## When to Use This Skill
- Implementing authentication/authorization
- Handling sensitive data (PII, credentials, tokens)
- Working with user input
- Building APIs and endpoints
- Reviewing code for security issues
- Implementing security controls

## OWASP Top 10 Security Risks

### 1. Broken Access Control
**Risk**: Users can access resources they shouldn't

**Prevention:**
```javascript
// ✅ Good - Check authorization
app.get('/api/users/:id/profile', authenticate, async (req, res) => {
  const requestedUserId = req.params.id;
  
  // Verify user can only access their own profile
  if (req.user.id !== requestedUserId && req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Access denied' });
  }
  
  const profile = await getUserProfile(requestedUserId);
  res.json(profile);
});

// ❌ Bad - No authorization check
app.get('/api/users/:id/profile', async (req, res) => {
  const profile = await getUserProfile(req.params.id);
  res.json(profile); // Anyone can view any profile!
});
```

### 2. Cryptographic Failures
**Risk**: Sensitive data exposed due to weak/missing encryption

**Prevention:**
```javascript
// ✅ Good - Hash passwords with bcrypt/argon2
const bcrypt = require('bcrypt');
const SALT_ROUNDS = 12;

async function hashPassword(password) {
  return await bcrypt.hash(password, SALT_ROUNDS);
}

async function verifyPassword(password, hash) {
  return await bcrypt.compare(password, hash);
}

// ❌ Bad - Storing plain text passwords
user.password = password; // NEVER DO THIS

// ❌ Bad - Weak hashing (MD5, SHA1)
user.password = crypto.createHash('md5').update(password).digest('hex');
```

**Encrypt Sensitive Data at Rest:**
```javascript
const crypto = require('crypto');

// Use environment variables for keys
const ENCRYPTION_KEY = process.env.ENCRYPTION_KEY; // Must be 32 bytes
const IV_LENGTH = 16;

function encrypt(text) {
  const iv = crypto.randomBytes(IV_LENGTH);
  const cipher = crypto.createCipheriv('aes-256-gcm', Buffer.from(ENCRYPTION_KEY), iv);
  let encrypted = cipher.update(text);
  encrypted = Buffer.concat([encrypted, cipher.final()]);
  const authTag = cipher.getAuthTag();
  return iv.toString('hex') + ':' + authTag.toString('hex') + ':' + encrypted.toString('hex');
}

function decrypt(text) {
  const parts = text.split(':');
  const iv = Buffer.from(parts[0], 'hex');
  const authTag = Buffer.from(parts[1], 'hex');
  const encrypted = Buffer.from(parts[2], 'hex');
  const decipher = crypto.createDecipheriv('aes-256-gcm', Buffer.from(ENCRYPTION_KEY), iv);
  decipher.setAuthTag(authTag);
  let decrypted = decipher.update(encrypted);
  decrypted = Buffer.concat([decrypted, decipher.final()]);
  return decrypted.toString();
}
```

### 3. Injection Attacks
**Risk**: Malicious code injected into queries/commands

**SQL Injection Prevention:**
```javascript
// ✅ Good - Parameterized queries
const user = await db.query(
  'SELECT * FROM users WHERE email = $1 AND password = $2',
  [email, hashedPassword]
);

// ❌ Bad - String concatenation
const user = await db.query(
  `SELECT * FROM users WHERE email = '${email}' AND password = '${password}'`
);
// Attacker can use: ' OR '1'='1
```

**NoSQL Injection Prevention:**
```javascript
// ✅ Good - Validate and sanitize
const email = String(req.body.email).trim();
if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
  return res.status(400).json({ error: 'Invalid email' });
}
const user = await User.findOne({ email });

// ❌ Bad - Direct object injection
const user = await User.findOne(req.body);
// Attacker can send: { $ne: null } to bypass authentication
```

**Command Injection Prevention:**
```javascript
// ✅ Good - Use libraries, avoid shell execution
const sharp = require('sharp');
await sharp(inputPath).resize(800, 600).toFile(outputPath);

// ❌ Bad - Executing shell commands with user input
const { exec } = require('child_process');
exec(`convert ${userInput} -resize 800x600 output.jpg`);
// Attacker can inject: ; rm -rf /
```

### 4. Insecure Design
**Risk**: Fundamental flaws in application architecture

**Prevention:**
- Implement principle of least privilege
- Use defense in depth (multiple layers of security)
- Fail securely (secure defaults)
- Separate duties (different roles/permissions)
- Follow secure design patterns

### 5. Security Misconfiguration
**Risk**: Insecure default configurations, unnecessary features enabled

**Prevention:**
```javascript
// ✅ Good - Secure Express configuration
const express = require('express');
const helmet = require('helmet');
const app = express();

// Security headers
app.use(helmet());

// Disable X-Powered-By header
app.disable('x-powered-by');

// CORS configuration
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS.split(','),
  credentials: true,
  maxAge: 86400
}));

// Rate limiting
const rateLimit = require('express-rate-limit');
app.use(rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // limit each IP to 100 requests per windowMs
}));

// ❌ Bad - Insecure configuration
app.use(cors({ origin: '*' })); // Allow all origins
app.set('trust proxy', true); // Without understanding implications
```

### 6. Vulnerable and Outdated Components
**Risk**: Using libraries with known vulnerabilities

**Prevention:**
```bash
# Regular dependency audits
npm audit
npm audit fix

# Use automated tools
npm install -g npm-check-updates
ncu -u

# Check for known vulnerabilities
npm install -g snyk
snyk test
```

### 7. Identification and Authentication Failures
**Risk**: Weak authentication mechanisms

**Prevention:**
```javascript
// ✅ Good - Secure session configuration
const session = require('express-session');
const RedisStore = require('connect-redis')(session);

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: true, // HTTPS only
    httpOnly: true, // No JavaScript access
    maxAge: 3600000, // 1 hour
    sameSite: 'strict' // CSRF protection
  }
}));

// ✅ Good - JWT with proper expiration
const jwt = require('jsonwebtoken');

function generateToken(userId) {
  return jwt.sign(
    { userId },
    process.env.JWT_SECRET,
    { expiresIn: '1h', issuer: 'your-app', audience: 'your-app-users' }
  );
}

function verifyToken(token) {
  try {
    return jwt.verify(token, process.env.JWT_SECRET, {
      issuer: 'your-app',
      audience: 'your-app-users'
    });
  } catch (error) {
    throw new Error('Invalid token');
  }
}

// ❌ Bad - Weak session configuration
app.use(session({
  secret: 'keyboard cat', // Hardcoded secret
  cookie: { secure: false } // Not HTTPS only
}));
```

**Multi-Factor Authentication:**
```javascript
const speakeasy = require('speakeasy');

// Generate secret for user
function generateMFASecret() {
  return speakeasy.generateSecret({
    name: 'YourApp',
    length: 32
  });
}

// Verify MFA token
function verifyMFAToken(secret, token) {
  return speakeasy.totp.verify({
    secret: secret,
    encoding: 'base32',
    token: token,
    window: 2 // Allow 2 time steps tolerance
  });
}
```

### 8. Software and Data Integrity Failures
**Risk**: Code/data modified without verification

**Prevention:**
```javascript
// ✅ Good - Verify package integrity
// package-lock.json maintains integrity hashes

// ✅ Good - Verify file uploads
const crypto = require('crypto');

function calculateHash(data) {
  return crypto.createHash('sha256').update(data).digest('hex');
}

function verifyIntegrity(data, expectedHash) {
  const actualHash = calculateHash(data);
  if (actualHash !== expectedHash) {
    throw new Error('Data integrity check failed');
  }
}
```

### 9. Security Logging and Monitoring Failures
**Risk**: Attacks go undetected

**Prevention:**
```javascript
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'security.log', level: 'warn' })
  ]
});

// Log security events
function logSecurityEvent(event, details) {
  logger.warn('SECURITY_EVENT', {
    event,
    ...details,
    timestamp: new Date().toISOString(),
    ip: details.ip,
    userId: details.userId
  });
}

// Examples
logSecurityEvent('FAILED_LOGIN', { email, ip: req.ip, attempts: 5 });
logSecurityEvent('UNAUTHORIZED_ACCESS', { userId, resource: '/admin', ip: req.ip });
logSecurityEvent('SUSPICIOUS_ACTIVITY', { userId, action: 'bulk_download', ip: req.ip });

// ⚠️ Never log sensitive data
// ❌ Don't log: passwords, tokens, credit cards, SSN, etc.
```

### 10. Server-Side Request Forgery (SSRF)
**Risk**: Attacker makes server perform unintended requests

**Prevention:**
```javascript
// ✅ Good - Whitelist allowed domains
const allowedDomains = ['api.example.com', 'cdn.example.com'];

function isAllowedURL(url) {
  try {
    const parsed = new URL(url);
    return allowedDomains.includes(parsed.hostname);
  } catch {
    return false;
  }
}

app.post('/fetch-content', async (req, res) => {
  const { url } = req.body;
  
  if (!isAllowedURL(url)) {
    return res.status(400).json({ error: 'Invalid URL' });
  }
  
  const response = await fetch(url);
  res.json(await response.json());
});

// ❌ Bad - No validation
app.post('/fetch-content', async (req, res) => {
  const response = await fetch(req.body.url);
  res.json(await response.json());
  // Attacker could access internal services: http://localhost:8080/admin
});
```

## Input Validation & Sanitization

### Validate All Inputs
```javascript
const Joi = require('joi');

const userSchema = Joi.object({
  email: Joi.string().email().required(),
  password: Joi.string().min(8).max(128).required(),
  age: Joi.number().integer().min(13).max(120),
  website: Joi.string().uri()
});

app.post('/register', async (req, res) => {
  const { error, value } = userSchema.validate(req.body);
  
  if (error) {
    return res.status(400).json({ error: error.details[0].message });
  }
  
  // Use validated 'value' instead of req.body
  await createUser(value);
  res.status(201).json({ message: 'User created' });
});
```

### Sanitize HTML Output (XSS Prevention)
```javascript
const DOMPurify = require('isomorphic-dompurify');

// ✅ Good - Sanitize user content
function sanitizeHTML(dirty) {
  return DOMPurify.sanitize(dirty, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p'],
    ALLOWED_ATTR: ['href']
  });
}

// In React/Vue - use proper escaping
// React automatically escapes by default
<div>{userInput}</div> // Safe

// ❌ Dangerous
<div dangerouslySetInnerHTML={{ __html: userInput }} /> // XSS risk
```

## Secrets Management

### Never Commit Secrets
```bash
# .gitignore
.env
.env.local
secrets/
*.pem
*.key

# Use environment variables
DATABASE_URL=postgresql://user:pass@localhost:5432/db
JWT_SECRET=your-secret-key-here
API_KEY=your-api-key-here
```

### Use Secret Management Services
```javascript
// AWS Secrets Manager
const AWS = require('aws-sdk');
const secretsManager = new AWS.SecretsManager();

async function getSecret(secretName) {
  const data = await secretsManager.getSecretValue({ SecretId: secretName }).promise();
  return JSON.parse(data.SecretString);
}

// HashiCorp Vault
const vault = require('node-vault')({
  endpoint: process.env.VAULT_ADDR,
  token: process.env.VAULT_TOKEN
});

async function getVaultSecret(path) {
  const result = await vault.read(path);
  return result.data;
}
```

## API Security

### Rate Limiting
```javascript
const rateLimit = require('express-rate-limit');

// General rate limit
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,
  message: 'Too many requests from this IP'
});

// Stricter limit for auth endpoints
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  skipSuccessfulRequests: true
});

app.use('/api/', limiter);
app.use('/api/auth/', authLimiter);
```

### CSRF Protection
```javascript
const csrf = require('csurf');
const csrfProtection = csrf({ cookie: true });

app.get('/form', csrfProtection, (req, res) => {
  res.render('form', { csrfToken: req.csrfToken() });
});

app.post('/submit', csrfProtection, (req, res) => {
  // Process form
});
```

### API Key Authentication
```javascript
async function validateAPIKey(req, res, next) {
  const apiKey = req.header('X-API-Key');
  
  if (!apiKey) {
    return res.status(401).json({ error: 'API key required' });
  }
  
  // Hash the API key before lookup
  const hashedKey = crypto.createHash('sha256').update(apiKey).digest('hex');
  const client = await findClientByHashedKey(hashedKey);
  
  if (!client || !client.isActive) {
    logSecurityEvent('INVALID_API_KEY', { key: apiKey.substring(0, 8), ip: req.ip });
    return res.status(401).json({ error: 'Invalid API key' });
  }
  
  req.client = client;
  next();
}
```

## Security Checklist

### Authentication
- [ ] Passwords hashed with bcrypt/argon2 (min 12 rounds)
- [ ] Account lockout after failed attempts
- [ ] Password complexity requirements enforced
- [ ] Secure password reset flow (time-limited tokens)
- [ ] MFA available for sensitive operations

### Authorization
- [ ] Role-based access control (RBAC) implemented
- [ ] Principle of least privilege applied
- [ ] Authorization checks on every request
- [ ] No direct object references (use UUIDs)

### Data Protection
- [ ] Sensitive data encrypted at rest
- [ ] HTTPS enforced (TLS 1.3+)
- [ ] Secure cookie flags (httpOnly, secure, sameSite)
- [ ] No sensitive data in logs
- [ ] PII/PHI properly handled per regulations

### Input/Output
- [ ] All inputs validated and sanitized
- [ ] Output encoded/escaped properly
- [ ] File uploads restricted (type, size)
- [ ] SQL queries parameterized
- [ ] No eval() or dynamic code execution

### Infrastructure
- [ ] Security headers configured (Helmet.js)
- [ ] CORS properly configured
- [ ] Rate limiting implemented
- [ ] Dependencies regularly updated
- [ ] Security scanning in CI/CD

### Monitoring
- [ ] Security events logged
- [ ] Failed login attempts tracked
- [ ] Anomaly detection configured
- [ ] Incident response plan exists
- [ ] Regular security audits scheduled

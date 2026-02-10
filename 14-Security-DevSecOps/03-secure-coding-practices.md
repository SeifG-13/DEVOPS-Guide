# Secure Coding Practices

## Overview

Secure coding is the practice of writing code that is resistant to vulnerabilities and attacks.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SECURE CODING PRINCIPLES                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Core Principles:                                                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 1. Never trust user input                                   │   │
│  │ 2. Defense in depth                                         │   │
│  │ 3. Principle of least privilege                             │   │
│  │ 4. Fail securely                                            │   │
│  │ 5. Keep security simple                                     │   │
│  │ 6. Avoid security by obscurity                              │   │
│  │ 7. Fix security issues correctly                            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## OWASP Top 10

### 2021 OWASP Top 10

```
┌─────────────────────────────────────────────────────────────────────┐
│                    OWASP TOP 10 (2021)                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  A01: Broken Access Control                                         │
│  ─────────────────────────                                          │
│  • Bypassing access controls                                       │
│  • IDOR (Insecure Direct Object References)                        │
│  • Missing function-level access control                           │
│                                                                     │
│  A02: Cryptographic Failures                                        │
│  ───────────────────────────                                        │
│  • Weak encryption algorithms                                      │
│  • Improper key management                                         │
│  • Data transmitted in clear text                                  │
│                                                                     │
│  A03: Injection                                                     │
│  ──────────────                                                     │
│  • SQL, NoSQL, OS command injection                                │
│  • LDAP, XPath injection                                           │
│  • Expression language injection                                   │
│                                                                     │
│  A04: Insecure Design                                               │
│  ────────────────────                                               │
│  • Missing security controls in design                             │
│  • Insufficient threat modeling                                    │
│  • Insecure business logic                                         │
│                                                                     │
│  A05: Security Misconfiguration                                     │
│  ──────────────────────────────                                     │
│  • Default credentials                                             │
│  • Unnecessary features enabled                                    │
│  • Missing security headers                                        │
│                                                                     │
│  A06: Vulnerable Components                                         │
│  ──────────────────────────                                         │
│  • Outdated dependencies                                           │
│  • Known vulnerable libraries                                      │
│  • Unmaintained components                                         │
│                                                                     │
│  A07: Authentication Failures                                       │
│  ────────────────────────────                                       │
│  • Weak passwords allowed                                          │
│  • Missing brute-force protection                                  │
│  • Improper session management                                     │
│                                                                     │
│  A08: Software and Data Integrity Failures                          │
│  ─────────────────────────────────────────                          │
│  • Untrusted CI/CD pipelines                                       │
│  • Auto-update without verification                                │
│  • Insecure deserialization                                        │
│                                                                     │
│  A09: Security Logging and Monitoring Failures                      │
│  ─────────────────────────────────────────────                      │
│  • Insufficient logging                                            │
│  • Logs not monitored                                              │
│  • No alerting on attacks                                          │
│                                                                     │
│  A10: Server-Side Request Forgery (SSRF)                           │
│  ───────────────────────────────────────                            │
│  • Fetching URLs without validation                                │
│  • Internal service access                                         │
│  • Cloud metadata exposure                                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Input Validation

### Validation Strategies

```
┌─────────────────────────────────────────────────────────────────────┐
│                    INPUT VALIDATION                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Rule: NEVER trust user input                                       │
│                                                                     │
│  Validation Types:                                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Allowlist (Whitelist) - Preferred                           │   │
│  │ • Define what IS allowed                                    │   │
│  │ • Reject everything else                                    │   │
│  │ • Example: Only allow [a-zA-Z0-9]                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Blocklist (Blacklist) - Less Secure                         │   │
│  │ • Define what is NOT allowed                                │   │
│  │ • Allow everything else                                     │   │
│  │ • Easy to bypass with encoding                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Validation Checks:                                                 │
│  • Type (string, integer, boolean)                                 │
│  • Length (min/max)                                                │
│  • Format (regex pattern)                                          │
│  • Range (numeric bounds)                                          │
│  • Business rules                                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Code Examples

```python
# Python - Input Validation

# BAD - No validation
def get_user(user_id):
    query = f"SELECT * FROM users WHERE id = {user_id}"
    return db.execute(query)

# GOOD - Parameterized query + type validation
def get_user(user_id):
    # Validate type
    if not isinstance(user_id, int):
        raise ValueError("User ID must be an integer")

    # Validate range
    if user_id < 1:
        raise ValueError("User ID must be positive")

    # Parameterized query
    query = "SELECT * FROM users WHERE id = %s"
    return db.execute(query, (user_id,))
```

```javascript
// JavaScript - Input Validation with Joi
const Joi = require('joi');

const userSchema = Joi.object({
    username: Joi.string()
        .alphanum()
        .min(3)
        .max(30)
        .required(),

    email: Joi.string()
        .email()
        .required(),

    password: Joi.string()
        .min(12)
        .pattern(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])/)
        .required(),

    age: Joi.number()
        .integer()
        .min(18)
        .max(120)
});

// Validate input
const { error, value } = userSchema.validate(req.body);
if (error) {
    return res.status(400).json({ error: error.details });
}
```

---

## Output Encoding

### Preventing Injection

```
┌─────────────────────────────────────────────────────────────────────┐
│                    OUTPUT ENCODING                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Context-Specific Encoding:                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ HTML Context      │ HTML entity encoding                    │   │
│  │                   │ < → &lt;  > → &gt;  & → &amp;           │   │
│  ├───────────────────┼─────────────────────────────────────────┤   │
│  │ JavaScript Context│ JavaScript escaping                     │   │
│  │                   │ ' → \'  " → \"  \ → \\                  │   │
│  ├───────────────────┼─────────────────────────────────────────┤   │
│  │ URL Context       │ URL encoding                            │   │
│  │                   │ space → %20  < → %3C                    │   │
│  ├───────────────────┼─────────────────────────────────────────┤   │
│  │ CSS Context       │ CSS escaping                            │   │
│  │                   │ Escape non-alphanumeric                 │   │
│  ├───────────────────┼─────────────────────────────────────────┤   │
│  │ SQL Context       │ Parameterized queries                   │   │
│  │                   │ NEVER concatenate                       │   │
│  └───────────────────┴─────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### XSS Prevention

```html
<!-- BAD - Direct output (vulnerable to XSS) -->
<div>Welcome, <%= username %></div>

<!-- GOOD - HTML encoded output -->
<div>Welcome, <%= htmlEncode(username) %></div>

<!-- GOOD - Using framework auto-escaping (React) -->
<div>Welcome, {username}</div>

<!-- CAREFUL - dangerouslySetInnerHTML bypasses escaping -->
<div dangerouslySetInnerHTML={{__html: content}} />  <!-- DANGEROUS! -->
```

```python
# Python Flask - Auto-escaping
from markupsafe import escape

@app.route('/user/<username>')
def show_user(username):
    # Auto-escaped in templates
    return render_template('user.html', username=username)

# Manual escaping when needed
safe_username = escape(username)
```

---

## SQL Injection Prevention

### Parameterized Queries

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SQL INJECTION PREVENTION                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  NEVER concatenate user input into SQL queries!                    │
│                                                                     │
│  BAD:                                                               │
│  query = "SELECT * FROM users WHERE name = '" + name + "'"         │
│  Input: ' OR '1'='1                                                │
│  Result: SELECT * FROM users WHERE name = '' OR '1'='1'            │
│                                                                     │
│  GOOD - Parameterized Query:                                        │
│  query = "SELECT * FROM users WHERE name = ?"                      │
│  db.execute(query, [name])                                         │
│  Input is treated as DATA, not CODE                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Examples by Language

```python
# Python - SQLAlchemy ORM (Recommended)
user = session.query(User).filter(User.name == username).first()

# Python - Raw SQL with parameters
cursor.execute("SELECT * FROM users WHERE name = %s", (username,))

# Python - NEVER DO THIS
cursor.execute(f"SELECT * FROM users WHERE name = '{username}'")  # VULNERABLE!
```

```javascript
// Node.js - Prepared statements
const query = 'SELECT * FROM users WHERE id = $1';
const result = await pool.query(query, [userId]);

// Using ORM (Sequelize)
const user = await User.findOne({ where: { id: userId } });
```

```java
// Java - PreparedStatement
String query = "SELECT * FROM users WHERE id = ?";
PreparedStatement stmt = connection.prepareStatement(query);
stmt.setInt(1, userId);
ResultSet rs = stmt.executeQuery();
```

---

## Authentication Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AUTHENTICATION                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Password Requirements:                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Minimum 12 characters                                     │   │
│  │ • Check against breached password lists                     │   │
│  │ • Allow all characters (including spaces)                   │   │
│  │ • No composition rules (uppercase, numbers, etc.)           │   │
│  │ • Use password strength meter                               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Password Storage:                                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Use Argon2id (preferred) or bcrypt                        │   │
│  │ • NEVER store plain text                                    │   │
│  │ • NEVER use MD5, SHA1, SHA256 alone                         │   │
│  │ • Use unique salt per password                              │   │
│  │ • Use appropriate work factor                               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Multi-Factor Authentication:                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Implement MFA for sensitive operations                    │   │
│  │ • Support TOTP (Google Authenticator)                       │   │
│  │ • Support WebAuthn/FIDO2 (hardware keys)                    │   │
│  │ • Avoid SMS-based MFA (SIM swapping risk)                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Password Hashing Example

```python
# Python - Using Argon2
from argon2 import PasswordHasher

ph = PasswordHasher()

# Hash password
hashed = ph.hash(password)

# Verify password
try:
    ph.verify(hashed, password)
except VerifyMismatchError:
    print("Invalid password")

# Rehash if needed (when parameters change)
if ph.check_needs_rehash(hashed):
    new_hash = ph.hash(password)
```

```javascript
// Node.js - Using bcrypt
const bcrypt = require('bcrypt');
const saltRounds = 12;

// Hash password
const hash = await bcrypt.hash(password, saltRounds);

// Verify password
const match = await bcrypt.compare(password, hash);
```

---

## Session Management

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SESSION SECURITY                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Session Token Requirements:                                        │
│  • Cryptographically random (128+ bits entropy)                    │
│  • Regenerate after login                                          │
│  • Invalidate on logout                                            │
│  • Set expiration time                                             │
│                                                                     │
│  Cookie Attributes:                                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Secure    │ Only send over HTTPS                            │   │
│  │ HttpOnly  │ Not accessible via JavaScript                   │   │
│  │ SameSite  │ CSRF protection (Strict or Lax)                 │   │
│  │ Path      │ Limit cookie scope                              │   │
│  │ Max-Age   │ Set expiration                                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Example Cookie:                                                    │
│  Set-Cookie: session=abc123; Secure; HttpOnly; SameSite=Strict;   │
│              Path=/; Max-Age=3600                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Security Headers

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HTTP SECURITY HEADERS                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Content-Security-Policy (CSP):                                     │
│  Content-Security-Policy: default-src 'self'; script-src 'self'   │
│                                                                     │
│  X-Content-Type-Options:                                            │
│  X-Content-Type-Options: nosniff                                   │
│                                                                     │
│  X-Frame-Options:                                                   │
│  X-Frame-Options: DENY                                             │
│                                                                     │
│  Strict-Transport-Security (HSTS):                                  │
│  Strict-Transport-Security: max-age=31536000; includeSubDomains   │
│                                                                     │
│  X-XSS-Protection (legacy):                                         │
│  X-XSS-Protection: 1; mode=block                                   │
│                                                                     │
│  Referrer-Policy:                                                   │
│  Referrer-Policy: strict-origin-when-cross-origin                  │
│                                                                     │
│  Permissions-Policy:                                                │
│  Permissions-Policy: geolocation=(), microphone=()                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Error Handling

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SECURE ERROR HANDLING                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  BAD - Information Disclosure:                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Error: Invalid SQL syntax near 'users' at line 1            │   │
│  │ Stack trace: /app/db/query.py line 42                       │   │
│  │ Database: PostgreSQL 14.2                                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  GOOD - Generic Error:                                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ { "error": "An error occurred. Please try again." }         │   │
│  │ { "error_id": "abc123" }  // For support reference          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Best Practices:                                                    │
│  • Return generic messages to users                                │
│  • Log detailed errors internally                                  │
│  • Include error ID for correlation                                │
│  • Different handling for dev vs production                        │
│  • Never expose stack traces in production                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Key Practices:                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Input Validation    │ Allowlist, validate type/length/format│   │
│  │ Output Encoding     │ Context-specific encoding             │   │
│  │ SQL Injection       │ Parameterized queries only            │   │
│  │ XSS Prevention      │ Encode output, use CSP                │   │
│  │ Authentication      │ Strong hashing, MFA                   │   │
│  │ Session Management  │ Secure cookies, regenerate tokens     │   │
│  │ Error Handling      │ Generic messages, log details         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Remember:                                                          │
│  • Never trust user input                                          │
│  • Use established libraries                                       │
│  • Keep dependencies updated                                       │
│  • Follow OWASP guidelines                                         │
│  • Code review for security                                        │
│                                                                     │
│  Next: Learn about SAST (Static Analysis)                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

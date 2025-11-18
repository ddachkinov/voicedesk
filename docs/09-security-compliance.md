# Security & Compliance

## Overview

The Virtual Receptionist system handles sensitive personal data including names, phone numbers, emails, and payment information (future). This document outlines security measures and compliance requirements for Bulgarian and EU regulations.

---

## Regulatory Compliance

### GDPR (General Data Protection Regulation)

Bulgaria, as an EU member state, must comply with GDPR requirements.

#### Key GDPR Principles

**1. Lawfulness, Fairness, and Transparency**
- Clear privacy policy in Bulgarian
- Explicit consent for data collection
- Transparent data processing practices

**2. Purpose Limitation**
- Data collected only for booking purposes
- No secondary use without consent
- Clear purpose statement

**3. Data Minimization**
- Collect only necessary data
- Don't request unnecessary information
- Regular data cleanup

**4. Accuracy**
- Allow guests to correct their data
- Regular data validation
- Update records promptly

**5. Storage Limitation**
- Retain data only as long as needed
- Default retention: 7 years for bookings (tax requirements)
- Call recordings: 90 days
- Analytics data: 2 years

**6. Integrity and Confidentiality**
- Encryption at rest and in transit
- Access controls
- Regular security audits

**7. Accountability**
- Document compliance measures
- Maintain processing records
- Designate Data Protection Officer (if required)

---

### Bulgarian Law on Personal Data Protection

**Закон за защита на личните данни (ЗЗЛД)**

**Requirements**:
- Register with Commission for Personal Data Protection (КЗЛД)
- Appoint Data Controller
- Implement technical and organizational measures
- Report data breaches within 72 hours

**Registration**:
```
Entity: Hotel (Data Controller)
Processor: Virtual Receptionist System
Purpose: Hotel booking management
Data Types: Names, phones, emails, booking details
Retention: 7 years (bookings), 90 days (calls)
```

---

## Data Protection Measures

### Data Classification

| Data Type | Classification | Encryption | Retention | Access Level |
|-----------|---------------|------------|-----------|--------------|
| Guest names | PII | At rest + transit | 7 years | Staff + System |
| Phone numbers | PII | At rest + transit | 7 years | Staff + System |
| Email addresses | PII | At rest + transit | 7 years | Staff + System |
| Call recordings | Sensitive | At rest + transit | 90 days | Authorized staff |
| Payment data | Highly Sensitive | Tokenized | Per PCI DSS | Payment processor |
| Analytics (aggregated) | Non-PII | Transit only | 2 years | Staff + System |

### Encryption

**At Rest**:
- **Database**: AES-256 encryption
- **Backups**: AES-256 encryption
- **Call Recordings**: AES-256 encryption in S3

```typescript
// Database encryption (PostgreSQL)
CREATE EXTENSION pgcrypto;

CREATE TABLE guests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    full_name VARCHAR(255) NOT NULL,
    phone_encrypted BYTEA,  -- Encrypted phone
    email_encrypted BYTEA,  -- Encrypted email
    encryption_key_id INTEGER NOT NULL
);

-- Encrypt on insert
INSERT INTO guests (full_name, phone_encrypted, email_encrypted)
VALUES (
    'Петър Георгиев',
    pgp_sym_encrypt('0888123456', 'encryption_key'),
    pgp_sym_encrypt('peter@example.com', 'encryption_key')
);

-- Decrypt on read
SELECT
    full_name,
    pgp_sym_decrypt(phone_encrypted, 'encryption_key') as phone,
    pgp_sym_decrypt(email_encrypted, 'encryption_key') as email
FROM guests;
```

**In Transit**:
- **HTTPS/TLS 1.3**: All web traffic
- **WSS (WebSocket Secure)**: Real-time updates
- **Encrypted API calls**: To all external services

```typescript
// Enforce HTTPS
app.use((req, res, next) => {
  if (!req.secure && process.env.NODE_ENV === 'production') {
    return res.redirect(301, `https://${req.headers.host}${req.url}`)
  }
  next()
})
```

### Access Control

**Role-Based Access Control (RBAC)**:

| Role | Access Level | Permissions |
|------|-------------|-------------|
| **Super Admin** | Full access | All operations, multi-property |
| **Org Admin** | Organization | Manage properties, users, view all data |
| **Property Manager** | Property | Manage bookings, calendar, view analytics |
| **Receptionist** | Limited | Create bookings, view calls, basic calendar |
| **Viewer** | Read-only | View bookings, calls (no edit) |
| **AI System** | Automated | Create bookings, read calendar (API key) |

**Implementation**:
```typescript
// Middleware for role checking
function requireRole(roles: UserRole[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user || !roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' })
    }
    next()
  }
}

// Usage
app.delete('/bookings/:id', requireRole(['property_manager', 'org_admin']), deleteBooking)
```

### Authentication

**JWT-Based Authentication**:
```typescript
// Token generation
function generateTokens(user: User) {
  const accessToken = jwt.sign(
    { userId: user.id, role: user.role },
    process.env.JWT_SECRET,
    { expiresIn: '1h' }
  )

  const refreshToken = jwt.sign(
    { userId: user.id },
    process.env.JWT_REFRESH_SECRET,
    { expiresIn: '7d' }
  )

  return { accessToken, refreshToken }
}
```

**Password Security**:
- Bcrypt hashing (cost factor: 12)
- Minimum password requirements: 12 characters, mixed case, numbers
- Password reset via email with expiring tokens
- Account lockout after 5 failed attempts

```typescript
import bcrypt from 'bcrypt'

async function hashPassword(password: string): Promise<string> {
  const saltRounds = 12
  return bcrypt.hash(password, saltRounds)
}

async function verifyPassword(password: string, hash: string): Promise<boolean> {
  return bcrypt.compare(password, hash)
}
```

---

## Data Privacy

### Privacy Policy Requirements

**Must Include**:
1. What data we collect
2. Why we collect it
3. How long we keep it
4. Who we share it with
5. User rights (access, deletion, portability)
6. How to exercise rights
7. Contact information

**Example Privacy Policy (excerpt in Bulgarian)**:
```
ПОЛИТИКА ЗА ПОВЕРИТЕЛНОСТ

1. СЪБИРАНЕ НА ДАННИ
Събираме следните лични данни за целите на резервации:
• Име и фамилия
• Телефонен номер
• Имейл адрес
• Дати на престой
• Специални пожелания

2. ЦЕЛИ НА ОБРАБОТКАТА
Вашите данни се използват единствено за:
• Обработка на резервации
• Комуникация относно вашия престой
• Подобряване на нашите услуги

3. СРОК НА СЪХРАНЕНИЕ
• Данни за резервации: 7 години (данъчни изисквания)
• Записи на обаждания: 90 дни
• Аналитични данни (анонимизирани): 2 години

4. ВАШИТЕ ПРАВА
Имате право на:
• Достъп до вашите данни
• Коригиране на неточни данни
• Изтриване на данни ("правото да бъдеш забравен")
• Преносимост на данни
• Възражение срещу обработката
```

### Consent Management

**Consent Requirements**:
- Explicit consent for call recording
- Opt-in for marketing communications
- Separate consent for data sharing

**Implementation**:
```typescript
interface ConsentRecord {
  userId: string
  callRecordingConsent: boolean
  marketingConsent: boolean
  dataProcessingConsent: boolean
  consentDate: Date
  ipAddress: string
}

// AI announces recording at call start
const greeting = `
  Добре дошли в ${hotelName}. Това обаждане може да бъде записано за
  качество и обучение. Продължавайки разговора, вие се съгласявате с
  това. Как мога да ви помогна?
`
```

### Data Subject Rights

**Right to Access**:
```typescript
// API endpoint for data export
app.get('/api/users/me/data-export', authenticate, async (req, res) => {
  const user = req.user

  // Gather all user data
  const userData = {
    profile: await getUserProfile(user.id),
    bookings: await getUserBookings(user.id),
    calls: await getUserCalls(user.id)
  }

  // Return as JSON
  res.json(userData)
})
```

**Right to Erasure ("Right to be Forgotten")**:
```typescript
// API endpoint for account deletion
app.delete('/api/users/me', authenticate, async (req, res) => {
  const user = req.user

  // Check if user has active bookings
  const activeBookings = await getActiveBookings(user.id)
  if (activeBookings.length > 0) {
    return res.status(400).json({
      error: 'Cannot delete account with active bookings'
    })
  }

  // Anonymize instead of delete (for audit trail)
  await anonymizeUser(user.id)

  res.status(204).send()
})

async function anonymizeUser(userId: string) {
  await db.query(`
    UPDATE guests
    SET
      full_name = 'Анонимен гост',
      phone_encrypted = NULL,
      email_encrypted = NULL,
      metadata = '{"anonymized": true, "date": "${new Date().toISOString()}"}'
    WHERE id = $1
  `, [userId])
}
```

---

## Security Measures

### Application Security

**OWASP Top 10 Mitigation**:

1. **Injection**: Parameterized queries, input validation
```typescript
// BAD
const query = `SELECT * FROM users WHERE email = '${email}'`  // SQL injection risk

// GOOD
const query = `SELECT * FROM users WHERE email = $1`
const result = await db.query(query, [email])
```

2. **Broken Authentication**: Strong passwords, JWT, rate limiting
```typescript
// Rate limiting
import rateLimit from 'express-rate-limit'

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // 5 attempts
  message: 'Твърде много опити за вход. Опитайте отново след 15 минути.'
})

app.post('/auth/login', loginLimiter, loginHandler)
```

3. **Sensitive Data Exposure**: Encryption, HTTPS
4. **XML External Entities**: Disable XML parsing or sanitize
5. **Broken Access Control**: RBAC, authorization checks
6. **Security Misconfiguration**: Hardened servers, security headers
```typescript
import helmet from 'helmet'

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", 'https:']
    }
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  }
}))
```

7. **XSS**: Input sanitization, Content Security Policy
8. **Insecure Deserialization**: Validate input, use safe parsers
9. **Using Components with Known Vulnerabilities**: Regular dependency updates
10. **Insufficient Logging**: Comprehensive audit logs

### API Security

**API Key Management**:
```typescript
// Generate API key
import crypto from 'crypto'

function generateApiKey(): string {
  return crypto.randomBytes(32).toString('hex')
}

// Store hashed
const apiKeyHash = crypto
  .createHash('sha256')
  .update(apiKey)
  .digest('hex')

// Validate
function validateApiKey(providedKey: string, storedHash: string): boolean {
  const providedHash = crypto
    .createHash('sha256')
    .update(providedKey)
    .digest('hex')

  return crypto.timingSafeEqual(
    Buffer.from(providedHash),
    Buffer.from(storedHash)
  )
}
```

**Rate Limiting**:
```typescript
// API rate limiting
const apiLimiter = rateLimit({
  windowMs: 60 * 60 * 1000, // 1 hour
  max: 1000, // 1000 requests per hour
  keyGenerator: (req) => req.headers['x-api-key'] || req.ip,
  handler: (req, res) => {
    res.status(429).json({
      error: 'Rate limit exceeded',
      retryAfter: req.rateLimit.resetTime
    })
  }
})

app.use('/api', apiLimiter)
```

### Infrastructure Security

**Network Security**:
- Private subnets for databases
- Security groups (firewall rules)
- VPN for administrative access
- DDoS protection (CloudFlare)

**DigitalOcean Security Configuration**:
```yaml
# Firewall rules
inbound:
  - protocol: tcp
    ports: 443
    sources:
      addresses: ["0.0.0.0/0"]  # HTTPS from anywhere
  - protocol: tcp
    ports: 22
    sources:
      addresses: ["10.0.0.0/8"]  # SSH from VPN only

outbound:
  - protocol: tcp
    ports: all
    destinations:
      addresses: ["0.0.0.0/0"]
```

---

## Audit Logging

### What to Log

```typescript
interface AuditLog {
  id: string
  timestamp: Date
  userId: string | null
  action: string
  resource: string
  resourceId: string
  changes: any
  ipAddress: string
  userAgent: string
  success: boolean
  errorMessage?: string
}
```

**Logged Actions**:
- User authentication (login, logout, failed attempts)
- Data access (view, export)
- Data modification (create, update, delete)
- Permission changes
- Configuration changes
- API calls
- System errors

**Example**:
```typescript
async function auditLog(log: AuditLog) {
  await db.query(`
    INSERT INTO audit_logs (
      user_id, action, resource, resource_id,
      changes, ip_address, user_agent, success
    ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
  `, [
    log.userId,
    log.action,
    log.resource,
    log.resourceId,
    JSON.stringify(log.changes),
    log.ipAddress,
    log.userAgent,
    log.success
  ])
}

// Usage
await auditLog({
  userId: user.id,
  action: 'update',
  resource: 'booking',
  resourceId: booking.id,
  changes: { status: 'cancelled' },
  ipAddress: req.ip,
  userAgent: req.get('user-agent'),
  success: true
})
```

### Log Retention

- **Security logs**: 1 year
- **Audit logs**: 7 years
- **Application logs**: 30 days
- **Access logs**: 90 days

---

## Data Breach Response

### Incident Response Plan

**Detection → Containment → Investigation → Notification → Recovery**

**1. Detection** (Within 1 hour):
- Automated monitoring alerts
- Manual reports from staff
- Security audit findings

**2. Containment** (Within 2 hours):
- Isolate affected systems
- Revoke compromised credentials
- Block malicious IPs

**3. Investigation** (Within 12 hours):
- Determine scope of breach
- Identify root cause
- Document all findings

**4. Notification** (Within 72 hours):
- Notify КЗЛД (Bulgarian DPA)
- Notify affected users
- Public disclosure if required

**5. Recovery** (Within 1 week):
- Patch vulnerabilities
- Restore from backups
- Implement additional safeguards

**Notification Template (Bulgarian)**:
```
УВЕДОМЛЕНИЕ ЗА НАРУШЕНИЕ НА СИГУРНОСТТА НА ДАННИТЕ

Уважаеми/а [Име],

С съжаление ви уведомяваме, че на [дата] е установено нарушение на
сигурността на личните данни в нашата система.

ЗАСЕГНАТИ ДАННИ:
• [Списък на засегнатите данни]

ПРЕДПРИЕТИ ДЕЙСТВИЯ:
• [Описание на действията]

ПРЕПОРЪКИ ЗА ВАС:
• [Препоръки за защита]

За допълнителна информация: [контакт]

С уважение,
[Име на хотела]
```

---

## Third-Party Security

### Vendor Security Assessment

Before integrating any third-party service, assess:

| Criteria | Requirement |
|----------|-------------|
| **GDPR Compliance** | Must be GDPR compliant |
| **Data Processing Agreement** | Must sign DPA |
| **Security Certifications** | ISO 27001, SOC 2, or equivalent |
| **Data Location** | EU or adequate protection |
| **Encryption** | At rest and in transit |
| **Access Controls** | Role-based access |
| **Audit Logs** | Comprehensive logging |
| **Breach Notification** | 24-hour notification |

**Current Vendors**:

| Vendor | Purpose | GDPR | DPA | Certifications |
|--------|---------|------|-----|----------------|
| **Twilio** | Telephony | ✅ | ✅ | SOC 2, ISO 27001 |
| **Google Cloud** | STT/TTS | ✅ | ✅ | SOC 2, ISO 27001 |
| **OpenAI** | LLM | ✅ | ✅ | SOC 2 |
| **DigitalOcean** | Infrastructure | ✅ | ✅ | SOC 2, ISO 27001 |

---

## Compliance Checklist

### GDPR Compliance

- [x] Privacy policy published
- [x] Terms of service published
- [x] Consent mechanisms implemented
- [x] Data minimization applied
- [x] Encryption at rest and in transit
- [x] Right to access implemented
- [x] Right to erasure implemented
- [x] Right to portability implemented
- [x] Data breach notification plan
- [x] DPO appointed (if required)
- [x] Processor agreements signed
- [x] Regular security audits scheduled

### Bulgarian Law Compliance

- [ ] Register with КЗЛД
- [x] Data controller identified
- [x] Technical measures documented
- [x] Organizational measures documented
- [x] 72-hour breach notification process

---

## Security Monitoring

### Monitoring Tools

**Application Performance Monitoring**:
- Sentry for error tracking
- DataDog for performance metrics

**Security Information and Event Management (SIEM)**:
- CloudFlare Analytics
- Custom log aggregation

**Alerts**:
- Failed login attempts (>5 in 5 min)
- API rate limit exceeded
- Database connection failures
- Unauthorized access attempts
- Data export requests
- High error rates
- Unusual traffic patterns

---

## Security Training

**Staff Training** (Annual):
- GDPR principles
- Password security
- Phishing awareness
- Social engineering
- Data handling procedures
- Incident reporting

**Developer Training** (Bi-annual):
- Secure coding practices
- OWASP Top 10
- Authentication/authorization
- Encryption standards
- Audit logging

---

## Regular Security Audits

### Quarterly Reviews
- Access control review
- User account audit
- API key rotation
- Dependency updates
- Log review

### Annual Audits
- Penetration testing
- Code security review
- Infrastructure audit
- Compliance assessment
- Disaster recovery test

---

## Next Steps

1. Review [Implementation Roadmap](./10-implementation-roadmap.md) for security implementation timeline
2. See [Deployment & Operations](./13-deployment-operations.md) for security operations
3. Check [Testing Strategy](./12-testing-strategy.md) for security testing approach

# Security Patterns

## Definition

Security patterns are architectural approaches designed to protect systems, data, and users from threats while maintaining functionality and performance.

## Authentication Patterns

### Single Sign-On (SSO)
```
Identity Provider (IdP)
    ↓
┌─────────────────────────────────────┐
│                                     │
│  Service A  Service B  Service C    │
│      ↓         ↓         ↓          │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐│
│  │   Auth  │ │   Auth  │ │  Auth   ││
│  │ Redirect│ │ Redirect│ │ Redirect││
│  └─────────┘ └─────────┘ └─────────┘│
│                                     │
└─────────────────────────────────────┘
```

**Benefits:**
- Single set of credentials
- Centralized user management
- Improved user experience
- Easier compliance

### Multi-Factor Authentication (MFA)
```
Authentication Flow:
Username/Password → OTP → Biometric → Success
      ↓              ↓         ↓
Factor 1      Factor 2   Factor 3
```

## Authorization Patterns

### OAuth 2.0 Authorization Code Flow
```
User → Client → Authorization Server
  ↓         ↓
Consent   Authorization Code
Token Exchange → Access Token
```

### Role-Based Access Control (RBAC)
```
User → Role → Permissions → Resource
  ↓     ↓        ↓
Check → Validate → Grant/Deny
```

### Attribute-Based Access Control (ABAC)
```
User Attributes + Resource Attributes + Environment
                    ↓
                Policy Engine
                    ↓
                Allow/Deny Decision
```

## Data Protection Patterns

### Encryption at Rest
```
Plain Text → Encryption Key → Encrypted Data
    ↓              ↓
Storage ←─────────────────┘
```

### Encryption in Transit
```
Client → TLS Handshake → Server
    ↓         ↓
Encrypted Communication
```

### Data Masking
```
Production Data:
┌─────────────────────────────────────┐
│ Name: John Doe                      │
│ Email: jo***@***.com                │
│ Phone: ***-***-1234                 │
│ SSN: ***-**-****                    │
└─────────────────────────────────────┘

Development Data:
┌─────────────────────────────────────┐
│ Name: John Doe                      │
│ Email: john@example.com             │
│ Phone: 555-123-4567                 │
│ SSN: 123-45-6789                    │
└─────────────────────────────────────┘
```

## Network Security Patterns

### Zero Trust Architecture
```
Every Request:
Verify → Authenticate → Authorize → Encrypt
    ↓          ↓           ↓
Never Trust, Always Verify
```

### API Gateway Security
```
Client → API Gateway → Microservices
    ↓         ↓
Rate Limiting → Authentication → Authorization
    ↓         ↓
Logging → Monitoring → WAF
```

### Web Application Firewall (WAF)
```
HTTP Request → WAF → Application
    ↓         ↓
Threat Detection → Filtering
    ↓         ↓SQL Injection → XSS → CSRF Protection
```

## Secure Communication Patterns

### Mutual TLS (mTLS)
```
Client Certificate ←→ Server Certificate
        ↓
Mutual Authentication
        ↓
Encrypted Communication
```

### Message Authentication
```
Message + HMAC → Verification
    ↓
Integrity Check → Authenticity Verification
```

## Input Validation Patterns

### Input Sanitization
```
User Input → Validation → Sanitization → Processing
    ↓           ↓
Type Check → Length Check → Format Check
    ↓           ↓HTML Encoding → SQL Escaping → XSS Prevention
```

### Parameterized Queries
```
Unsafe:
"SELECT * FROM users WHERE name = '" + user_input + "'"

Safe:
"SELECT * FROM users WHERE name = ?"
```

## Security Monitoring Patterns

### Intrusion Detection
```
System Events → Analysis → Alerting
    ↓          ↓
Anomaly Detection → Pattern Matching → Response
```

### Security Information and Event Management (SIEM)
```
Multiple Sources → SIEM → Correlation → Dashboard
    ↓              ↓
Logs → Events → Metrics → Alerts
```

## Real-World Examples

### AWS Security Patterns
```
Components:
- IAM for identity management
- VPC for network isolation
- Security Groups for firewall rules
- CloudTrail for auditing
- GuardDuty for threat detection
```

### Google Cloud Security Patterns
```
Components:
- Cloud Identity for SSO
- VPC Service for networking
- Cloud Armor for WAF
- Security Command Center for monitoring
- Binary Authorization for container security
```

### Netflix Security Architecture
```
Components:
- Custom authentication system
- API gateway with rate limiting
- Content protection (DRM)
- Threat detection systems
- Security monitoring and analytics
```

## Interview Tips

### Common Questions
1. "How would you secure a REST API?"
2. "What's the difference between authentication and authorization?"
3. "How does OAuth 2.0 work?"
4. "What are common security vulnerabilities?"
5. "How would you implement zero trust?"

### Answer Framework
1. **Identify Threats**: What are we protecting against?
2. **Layer Security**: Defense in depth approach
3. **Choose Patterns**: Based on requirements and constraints
4. **Implement Controls**: Technical and procedural measures
5. **Monitor and Respond**: Continuous security monitoring

### Key Points to Emphasize
- Security is a process, not a product
- Principle of least privilege
- Defense in depth strategy
- Regular security assessments
- Incident response planning

## Practice Problems

1. **Design secure authentication system**
2. **Implement API security**
3. **Design data protection strategy**
4. **Create security monitoring system**
5. **Design zero trust architecture**

## Further Reading

- **Books**: "The Web Application Hacker's Handbook" by Dafydd Stuttard
- **Standards**: OWASP Top 10, NIST Cybersecurity Framework
- **Certifications**: CISSP, CEH, Security+

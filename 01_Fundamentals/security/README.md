# Security Fundamentals

## Definition

Security in system design encompasses the principles, practices, and patterns used to protect systems, data, and users from unauthorized access, attacks, and data breaches while ensuring confidentiality, integrity, and availability.

## Security Principles (CIA Triad + Beyond)

### Confidentiality
- **Definition**: Information is accessible only to authorized users
- **Implementation**: Encryption, access control, authentication
- **Example**: Encrypting sensitive user data

### Integrity
- **Definition**: Information is accurate and has not been tampered with
- **Implementation**: Hashing, digital signatures, checksums
- **Example**: Verifying API request signatures

### Availability
- **Definition**: Systems and data are accessible when needed
- **Implementation**: Redundancy, DDoS protection, load balancing
- **Example**: Designing highly available services

### Additional Principles

#### Non-Repudiation
- **Definition**: Actions cannot be denied by the actor
- **Implementation**: Digital signatures, audit logs, blockchain
- **Example**: Signed financial transactions

#### Accountability
- **Definition**: Actions can be traced to specific users
- **Implementation**: Audit trails, session tracking, logging
- **Example**: User activity monitoring

#### Privacy by Design
- **Definition**: Privacy considerations built into system design
- **Implementation**: Data minimization, purpose limitation, consent management
- **Example**: GDPR-compliant data processing

### Zero-Trust Architecture
```
Traditional: Trust internal network
Zero-Trust: Never trust, always verify

Perimeter Security → Zero-Trust Security
     ↓                      ↓
Trust by Location      Trust by Identity + Context
```

**Core Principles:**
- **Verify Identity**: Authenticate all users and devices
- **Least Privilege**: Grant minimum required access
- **Micro-Segmentation**: Divide network into small segments
- **Continuous Monitoring**: Monitor all traffic and behavior
- **Assume Breach**: Design for compromised credentials

**Implementation:**
- **Identity Providers**: OAuth, SAML, OpenID Connect
- **Policy Engines**: Attribute-based access control (ABAC)
- **Service Mesh**: mTLS between services
- **Network Segmentation**: VPCs, security groups, network ACLs

## Authentication and Authorization

### Authentication (Who are you?)
```
User → Credentials → System → Verification → Access Granted/Denied
```

**Authentication Factors:**
1. **Something you know**: Password, PIN, security questions
2. **Something you have**: Phone, token, smart card
3. **Something you are**: Biometrics (fingerprint, face)

**Multi-Factor Authentication (MFA):**
```python
class AuthenticationService:
    def authenticate(self, username, password, token=None):
        # Factor 1: Password
        user = self.database.get_user(username)
        if not self.verify_password(password, user.password_hash):
            return False
        
        # Factor 2: Token (optional)
        if token and not self.verify_token(token, user.mfa_secret):
            return False
        
        return True
    
    def verify_password(self, input_password, stored_hash):
        return bcrypt.checkpw(input_password.encode(), stored_hash.encode())
    
    def verify_token(self, input_token, mfa_secret):
        return pyotp.TOTP(mfa_secret).verify(input_token)
```

### Authorization (What can you do?)
```
Authenticated User → Permissions → Resource Access Decision
```

**Authorization Models:**

#### 1. Role-Based Access Control (RBAC)
```python
class RBAC:
    def __init__(self):
        self.roles = {
            'admin': ['read', 'write', 'delete', 'manage_users'],
            'editor': ['read', 'write'],
            'viewer': ['read']
        }
    
    def can_access(self, user_role, permission):
        return permission in self.roles.get(user_role, [])
```

#### 2. Attribute-Based Access Control (ABAC)
```python
class ABAC:
    def can_access(self, user, resource, action):
        policies = [
            self.department_policy(user, resource),
            self.time_policy(user),
            self.location_policy(user),
            self.sensitivity_policy(resource, action)
        ]
        return all(policies)
    
    def department_policy(self, user, resource):
        return user.department == resource.owner_department
    
    def time_policy(self, user):
        return 9 <= datetime.now().hour <= 17  # Business hours
    
    def location_policy(self, user):
        return user.location in ['US', 'CA', 'UK']
    
    def sensitivity_policy(self, resource, action):
        if resource.sensitivity == 'high':
            return action in ['read'] and user.clearance >= 'secret'
        return True
```

## Cryptography

### Symmetric Encryption
```
Plaintext → [Encrypt with Secret Key] → Ciphertext
Ciphertext → [Decrypt with Secret Key] → Plaintext
```

**Use Cases:**
- Data at rest encryption
- Database field encryption
- Secure communication

**Implementation (AES):**
```python
from cryptography.fernet import Fernet

class SymmetricEncryption:
    def __init__(self, key):
        self.cipher = Fernet(key)
    
    def encrypt(self, data):
        return self.cipher.encrypt(data.encode())
    
    def decrypt(self, encrypted_data):
        return self.cipher.decrypt(encrypted_data).decode()
    
    @staticmethod
    def generate_key():
        return Fernet.generate_key()
```

### Asymmetric Encryption
```
Plaintext → [Encrypt with Public Key] → Ciphertext
Ciphertext → [Decrypt with Private Key] → Plaintext
```

**Use Cases:**
- Key exchange
- Digital signatures
- Secure email

**Implementation (RSA):**
```python
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import rsa, padding

class AsymmetricEncryption:
    def __init__(self):
        self.private_key = rsa.generate_private_key(
            public_exponent=65537,
            key_size=2048
        )
        self.public_key = self.private_key.public_key()
    
    def encrypt(self, data):
        encrypted = self.public_key.encrypt(
            data.encode(),
            padding.OAEP(
                mgf=padding.MGF1(algorithm=hashes.SHA256()),
                algorithm=hashes.SHA256(),
                label=None
            )
        )
        return encrypted
    
    def decrypt(self, encrypted_data):
        decrypted = self.private_key.decrypt(
            encrypted_data,
            padding.OAEP(
                mgf=padding.MGF1(algorithm=hashes.SHA256()),
                algorithm=hashes.SHA256(),
                label=None
            )
        )
        return decrypted.decode()
    
    def sign(self, data):
        signature = self.private_key.sign(
            data.encode(),
            padding.PSS(
                mgf=padding.MGF1(hashes.SHA256()),
                salt_length=padding.PSS.MAX_LENGTH
            ),
            hashes.SHA256()
        )
        return signature
    
    def verify(self, data, signature):
        try:
            self.public_key.verify(
                signature,
                data.encode(),
                padding.PSS(
                    mgf=padding.MGF1(hashes.SHA256()),
                    salt_length=padding.PSS.MAX_LENGTH
                ),
                hashes.SHA256()
            )
            return True
        except:
            return False
```

## API Security

### API Authentication Patterns

#### 1. API Keys
```python
class APIKeyAuth:
    def __init__(self, database):
        self.db = database
    
    def authenticate(self, api_key):
        key_info = self.db.get_api_key(api_key)
        if not key_info or not key_info['active']:
            return None
        
        # Update last used timestamp
        self.db.update_last_used(api_key)
        
        return {
            'user_id': key_info['user_id'],
            'permissions': key_info['permissions'],
            'rate_limit': key_info['rate_limit']
        }
```

#### 2. JWT (JSON Web Tokens)
```python
import jwt
from datetime import datetime, timedelta

class JWTAuth:
    def __init__(self, secret_key):
        self.secret_key = secret_key
    
    def generate_token(self, user_id, permissions, expires_in=3600):
        payload = {
            'user_id': user_id,
            'permissions': permissions,
            'exp': datetime.utcnow() + timedelta(seconds=expires_in),
            'iat': datetime.utcnow()
        }
        return jwt.encode(payload, self.secret_key, algorithm='HS256')
    
    def verify_token(self, token):
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=['HS256'])
            return payload
        except jwt.ExpiredSignatureError:
            raise Exception("Token has expired")
        except jwt.InvalidTokenError:
            raise Exception("Invalid token")
```

#### 3. OAuth 2.0 Flow
```
User → Client → Authorization Server → Resource Server
  ↓       ↓           ↓                ↓
Login  Request    Authorization    Access Token
  ↓       ↓           ↓                ↓
Grant  Redirect    Access Token    API Request
```

### API Security Best Practices

#### Rate Limiting
```python
class RateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client
    
    def is_allowed(self, client_id, limit=100, window=3600):
        key = f"rate_limit:{client_id}"
        current = self.redis.incr(key)
        
        if current == 1:
            self.redis.expire(key, window)
        
        return current <= limit
    
    def get_remaining_requests(self, client_id, limit=100, window=3600):
        key = f"rate_limit:{client_id}"
        current = int(self.redis.get(key) or 0)
        return max(0, limit - current)
```

#### Input Validation
```python
import re
from html import escape

class InputValidator:
    @staticmethod
    def validate_email(email):
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        return re.match(pattern, email) is not None
    
    @staticmethod
    def sanitize_html(input_string):
        # Escape HTML to prevent XSS
        return escape(input_string)
    
    @staticmethod
    def validate_sql_input(input_string):
        # Check for SQL injection patterns
        dangerous_patterns = [
            r"(\b(SELECT|INSERT|UPDATE|DELETE|DROP|CREATE|ALTER)\b)",
            r"(--|#|/\*|\*/)",
            r"(\bOR\b.*=.*\bOR\b)",
            r"(\bAND\b.*=.*\bAND\b)"
        ]
        
        for pattern in dangerous_patterns:
            if re.search(pattern, input_string, re.IGNORECASE):
                return False
        return True
```

## Cloud Security

### Identity and Access Management (IAM)
```
AWS IAM Example:
┌─────────────────────────────────────────────────────────┐
│                    AWS Account                         │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   Users     │  │   Roles     │  │   Groups    │     │
│  │             │  │             │  │             │     │
│  │ • IAM Users │  │ • EC2 Role  │  │ • Developers│     │
│  │ • Passwords │  │ • Lambda    │  │ • Admins    │     │
│  │ • MFA       │  │   Role      │  │             │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ Policies    │  │ Permissions │  │ Conditions │     │
│  │             │  │             │  │             │     │
│  │ • Allow S3  │  │ • s3:Get*   │  │ • IP Range  │     │
│  │ • Deny EC2  │  │ • ec2:*     │  │ • Time      │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘
```

**Best Practices:**
- **Principle of Least Privilege**: Grant minimum required permissions
- **Role-Based Access**: Use roles instead of users for applications
- **Multi-Factor Authentication**: Enable MFA for all users
- **Regular Audits**: Review and rotate access keys
- **Conditional Access**: IP restrictions, time-based access

### Virtual Private Cloud (VPC) Security
```
Secure VPC Design:
┌─────────────────────────────────────────────────────────┐
│                    Public Subnet                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   Load      │  │   NAT       │  │   Bastion   │     │
│  │ Balancer    │  │ Gateway     │  │   Host      │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│                                                         │
│                    Private Subnet                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ Application │  │ Database    │  │   Cache     │     │
│  │   Servers   │  │   Servers   │  │   Servers   │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘
```

**Security Layers:**
- **Network ACLs**: Stateless subnet-level filtering
- **Security Groups**: Stateful instance-level filtering
- **VPC Endpoints**: Private access to AWS services
- **VPC Flow Logs**: Network traffic monitoring

### Container Security

#### Docker Security Best Practices
```dockerfile
# Secure Dockerfile
FROM ubuntu:20.04

# Use non-root user
RUN useradd -m -s /bin/bash appuser

# Update packages and install security updates
RUN apt-get update && apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Copy application
COPY --chown=appuser:appuser app.py /app/app.py

# Switch to non-root user
USER appuser

# Run application
CMD ["python", "/app/app.py"]
```

#### Kubernetes Security
```
Pod Security Standards:
┌─────────────────────────────────────────────────────────┐
│                    Pod Security                        │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ Privileged  │  │   Host      │  │   Root      │     │
│  │ Containers  │  │   Access    │  │   User      │     │
│  │             │  │             │  │             │     │
│  │ ✗ Avoid     │  │ ✗ Restrict  │  │ ✗ Prevent   │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ Network     │  │   Secrets   │  │   RBAC      │     │
│  │ Policies    │  │ Management │  │             │     │
│  │             │  │             │  │             │     │
│  │ ✓ Implement │  │ ✓ Secure    │  │ ✓ Enforce   │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘
```

**Security Contexts:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: app
    image: secure-app:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
```

## DevSecOps Practices

### Security in CI/CD Pipeline
```
Development → Build → Test → Deploy
     ↓         ↓       ↓       ↓
Security     SAST    DAST   Runtime
Scanning   Scanning Scanning Security
```

**Security Gates:**
- **SAST (Static Application Security Testing)**: Code analysis
- **DAST (Dynamic Application Security Testing)**: Runtime testing
- **SCA (Software Composition Analysis)**: Dependency scanning
- **Container Scanning**: Image vulnerability assessment

### Infrastructure as Code Security
```terraform
# Secure Terraform configuration
resource "aws_s3_bucket" "secure_bucket" {
  bucket = "my-secure-bucket"
  
  # Encryption
  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
  
  # Versioning
  versioning {
    enabled = true
  }
  
  # Access logging
  logging {
    target_bucket = aws_s3_bucket.log_bucket.id
    target_prefix = "access-log/"
  }
  
  # Public access block
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### Compliance Frameworks

#### GDPR (General Data Protection Regulation)
```
GDPR Principles:
├── Lawfulness, Fairness, Transparency
├── Purpose Limitation
├── Data Minimization
├── Accuracy
├── Storage Limitation
├── Integrity and Confidentiality
├── Accountability
└── Data Subject Rights
```

**Implementation:**
- **Data Mapping**: Track all personal data
- **Consent Management**: User consent for data processing
- **Data Protection Impact Assessment**: DPIA for high-risk processing
- **Data Breach Notification**: 72-hour notification requirement

#### SOC 2 Compliance
```
SOC 2 Trust Principles:
├── Security (Protect against unauthorized access)
├── Availability (System availability)
├── Processing Integrity (System processing accuracy)
├── Confidentiality (Sensitive information protection)
└── Privacy (Personal information collection/use)
```

**Controls:**
- **Access Controls**: Multi-factor authentication, least privilege
- **Monitoring**: Continuous security monitoring and alerting
- **Incident Response**: Documented incident response procedures
- **Change Management**: Secure change management processes

## Network Security

### SSL/TLS Configuration
```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    # SSL Configuration
    ssl_certificate /path/to/certificate.crt;
    ssl_certificate_key /path/to/private.key;

    # Modern SSL Configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
    ssl_prefer_server_ciphers off;

    # HSTS
    add_header Strict-Transport-Security "max-age=63072000" always;

    # Other Security Headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
}
```

### Firewall Configuration
```bash
# Basic iptables rules
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -j DROP
```

## Data Protection

### Data Encryption at Rest
```python
class DatabaseEncryption:
    def __init__(self, encryption_key):
        self.cipher = Fernet(encryption_key)
    
    def encrypt_sensitive_field(self, data):
        if data is None:
            return None
        return self.cipher.encrypt(data.encode()).decode()
    
    def decrypt_sensitive_field(self, encrypted_data):
        if encrypted_data is None:
            return None
        return self.cipher.decrypt(encrypted_data.encode()).decode()
    
    def encrypt_user_data(self, user_data):
        sensitive_fields = ['ssn', 'credit_card', 'email']
        encrypted_data = user_data.copy()
        
        for field in sensitive_fields:
            if field in encrypted_data:
                encrypted_data[field] = self.encrypt_sensitive_field(
                    encrypted_data[field]
                )
        
        return encrypted_data
```

### Data Masking
```python
class DataMasking:
    @staticmethod
    def mask_email(email):
        if '@' not in email:
            return email
        local, domain = email.split('@', 1)
        if len(local) <= 2:
            masked_local = '*' * len(local)
        else:
            masked_local = local[0] + '*' * (len(local) - 2) + local[-1]
        return f"{masked_local}@{domain}"
    
    @staticmethod
    def mask_credit_card(card_number):
        # Show only last 4 digits
        return '*' * (len(card_number) - 4) + card_number[-4:]
    
    @staticmethod
    def mask_phone_number(phone):
        # Show only last 4 digits
        return '*' * (len(phone) - 4) + phone[-4:]
```

## Security Monitoring

### Intrusion Detection
```python
class IntrusionDetector:
    def __init__(self):
        self.suspicious_patterns = [
            r'(\.\./)+',  # Directory traversal
            r'<script.*?>.*?</script>',  # XSS
            r'(union|select|insert|update|delete).*from',  # SQL injection
            r'(\${.*})',  # Template injection
        ]
    
    def detect_intrusion(self, request_data):
        alerts = []
        
        for pattern in self.suspicious_patterns:
            if re.search(pattern, str(request_data), re.IGNORECASE):
                alerts.append({
                    'type': 'suspicious_pattern',
                    'pattern': pattern,
                    'data': request_data,
                    'timestamp': datetime.utcnow()
                })
        
        return alerts
    
    def check_rate_limit_abuse(self, client_ip, requests_per_minute=100):
        # Implement rate limit abuse detection
        key = f"requests:{client_ip}"
        current_requests = self.redis.incr(key)
        
        if current_requests == 1:
            self.redis.expire(key, 60)
        
        if current_requests > requests_per_minute:
            return {
                'type': 'rate_limit_abuse',
                'client_ip': client_ip,
                'requests': current_requests,
                'timestamp': datetime.utcnow()
            }
        
        return None
```

### Security Logging
```python
import logging
from datetime import datetime

class SecurityLogger:
    def __init__(self):
        self.logger = logging.getLogger('security')
        handler = logging.FileHandler('/var/log/security.log')
        formatter = logging.Formatter(
            '%(asctime)s - %(levelname)s - %(message)s'
        )
        handler.setFormatter(formatter)
        self.logger.addHandler(handler)
        self.logger.setLevel(logging.INFO)
    
    def log_login_attempt(self, username, ip_address, success):
        event = {
            'event_type': 'login_attempt',
            'username': username,
            'ip_address': ip_address,
            'success': success,
            'timestamp': datetime.utcnow().isoformat()
        }
        self.logger.info(f"LOGIN_ATTEMPT: {event}")
    
    def log_permission_denied(self, user_id, resource, action):
        event = {
            'event_type': 'permission_denied',
            'user_id': user_id,
            'resource': resource,
            'action': action,
            'timestamp': datetime.utcnow().isoformat()
        }
        self.logger.warning(f"PERMISSION_DENIED: {event}")
    
    def log_security_violation(self, violation_type, details):
        event = {
            'event_type': 'security_violation',
            'violation_type': violation_type,
            'details': details,
            'timestamp': datetime.utcnow().isoformat()
        }
        self.logger.error(f"SECURITY_VIOLATION: {event}")
```

## Common Security Vulnerabilities

### OWASP Top 10

#### 1. Injection Attacks
```python
# Vulnerable code
def get_user(user_id):
    query = f"SELECT * FROM users WHERE id = {user_id}"
    return database.execute(query)

# Secure code
def get_user(user_id):
    query = "SELECT * FROM users WHERE id = %s"
    return database.execute(query, (user_id,))
```

#### 2. Broken Authentication
```python
# Vulnerable: No rate limiting
def login(username, password):
    user = get_user(username)
    if user and verify_password(password, user.password_hash):
        return create_session(user)
    return None

# Secure: With rate limiting
def login(username, password, ip_address):
    if rate_limiter.is_blocked(ip_address):
        raise Exception("Too many login attempts")
    
    user = get_user(username)
    if user and verify_password(password, user.password_hash):
        rate_limiter.reset_attempts(ip_address)
        return create_session(user)
    else:
        rate_limiter.record_failed_attempt(ip_address)
        return None
```

#### 3. Sensitive Data Exposure
```python
# Vulnerable: Logging sensitive data
def process_payment(card_number, amount):
    logger.info(f"Processing payment for card {card_number}")
    # Process payment...

# Secure: Mask sensitive data
def process_payment(card_number, amount):
    masked_card = mask_credit_card(card_number)
    logger.info(f"Processing payment for card {masked_card}")
    # Process payment...
```

## Interview Tips

### Common Questions
1. "How would you secure a REST API?"
2. "What's the difference between authentication and authorization?"
3. "How would you prevent SQL injection?"
4. "What are the best practices for password storage?"
5. "How do you implement zero-trust security?"
6. "What's the difference between RBAC and ABAC?"
7. "How do you secure containers in Kubernetes?"
8. "What are the key principles of DevSecOps?"

### Answer Framework
1. **Identify Threats**: What are we protecting against?
2. **Layer Security**: Defense in depth approach (CIA triad + zero-trust)
3. **Implement Controls**: Authentication, authorization, encryption, monitoring
4. **Cloud Security**: IAM, VPC, container security
5. **Compliance**: GDPR, SOC 2, industry-specific requirements
6. **DevSecOps**: Security in CI/CD, automated testing
7. **Monitor and Respond**: Detection, incident response, continuous monitoring
8. **Review and Update**: Regular security assessments and updates

### Key Points to Emphasize
- Security is a process, not a product
- Principle of least privilege and zero-trust
- Defense in depth with multiple security layers
- Compliance requirements drive security decisions
- DevSecOps integrates security throughout development
- Regular security audits and incident response planning
- Cloud security differs from traditional security

## Practice Problems

1. **Design secure authentication system with zero-trust**
2. **Implement API rate limiting and security monitoring**
3. **Design data encryption strategy for cloud storage**
4. **Create security monitoring system with SIEM**
5. **Design secure container orchestration (Kubernetes)**
6. **Implement DevSecOps pipeline with security gates**
7. **Design GDPR-compliant data processing system**
8. **Create multi-cloud security architecture**

## Further Reading

- **Books**: "The Web Application Hacker's Handbook" by Dafydd Stuttard
- **Standards**: OWASP Top 10, NIST Cybersecurity Framework
- **Certifications**: CISSP, CEH, CompTIA Security+
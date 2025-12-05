# Authentication and Authorization

## Definition

Authentication is the process of verifying who a user is, while authorization is the process of verifying what they have access to. Together, they form the foundation of secure system access control.

## Authentication Fundamentals

### Authentication Factors
```
Something You Know:
├── Passwords
├── PINs
├── Security Questions
└── One-Time Passwords (OTPs)

Something You Have:
├── Mobile Devices
├── Hardware Tokens
├── Smart Cards
└── Biometric Devices

Something You Are:
├── Fingerprints
├── Face Recognition
├── Voice Recognition
└── Iris Scans
```

### Multi-Factor Authentication (MFA)
```
Level 1: Single Factor (Password Only)
Level 2: Two-Factor (Password + SMS/Email)
Level 3: Multi-Factor (Password + App + Biometric)
```

## Authentication Methods

### Password-Based Authentication
```python
import bcrypt
import secrets
import re
from datetime import datetime, timedelta

class PasswordAuth:
    def __init__(self):
        self.min_password_length = 8
        self.max_password_length = 128
        self.password_history = {}
    
    def validate_password_strength(self, password):
        """Validate password strength requirements"""
        errors = []
        
        # Length requirements
        if len(password) < self.min_password_length:
            errors.append(f"Password must be at least {self.min_password_length} characters")
        
        if len(password) > self.max_password_length:
            errors.append(f"Password must not exceed {self.max_password_length} characters")
        
        # Complexity requirements
        if not re.search(r'[A-Z]', password):
            errors.append("Password must contain at least one uppercase letter")
        
        if not re.search(r'[a-z]', password):
            errors.append("Password must contain at least one lowercase letter")
        
        if not re.search(r'\d', password):
            errors.append("Password must contain at least one digit")
        
        if not re.search(r'[!@#$%^&*(),.?":{}|<>]', password):
            errors.append("Password must contain at least one special character")
        
        # Common patterns
        if re.search(r'(.)\1{2,}', password):
            errors.append("Password cannot contain 3 or more repeating characters")
        
        return errors
    
    def hash_password(self, password):
        """Hash password using bcrypt"""
        salt = bcrypt.gensalt()
        hashed = bcrypt.hashpw(password.encode('utf-8'), salt)
        return hashed.decode('utf-8')
    
    def verify_password(self, password, hashed_password):
        """Verify password against hash"""
        return bcrypt.checkpw(
            password.encode('utf-8'),
            hashed_password.encode('utf-8')
        )
    
    def generate_secure_password(self, length=16):
        """Generate secure random password"""
        alphabet = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%^&*"
        password = ''.join(secrets.choice(alphabet) for _ in range(length))
        return password
    
    def check_password_history(self, user_id, new_password):
        """Check if password was used before"""
        if user_id not in self.password_history:
            return True
        
        user_history = self.password_history[user_id]
        
        for old_hashed_password in user_history:
            if self.verify_password(new_password, old_hashed_password):
                return False
        
        return True
    
    def add_to_password_history(self, user_id, hashed_password):
        """Add password to user's history"""
        if user_id not in self.password_history:
            self.password_history[user_id] = []
        
        user_history = self.password_history[user_id]
        user_history.append(hashed_password)
        
        # Keep only last 5 passwords
        if len(user_history) > 5:
            user_history.pop(0)

# Usage
auth = PasswordAuth()

# Validate password
password = "SecurePass123!"
errors = auth.validate_password_strength(password)
if errors:
    print("Password validation errors:")
    for error in errors:
        print(f"  - {error}")
else:
    print("Password is strong")
    
    # Hash password
    hashed_password = auth.hash_password(password)
    print(f"Hashed password: {hashed_password}")
    
    # Verify password
    is_valid = auth.verify_password(password, hashed_password)
    print(f"Password verification: {is_valid}")
```

### Token-Based Authentication (JWT)
```python
import jwt
import uuid
from datetime import datetime, timedelta
from functools import wraps

class JWTAuth:
    def __init__(self, secret_key, algorithm='HS256', token_expiry=3600):
        self.secret_key = secret_key
        self.algorithm = algorithm
        self.token_expiry = token_expiry
        self.blacklisted_tokens = set()
    
    def generate_token(self, user_id, user_data=None):
        """Generate JWT token"""
        payload = {
            'user_id': user_id,
            'jti': str(uuid.uuid4()),  # JWT ID
            'iat': datetime.utcnow(),
            'exp': datetime.utcnow() + timedelta(seconds=self.token_expiry)
        }
        
        if user_data:
            payload.update(user_data)
        
        token = jwt.encode(payload, self.secret_key, algorithm=self.algorithm)
        return token
    
    def verify_token(self, token):
        """Verify JWT token"""
        try:
            payload = jwt.decode(
                token,
                self.secret_key,
                algorithms=[self.algorithm]
            )
            
            # Check if token is blacklisted
            if payload.get('jti') in self.blacklisted_tokens:
                return None, "Token has been revoked"
            
            return payload, None
        
        except jwt.ExpiredSignatureError:
            return None, "Token has expired"
        except jwt.InvalidTokenError:
            return None, "Invalid token"
    
    def refresh_token(self, token):
        """Refresh JWT token"""
        payload, error = self.verify_token(token)
        if error:
            return None, error
        
        # Remove old token from blacklist
        if 'jti' in payload:
            self.blacklisted_tokens.discard(payload['jti'])
        
        # Generate new token
        user_data = {k: v for k, v in payload.items() 
                     if k not in ['iat', 'exp', 'jti']}
        
        new_token = self.generate_token(payload['user_id'], user_data)
        return new_token, None
    
    def revoke_token(self, token):
        """Revoke JWT token"""
        payload, error = self.verify_token(token)
        if not error and 'jti' in payload:
            self.blacklisted_tokens.add(payload['jti'])
    
    def token_required(self, f):
        """Decorator to require JWT token"""
        @wraps(f)
        def decorated_function(*args, **kwargs):
            token = None
            
            # Get token from Authorization header
            auth_header = kwargs.get('headers', {}).get('Authorization')
            if auth_header and auth_header.startswith('Bearer '):
                token = auth_header.split(' ')[1]
            
            if not token:
                return {'error': 'Token is missing'}, 401
            
            payload, error = self.verify_token(token)
            if error:
                return {'error': error}, 401
            
            # Add user info to kwargs
            kwargs['current_user'] = payload
            kwargs['user_id'] = payload['user_id']
            
            return f(*args, **kwargs)
        
        return decorated_function

# Usage
jwt_auth = JWTAuth('your-secret-key', token_expiry=3600)

# Generate token
user_data = {
    'username': 'john_doe',
    'email': 'john@example.com',
    'role': 'user'
}

token = jwt_auth.generate_token('user123', user_data)
print(f"Generated token: {token}")

# Verify token
payload, error = jwt_auth.verify_token(token)
if error:
    print(f"Token verification failed: {error}")
else:
    print(f"Token payload: {payload}")

# Decorator usage
@jwt_auth.token_required
def protected_route(user_id, current_user, **kwargs):
    return {
        'message': 'Access granted to protected resource',
        'user_id': user_id,
        'user_data': current_user
    }

# Simulate request
headers = {'Authorization': f'Bearer {token}'}
result = protected_route(headers=headers)
print(f"Protected route result: {result}")
```

### OAuth 2.0 Implementation
```python
import secrets
import hashlib
from urllib.parse import urlencode
from datetime import datetime, timedelta

class OAuth2Server:
    def __init__(self):
        self.clients = {}
        self.authorization_codes = {}
        self.access_tokens = {}
        self.refresh_tokens = {}
        self.users = {}
    
    def register_client(self, client_id, client_secret, redirect_uris, scopes):
        """Register OAuth client"""
        self.clients[client_id] = {
            'client_secret': client_secret,
            'redirect_uris': redirect_uris,
            'scopes': scopes,
            'created_at': datetime.utcnow()
        }
    
    def generate_authorization_code(self, client_id, user_id, redirect_uri, scope):
        """Generate authorization code"""
        code = secrets.token_urlsafe(32)
        
        self.authorization_codes[code] = {
            'client_id': client_id,
            'user_id': user_id,
            'redirect_uri': redirect_uri,
            'scope': scope,
            'created_at': datetime.utcnow(),
            'expires_at': datetime.utcnow() + timedelta(minutes=10)
        }
        
        return code
    
    def generate_access_token(self, client_id, user_id, scope):
        """Generate access token"""
        access_token = secrets.token_urlsafe(32)
        refresh_token = secrets.token_urlsafe(32)
        
        self.access_tokens[access_token] = {
            'client_id': client_id,
            'user_id': user_id,
            'scope': scope,
            'created_at': datetime.utcnow(),
            'expires_at': datetime.utcnow() + timedelta(hours=1)
        }
        
        self.refresh_tokens[refresh_token] = {
            'client_id': client_id,
            'user_id': user_id,
            'scope': scope,
            'created_at': datetime.utcnow(),
            'access_token': access_token
        }
        
        return {
            'access_token': access_token,
            'token_type': 'Bearer',
            'expires_in': 3600,
            'refresh_token': refresh_token,
            'scope': scope
        }
    
    def validate_authorization_code(self, code, client_id):
        """Validate authorization code"""
        if code not in self.authorization_codes:
            return None, "Invalid authorization code"
        
        auth_data = self.authorization_codes[code]
        
        if auth_data['client_id'] != client_id:
            return None, "Client ID mismatch"
        
        if datetime.utcnow() > auth_data['expires_at']:
            del self.authorization_codes[code]
            return None, "Authorization code expired"
        
        return auth_data, None
    
    def validate_access_token(self, token):
        """Validate access token"""
        if token not in self.access_tokens:
            return None, "Invalid access token"
        
        token_data = self.access_tokens[token]
        
        if datetime.utcnow() > token_data['expires_at']:
            del self.access_tokens[token]
            return None, "Access token expired"
        
        return token_data, None
    
    def refresh_access_token(self, refresh_token, client_id):
        """Refresh access token using refresh token"""
        if refresh_token not in self.refresh_tokens:
            return None, "Invalid refresh token"
        
        refresh_data = self.refresh_tokens[refresh_token]
        
        if refresh_data['client_id'] != client_id:
            return None, "Client ID mismatch"
        
        # Generate new access token
        new_token_data = self.generate_access_token(
            refresh_data['client_id'],
            refresh_data['user_id'],
            refresh_data['scope']
        )
        
        # Remove old refresh token
        del self.refresh_tokens[refresh_token]
        
        return new_token_data, None

# OAuth 2.0 Flow Implementation
class OAuth2Flow:
    def __init__(self, oauth_server):
        self.server = oauth_server
    
    def authorization_endpoint(self, client_id, redirect_uri, scope, response_type='code'):
        """Handle authorization request"""
        # Validate client
        if client_id not in self.server.clients:
            return {'error': 'invalid_client'}, 400
        
        client = self.server.clients[client_id]
        
        if redirect_uri not in client['redirect_uris']:
            return {'error': 'invalid_redirect_uri'}, 400
        
        # In real implementation, show user consent page
        # For demo, auto-approve
        user_id = 'user123'  # Get from authenticated user
        
        auth_code = self.server.generate_authorization_code(
            client_id, user_id, redirect_uri, scope
        )
        
        redirect_url = f"{redirect_uri}?code={auth_code}"
        return {'redirect': redirect_url}, 302
    
    def token_endpoint(self, grant_type, code=None, refresh_token=None, 
                     client_id=None, client_secret=None):
        """Handle token request"""
        # Validate client
        if client_id not in self.server.clients:
            return {'error': 'invalid_client'}, 401
        
        client = self.server.clients[client_id]
        if client['client_secret'] != client_secret:
            return {'error': 'invalid_client'}, 401
        
        if grant_type == 'authorization_code':
            if not code:
                return {'error': 'invalid_request'}, 400
            
            auth_data, error = self.server.validate_authorization_code(code, client_id)
            if error:
                return {'error': error}, 400
            
            # Generate access token
            token_data = self.server.generate_access_token(
                auth_data['client_id'],
                auth_data['user_id'],
                auth_data['scope']
            )
            
            # Remove authorization code
            del self.server.authorization_codes[code]
            
            return token_data, 200
        
        elif grant_type == 'refresh_token':
            if not refresh_token:
                return {'error': 'invalid_request'}, 400
            
            token_data, error = self.server.refresh_access_token(refresh_token, client_id)
            if error:
                return {'error': error}, 400
            
            return token_data, 200
        
        else:
            return {'error': 'unsupported_grant_type'}, 400

# Usage
oauth_server = OAuth2Server()

# Register client
oauth_server.register_client(
    client_id='client123',
    client_secret='secret456',
    redirect_uris=['https://app.example.com/callback'],
    scopes=['read', 'write']
)

oauth_flow = OAuth2Flow(oauth_server)

# Authorization flow
auth_result = auth_flow.authorization_endpoint(
    client_id='client123',
    redirect_uri='https://app.example.com/callback',
    scope='read write'
)

# Token exchange
token_result = oauth_flow.token_endpoint(
    grant_type='authorization_code',
    code='auth_code_from_redirect',
    client_id='client123',
    client_secret='secret456'
)

print(f"Access token result: {token_result}")
```

## Authorization Models

### Role-Based Access Control (RBAC)
```python
from enum import Enum
from typing import Set, Dict, List

class Permission(Enum):
    READ = "read"
    WRITE = "write"
    DELETE = "delete"
    ADMIN = "admin"

class Role:
    def __init__(self, name: str, permissions: Set[Permission]):
        self.name = name
        self.permissions = permissions
    
    def has_permission(self, permission: Permission) -> bool:
        return permission in self.permissions

class RBACSystem:
    def __init__(self):
        self.roles = {}
        self.user_roles = {}
        self.role_hierarchy = {}
    
    def create_role(self, role_name: str, permissions: Set[Permission]):
        role = Role(role_name, permissions)
        self.roles[role_name] = role
        return role
    
    def assign_role_to_user(self, user_id: str, role_name: str):
        if user_id not in self.user_roles:
            self.user_roles[user_id] = set()
        
        self.user_roles[user_id].add(role_name)
    
    def set_role_hierarchy(self, parent_role: str, child_role: str):
        if parent_role not in self.role_hierarchy:
            self.role_hierarchy[parent_role] = set()
        
        self.role_hierarchy[parent_role].add(child_role)
    
    def get_user_permissions(self, user_id: str) -> Set[Permission]:
        """Get all permissions for a user including inherited permissions"""
        if user_id not in self.user_roles:
            return set()
        
        all_permissions = set()
        visited_roles = set()
        
        def collect_permissions(role_name: str):
            if role_name in visited_roles or role_name not in self.roles:
                return
            
            visited_roles.add(role_name)
            role = self.roles[role_name]
            all_permissions.update(role.permissions)
            
            # Add permissions from child roles
            if role_name in self.role_hierarchy:
                for child_role in self.role_hierarchy[role_name]:
                    collect_permissions(child_role)
        
        for role_name in self.user_roles[user_id]:
            collect_permissions(role_name)
        
        return all_permissions
    
    def check_permission(self, user_id: str, permission: Permission) -> bool:
        """Check if user has specific permission"""
        user_permissions = self.get_user_permissions(user_id)
        return permission in user_permissions
    
    def check_any_permission(self, user_id: str, permissions: Set[Permission]) -> bool:
        """Check if user has any of the specified permissions"""
        user_permissions = self.get_user_permissions(user_id)
        return any(perm in user_permissions for perm in permissions)
    
    def check_all_permissions(self, user_id: str, permissions: Set[Permission]) -> bool:
        """Check if user has all of the specified permissions"""
        user_permissions = self.get_user_permissions(user_id)
        return all(perm in user_permissions for perm in permissions)

# Usage
rbac = RBACSystem()

# Create roles
admin_role = rbac.create_role("admin", {Permission.READ, Permission.WRITE, Permission.DELETE, Permission.ADMIN})
editor_role = rbac.create_role("editor", {Permission.READ, Permission.WRITE})
viewer_role = rbac.create_role("viewer", {Permission.READ})

# Set role hierarchy
rbac.set_role_hierarchy("admin", "editor")
rbac.set_role_hierarchy("editor", "viewer")

# Assign roles to users
rbac.assign_role_to_user("user1", "viewer")
rbac.assign_role_to_user("user2", "editor")
rbac.assign_role_to_user("user3", "admin")

# Check permissions
print(f"User1 can read: {rbac.check_permission('user1', Permission.READ)}")
print(f"User1 can write: {rbac.check_permission('user1', Permission.WRITE)}")
print(f"User2 can delete: {rbac.check_permission('user2', Permission.DELETE)}")
print(f"User3 can admin: {rbac.check_permission('user3', Permission.ADMIN)}")

# Get all permissions for user
user2_permissions = rbac.get_user_permissions("user2")
print(f"User2 permissions: {[perm.value for perm in user2_permissions]}")
```

### Attribute-Based Access Control (ABAC)
```python
from typing import Dict, Any, List
from dataclasses import dataclass
from datetime import datetime

@dataclass
class User:
    id: str
    name: str
    department: str
    role: str
    clearance_level: int
    location: str

@dataclass
class Resource:
    id: str
    name: str
    type: str
    owner_department: str
    sensitivity_level: int
    location: str

@dataclass
class Action:
    name: str
    type: str  # read, write, delete, etc.

class ABACPolicy:
    def __init__(self, name: str, description: str):
        self.name = name
        self.description = description
        self.conditions = []
    
    def add_condition(self, condition_func):
        """Add a condition function to the policy"""
        self.conditions.append(condition_func)
    
    def evaluate(self, user: User, resource: Resource, action: Action, context: Dict[str, Any]) -> bool:
        """Evaluate policy against user, resource, action, and context"""
        return all(condition(user, resource, action, context) for condition in self.conditions)

class ABACSystem:
    def __init__(self):
        self.policies = []
        self.policy_cache = {}
    
    def add_policy(self, policy: ABACPolicy):
        self.policies.append(policy)
    
    def check_access(self, user: User, resource: Resource, action: Action, 
                   context: Dict[str, Any] = None) -> bool:
        """Check if user can perform action on resource"""
        if context is None:
            context = {}
        
        # Check cache first
        cache_key = self._generate_cache_key(user, resource, action, context)
        if cache_key in self.policy_cache:
            return self.policy_cache[cache_key]
        
        # Evaluate all policies
        for policy in self.policies:
            if policy.evaluate(user, resource, action, context):
                self.policy_cache[cache_key] = True
                return True
        
        self.policy_cache[cache_key] = False
        return False
    
    def _generate_cache_key(self, user: User, resource: Resource, action: Action, context: Dict[str, Any]) -> str:
        """Generate cache key for access decision"""
        key_data = f"{user.id}:{resource.id}:{action.name}:{hash(str(context))}"
        return hashlib.md5(key_data.encode()).hexdigest()

# Predefined policy conditions
def same_department_condition(user: User, resource: Resource, action: Action, context: Dict[str, Any]) -> bool:
    return user.department == resource.owner_department

def sufficient_clearance_condition(user: User, resource: Resource, action: Action, context: Dict[str, Any]) -> bool:
    return user.clearance_level >= resource.sensitivity_level

def business_hours_condition(user: User, resource: Resource, action: Action, context: Dict[str, Any]) -> bool:
    current_time = context.get('current_time', datetime.now())
    return 9 <= current_time.hour <= 17

def same_location_condition(user: User, resource: Resource, action: Action, context: Dict[str, Any]) -> bool:
    return user.location == resource.location

def read_only_condition(user: User, resource: Resource, action: Action, context: Dict[str, Any]) -> bool:
    return action.type == 'read'

def owner_condition(user: User, resource: Resource, action: Action, context: Dict[str, Any]) -> bool:
    return user.id == resource.owner_department

# Usage
abac = ABACSystem()

# Create policies
read_policy = ABACPolicy("Read Policy", "Allow reading resources in same department")
read_policy.add_condition(same_department_condition)
read_policy.add_condition(sufficient_clearance_condition)

admin_policy = ABACPolicy("Admin Policy", "Admin users can access any resource")
admin_policy.add_condition(lambda u, r, a, c: u.role == 'admin')

business_hours_policy = ABACPolicy("Business Hours Policy", "Only allow access during business hours")
business_hours_policy.add_condition(business_hours_condition)

sensitive_data_policy = ABACPolicy("Sensitive Data Policy", "Require same location for sensitive data")
sensitive_data_policy.add_condition(lambda u, r, a, c: r.sensitivity_level >= 3)
sensitive_data_policy.add_condition(same_location_condition)

abac.add_policy(read_policy)
abac.add_policy(admin_policy)
abac.add_policy(business_hours_policy)
abac.add_policy(sensitive_data_policy)

# Create test data
user1 = User("user1", "John Doe", "Engineering", "Developer", 3, "US")
user2 = User("user2", "Jane Smith", "HR", "Manager", 4, "US")
user3 = User("user3", "Bob Johnson", "Engineering", "Admin", 5, "EU")

resource1 = Resource("doc1", "Project Plan", "document", "Engineering", 2, "US")
resource2 = Resource("doc2", "Employee Records", "document", "HR", 4, "US")
resource3 = Resource("doc3", "Secret Project", "document", "Engineering", 5, "EU")

actions = [
    Action("read", "read"),
    Action("write", "write"),
    Action("delete", "delete")
]

# Test access
context = {'current_time': datetime.now()}

for user in [user1, user2, user3]:
    for resource in [resource1, resource2, resource3]:
        for action in actions:
            access = abac.check_access(user, resource, action, context)
            print(f"{user.name} can {action.name} {resource.name}: {access}")
```

## Session Management

### Session-Based Authentication
```python
import uuid
import time
from datetime import datetime, timedelta
from typing import Dict, Optional

class SessionManager:
    def __init__(self, session_timeout=3600):
        self.sessions = {}
        self.session_timeout = session_timeout
    
    def create_session(self, user_id: str, user_data: Dict[str, Any] = None) -> str:
        """Create new session for user"""
        session_id = str(uuid.uuid4())
        
        session_data = {
            'session_id': session_id,
            'user_id': user_id,
            'created_at': datetime.utcnow(),
            'last_accessed': datetime.utcnow(),
            'ip_address': None,
            'user_agent': None,
            'user_data': user_data or {}
        }
        
        self.sessions[session_id] = session_data
        return session_id
    
    def get_session(self, session_id: str) -> Optional[Dict[str, Any]]:
        """Get session data"""
        if session_id not in self.sessions:
            return None
        
        session = self.sessions[session_id]
        
        # Check if session has expired
        if datetime.utcnow() - session['last_accessed'] > timedelta(seconds=self.session_timeout):
            self.destroy_session(session_id)
            return None
        
        # Update last accessed time
        session['last_accessed'] = datetime.utcnow()
        
        return session
    
    def update_session(self, session_id: str, data: Dict[str, Any]) -> bool:
        """Update session data"""
        if session_id not in self.sessions:
            return False
        
        session = self.sessions[session_id]
        session['user_data'].update(data)
        session['last_accessed'] = datetime.utcnow()
        
        return True
    
    def destroy_session(self, session_id: str) -> bool:
        """Destroy session"""
        if session_id in self.sessions:
            del self.sessions[session_id]
            return True
        return False
    
    def destroy_user_sessions(self, user_id: str) -> int:
        """Destroy all sessions for a user"""
        sessions_to_destroy = []
        
        for session_id, session_data in self.sessions.items():
            if session_data['user_id'] == user_id:
                sessions_to_destroy.append(session_id)
        
        for session_id in sessions_to_destroy:
            del self.sessions[session_id]
        
        return len(sessions_to_destroy)
    
    def cleanup_expired_sessions(self) -> int:
        """Clean up expired sessions"""
        expired_sessions = []
        current_time = datetime.utcnow()
        
        for session_id, session_data in self.sessions.items():
            if current_time - session_data['last_accessed'] > timedelta(seconds=self.session_timeout):
                expired_sessions.append(session_id)
        
        for session_id in expired_sessions:
            del self.sessions[session_id]
        
        return len(expired_sessions)
    
    def get_active_sessions(self, user_id: str) -> List[Dict[str, Any]]:
        """Get all active sessions for a user"""
        active_sessions = []
        
        for session_data in self.sessions.values():
            if session_data['user_id'] == user_id:
                active_sessions.append(session_data)
        
        return active_sessions
    
    def get_session_stats(self) -> Dict[str, Any]:
        """Get session statistics"""
        total_sessions = len(self.sessions)
        unique_users = len(set(session['user_id'] for session in self.sessions.values()))
        
        # Calculate average session age
        current_time = datetime.utcnow()
        total_age = sum(
            (current_time - session['created_at']).total_seconds()
            for session in self.sessions.values()
        )
        avg_age = total_age / total_sessions if total_sessions > 0 else 0
        
        return {
            'total_sessions': total_sessions,
            'unique_users': unique_users,
            'average_age_seconds': avg_age,
            'oldest_session': min(
                (session['created_at'] for session in self.sessions.values()),
                default=None
            ),
            'newest_session': max(
                (session['created_at'] for session in self.sessions.values()),
                default=None
            )
        }

# Session Middleware for Web Frameworks
class SessionMiddleware:
    def __init__(self, session_manager: SessionManager):
        self.session_manager = session_manager
    
    def process_request(self, request_handler):
        """Process request with session management"""
        def wrapper(request):
            # Get session ID from cookie
            session_id = request.cookies.get('session_id')
            
            # Get session data
            session = self.session_manager.get_session(session_id) if session_id else None
            
            # Add session to request
            request.session = session
            
            # Process request
            response = request_handler(request)
            
            # Set session cookie
            if session:
                response.set_cookie('session_id', session['session_id'], 
                                 max_age=self.session_manager.session_timeout,
                                 httponly=True, secure=True)
            
            return response
        
        return wrapper

# Usage
session_manager = SessionManager(session_timeout=1800)  # 30 minutes

# Create session
user_id = "user123"
user_data = {"username": "john_doe", "role": "user"}
session_id = session_manager.create_session(user_id, user_data)

# Get session
session = session_manager.get_session(session_id)
print(f"Session data: {session}")

# Update session
session_manager.update_session(session_id, {"last_action": "view_dashboard"})

# Get updated session
updated_session = session_manager.get_session(session_id)
print(f"Updated session: {updated_session}")

# Get session stats
stats = session_manager.get_session_stats()
print(f"Session stats: {stats}")
```

## Security Best Practices

### Rate Limiting
```python
import time
from collections import defaultdict, deque
from typing import Dict

class RateLimiter:
    def __init__(self):
        self.attempts = defaultdict(deque)
        self.blocked_ips = {}
        self.max_attempts = 5
        self.block_duration = 900  # 15 minutes
        self.window_size = 300  # 5 minutes
    
    def is_allowed(self, identifier: str) -> bool:
        """Check if identifier is allowed to make a request"""
        current_time = time.time()
        
        # Check if identifier is blocked
        if identifier in self.blocked_ips:
            if current_time - self.blocked_ips[identifier] < self.block_duration:
                return False
            else:
                # Unblock after duration
                del self.blocked_ips[identifier]
        
        # Clean old attempts
        attempts = self.attempts[identifier]
        while attempts and current_time - attempts[0] > self.window_size:
            attempts.popleft()
        
        # Check if too many attempts
        if len(attempts) >= self.max_attempts:
            self.blocked_ips[identifier] = current_time
            return False
        
        # Record this attempt
        attempts.append(current_time)
        return True
    
    def get_remaining_attempts(self, identifier: str) -> int:
        """Get remaining attempts before blocking"""
        current_time = time.time()
        attempts = self.attempts[identifier]
        
        # Clean old attempts
        while attempts and current_time - attempts[0] > self.window_size:
            attempts.popleft()
        
        return max(0, self.max_attempts - len(attempts))
    
    def get_time_until_reset(self, identifier: str) -> int:
        """Get time until rate limit resets"""
        if identifier not in self.attempts or not self.attempts[identifier]:
            return 0
        
        oldest_attempt = self.attempts[identifier][0]
        reset_time = oldest_attempt + self.window_size
        current_time = time.time()
        
        return max(0, int(reset_time - current_time))

# Usage
rate_limiter = RateLimiter()

# Test rate limiting
identifier = "192.168.1.1"

for i in range(10):
    allowed = rate_limiter.is_allowed(identifier)
    remaining = rate_limiter.get_remaining_attempts(identifier)
    reset_time = rate_limiter.get_time_until_reset(identifier)
    
    print(f"Attempt {i+1}: Allowed={allowed}, Remaining={remaining}, Reset in={reset_time}s")
    
    if not allowed:
        print("Rate limit exceeded!")
        break
```

### Password Security
```python
import secrets
import string
import re
from typing import List

class PasswordSecurity:
    @staticmethod
    def generate_password(length=16, include_symbols=True) -> str:
        """Generate secure password"""
        characters = string.ascii_letters + string.digits
        if include_symbols:
            characters += "!@#$%^&*()_+-="
        
        password = ''.join(secrets.choice(characters) for _ in range(length))
        return password
    
    @staticmethod
    def check_password_breached(password: str) -> bool:
        """Check if password is in known breached passwords list"""
        # In real implementation, check against haveibeenpwned API
        common_passwords = [
            "password", "123456", "password123", "admin", "qwerty",
            "letmein", "welcome", "monkey", "dragon", "master"
        ]
        
        return password.lower() in common_passwords
    
    @staticmethod
    def estimate_password_strength(password: str) -> Dict[str, Any]:
        """Estimate password strength"""
        score = 0
        feedback = []
        
        # Length
        if len(password) >= 8:
            score += 1
        else:
            feedback.append("Password should be at least 8 characters")
        
        if len(password) >= 12:
            score += 1
        
        if len(password) >= 16:
            score += 1
        
        # Character variety
        if re.search(r'[a-z]', password):
            score += 1
        else:
            feedback.append("Include lowercase letters")
        
        if re.search(r'[A-Z]', password):
            score += 1
        else:
            feedback.append("Include uppercase letters")
        
        if re.search(r'\d', password):
            score += 1
        else:
            feedback.append("Include numbers")
        
        if re.search(r'[!@#$%^&*()_+-=]', password):
            score += 1
        else:
            feedback.append("Include special characters")
        
        # Common patterns
        if re.search(r'(.)\1{2,}', password):
            score -= 1
            feedback.append("Avoid repeating characters")
        
        if re.search(r'(123|abc|qwe)', password.lower()):
            score -= 1
            feedback.append("Avoid common sequences")
        
        # Determine strength
        if score >= 7:
            strength = "Very Strong"
        elif score >= 5:
            strength = "Strong"
        elif score >= 3:
            strength = "Medium"
        else:
            strength = "Weak"
        
        return {
            'score': max(0, score),
            'strength': strength,
            'feedback': feedback
        }

# Usage
password = PasswordSecurity.generate_password(length=16)
print(f"Generated password: {password}")

strength = PasswordSecurity.estimate_password_strength(password)
print(f"Password strength: {strength}")

is_breached = PasswordSecurity.check_password_breached("password123")
print(f"Is password breached: {is_breached}")
```

## Real-World Examples

### Google OAuth 2.0
```
Features:
- Multiple scopes
- Refresh tokens
- Revocation
- Security best practices

Flow:
1. Redirect to Google
2. User consent
3. Authorization code
4. Token exchange
5. Access API
```

### AWS IAM
```
Features:
- Role-based access
- Resource-based policies
- Multi-factor authentication
- Temporary credentials

Components:
- Users
- Groups
- Roles
- Policies
```

### Auth0 Implementation
```
Features:
- Social login
- MFA support
- JWT tokens
- User management

Architecture:
- Authentication API
- Management API
- JWT validation
- User directories
```

## Interview Tips

### Common Questions
1. "What's the difference between authentication and authorization?"
2. "How does OAuth 2.0 work?"
3. "What are JWT tokens and how do they work?"
4. "How would you implement session management?"
5. "What are security best practices for authentication?"

### Answer Framework
1. **Define Requirements**: Security level, user experience, scalability
2. **Choose Authentication Method**: Password, MFA, OAuth, SAML
3. **Design Authorization Model**: RBAC, ABAC, ACL
4. **Implement Security**: Rate limiting, encryption, monitoring
5. **Consider Edge Cases**: Token revocation, session management, recovery

### Key Points to Emphasize
- Security vs usability trade-offs
- Token-based vs session-based authentication
- Role-based vs attribute-based authorization
- Security best practices
- Scalability considerations

## Practice Problems

1. **Design secure authentication system**
2. **Implement OAuth 2.0 flow**
3. **Create role-based access control**
4. **Design session management**
5. **Build password security system**

## Further Reading

- **Books**: "Identity and Access Management" by Phil Windley, "OAuth 2.0 in Action" by Justin Richer
- **Standards**: OAuth 2.0 RFC 6749, JWT RFC 7519, OWASP Authentication Cheat Sheet
- **Documentation**: Auth0 Documentation, Okta Developer Guide, AWS IAM User Guide
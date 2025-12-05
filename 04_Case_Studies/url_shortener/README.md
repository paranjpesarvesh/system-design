# URL Shortener (TinyURL)

## Problem Statement

Design a URL shortening service like TinyURL or bit.ly that takes long URLs and converts them into short, aliases that are easier to share and remember.

## Requirements

### Functional Requirements
- Generate short URL from long URL
- Redirect short URL to original long URL
- Custom alias support (optional)
- URL expiration (optional)
- Analytics and click tracking (optional)

### Non-Functional Requirements
- High availability (99.99%)
- Low latency (redirect < 100ms)
- Scalability to handle millions of URLs
- Durability to prevent URL loss

## Scale Estimation

### Traffic Estimates
- **Daily Active Users**: 1 million
- **URLs created per day**: 10 million
- **Redirects per day**: 100 million
- **Peak QPS**: 1000 creates/s, 10000 redirects/s

### Storage Estimates
- **URL length**: Average 100 characters
- **Short code**: 6-8 characters
- **Metadata**: Creation time, expiration, user info
- **Total per URL**: ~200 bytes
- **Annual storage**: 10M × 365 × 200B = 730GB

## High-Level Design

```
Client → Load Balancer → API Servers → Database
                    ↓
                Cache (Redis)
                    ↓
                Analytics Service
```

## API Design

### Create Short URL
```
POST /api/v1/urls
Content-Type: application/json

{
  "long_url": "https://www.example.com/very/long/path/to/resource",
  "custom_alias": "mylink",  // optional
  "expires_at": "2024-12-31T23:59:59Z"  // optional
}

Response:
{
  "short_url": "https://short.ly/abc123",
  "short_code": "abc123",
  "long_url": "https://www.example.com/very/long/path/to/resource",
  "created_at": "2024-01-01T12:00:00Z",
  "expires_at": "2024-12-31T23:59:59Z"
}
```

### Redirect
```
GET /abc123
Response: 301 Redirect to long_url
```

### Get URL Info
```
GET /api/v1/urls/abc123
Response:
{
  "short_code": "abc123",
  "long_url": "https://www.example.com/very/long/path/to/resource",
  "created_at": "2024-01-01T12:00:00Z",
  "click_count": 150,
  "last_accessed": "2024-01-15T10:30:00Z"
}
```

## Database Design

### URL Table
```sql
CREATE TABLE urls (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    long_url TEXT NOT NULL,
    custom_alias VARCHAR(50) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NULL,
    user_id BIGINT,
    click_count BIGINT DEFAULT 0,
    last_accessed TIMESTAMP NULL,
    INDEX idx_short_code (short_code),
    INDEX idx_custom_alias (custom_alias),
    INDEX idx_user_id (user_id)
);
```

### Analytics Table
```sql
CREATE TABLE url_analytics (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    short_code VARCHAR(10) NOT NULL,
    ip_address VARCHAR(45),
    user_agent TEXT,
    referrer TEXT,
    country VARCHAR(2),
    city VARCHAR(100),
    accessed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_short_code (short_code),
    INDEX idx_accessed_at (accessed_at)
);
```

## Short Code Generation

### Base62 Encoding
```python
import string

BASE62_ALPHABET = string.digits + string.ascii_lowercase + string.ascii_uppercase

def encode_base62(number):
    if number == 0:
        return BASE62_ALPHABET[0]
    
    encoded = []
    while number > 0:
        number, remainder = divmod(number, 62)
        encoded.append(BASE62_ALPHABET[remainder])
    
    return ''.join(reversed(encoded))

def decode_base62(encoded):
    number = 0
    for char in encoded:
        number = number * 62 + BASE62_ALPHABET.index(char)
    return number

# Generate short code from database ID
def generate_short_code(url_id):
    return encode_base62(url_id)
```

### Distributed ID Generation
```python
import time
import random

class SnowflakeGenerator:
    def __init__(self, machine_id):
        self.machine_id = machine_id
        self.sequence = 0
        self.last_timestamp = 0
    
    def generate_id(self):
        timestamp = int(time.time() * 1000)
        
        if timestamp == self.last_timestamp:
            self.sequence = (self.sequence + 1) & 0xFFF
            if self.sequence == 0:
                timestamp = self.wait_next_millis(timestamp)
        else:
            self.sequence = 0
        
        self.last_timestamp = timestamp
        
        return ((timestamp << 22) | 
                (self.machine_id << 12) | 
                self.sequence)
```

## Caching Strategy

### Multi-Level Caching
```
Client → CDN Cache → Application Cache → Database
```

### Cache Implementation
```python
import redis

redis_client = redis.Redis(host='localhost', port=6379, db=0)

class URLService:
    def __init__(self):
        self.cache_ttl = 3600  # 1 hour
    
    def get_long_url(self, short_code):
        # Try cache first
        cache_key = f"url:{short_code}"
        cached_url = redis_client.get(cache_key)
        
        if cached_url:
            return cached_url.decode('utf-8')
        
        # Cache miss - query database
        url_data = self.database.get_url(short_code)
        if url_data:
            # Cache the result
            redis_client.setex(
                cache_key, 
                self.cache_ttl, 
                url_data['long_url']
            )
            return url_data['long_url']
        
        return None
    
    def create_short_url(self, long_url, custom_alias=None):
        # Generate unique short code
        if custom_alias:
            if self.database.alias_exists(custom_alias):
                raise ValueError("Custom alias already exists")
            short_code = custom_alias
        else:
            url_id = self.database.insert_url(long_url)
            short_code = encode_base62(url_id)
        
        # Cache immediately
        cache_key = f"url:{short_code}"
        redis_client.setex(cache_key, self.cache_ttl, long_url)
        
        return short_code
```

## Load Balancing

### Geographic Distribution
```
Client → DNS → Nearest Data Center
                ↓
            Load Balancer
                ↓
        [Web Server 1, Web Server 2, Web Server 3]
```

### Health Checking
```python
class HealthChecker:
    def __init__(self, servers):
        self.servers = servers
        self.healthy_servers = set(servers)
    
    def check_health(self):
        for server in self.servers:
            try:
                response = requests.get(f"{server}/health", timeout=5)
                if response.status_code == 200:
                    self.healthy_servers.add(server)
                else:
                    self.healthy_servers.discard(server)
            except:
                self.healthy_servers.discard(server)
    
    def get_healthy_servers(self):
        return list(self.healthy_servers)
```

## Analytics Implementation

### Real-time Analytics
```python
class AnalyticsService:
    def __init__(self):
        self.redis_client = redis.Redis()
        self.kafka_producer = KafkaProducer()
    
    def track_click(self, short_code, request_info):
        # Increment click count in cache
        self.redis_client.incr(f"clicks:{short_code}")
        
        # Send to analytics pipeline
        analytics_event = {
            'short_code': short_code,
            'timestamp': time.time(),
            'ip_address': request_info['ip'],
            'user_agent': request_info['user_agent'],
            'referrer': request_info['referrer']
        }
        
        self.kafka_producer.send('url_clicks', analytics_event)
    
    def get_click_stats(self, short_code):
        # Get total clicks from cache
        total_clicks = self.redis_client.get(f"clicks:{short_code}")
        return int(total_clicks) if total_clicks else 0
```

## Security Considerations

### Input Validation
```python
def validate_url(url):
    # Check URL format
    if not re.match(r'^https?://', url):
        raise ValueError("URL must start with http:// or https://")
    
    # Check for malicious patterns
    malicious_patterns = ['javascript:', 'data:', 'vbscript:']
    for pattern in malicious_patterns:
        if pattern in url.lower():
            raise ValueError("Potentially malicious URL detected")
    
    # Check URL length
    if len(url) > 2048:
        raise ValueError("URL too long")
    
    return True
```

### Rate Limiting
```python
class RateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client
    
    def is_allowed(self, user_id, limit=100, window=3600):
        key = f"rate_limit:{user_id}"
        current = self.redis.incr(key)
        
        if current == 1:
            self.redis.expire(key, window)
        
        return current <= limit
```

## Performance Optimization

### Database Optimization
```sql
-- Partition by date for analytics
CREATE TABLE url_analytics_2024_01 PARTITION OF url_analytics
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Read replicas for redirect queries
CREATE DATABASE url_shortener_readonly;
```

### Connection Pooling
```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    'postgresql://user:pass@localhost/urlshortener',
    poolclass=QueuePool,
    pool_size=20,
    max_overflow=30,
    pool_pre_ping=True
)
```

## Monitoring and Metrics

### Key Metrics
- **QPS**: Requests per second
- **Latency**: Response time percentiles
- **Error Rate**: Failed requests percentage
- **Cache Hit Ratio**: Cache effectiveness
- **Database Connections**: Connection pool usage

### Monitoring Implementation
```python
from prometheus_client import Counter, Histogram, Gauge

# Metrics
url_creates = Counter('url_creates_total', 'Total URLs created')
url_redirects = Counter('url_redirects_total', 'Total redirects')
request_duration = Histogram('request_duration_seconds', 'Request duration')
active_connections = Gauge('active_db_connections', 'Active DB connections')

def track_request_duration(func):
    def wrapper(*args, **kwargs):
        with request_duration.time():
            return func(*args, **kwargs)
    return wrapper
```

## Deployment Architecture

### Container Orchestration
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: url-shortener
spec:
  replicas: 10
  selector:
    matchLabels:
      app: url-shortener
  template:
    metadata:
      labels:
        app: url-shortener
    spec:
      containers:
      - name: app
        image: url-shortener:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

## Cost Optimization

### Storage Optimization
- **Compression**: Compress long URLs
- **Data Archival**: Move old analytics to cold storage
- **Database Sharding**: Distribute load across multiple databases

### Compute Optimization
- **Auto-scaling**: Scale based on traffic patterns
- **Serverless**: Use functions for redirect service
- **CDN**: Offload redirect traffic to CDN

## Interview Discussion Points

### Trade-offs
1. **Short Code Length**: Shorter codes = less available, longer codes = better distribution
2. **Consistency**: Strong consistency vs performance for redirects
3. **Analytics**: Real-time vs batch processing
4. **Custom Aliases**: User experience vs collision complexity

### Follow-up Questions
1. How would you handle URL expiration?
2. What about malicious URL detection?
3. How would you implement custom domains?
4. How to handle billions of URLs?
5. What about GDPR compliance?

## Extensions

### Advanced Features
- **QR Code Generation**: Generate QR codes for short URLs
- **Bulk Operations**: Create multiple short URLs at once
- **API Rate Limiting**: Tiered access levels
- **Custom Domains**: Allow users to use their own domains
- **Link Preview**: Generate preview images and descriptions

### Business Features
- **Monetization**: Premium features, analytics
- **Enterprise Features**: SSO, custom branding
- **API Documentation**: Swagger/OpenAPI integration
- **Developer Portal**: API keys, usage dashboard

## Further Reading

- **Papers**: "URL Shortening Services: A Survey" 
- **Books**: "High Performance Browser Networking" by Ilya Grigorik
- **Documentation**: Redis Documentation, PostgreSQL Partitioning
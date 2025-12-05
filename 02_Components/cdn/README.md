# Content Delivery Networks (CDN)

## Definition

A Content Delivery Network (CDN) is a geographically distributed network of proxy servers and their data centers that delivers web content and other assets to users based on their geographic location, the origin of the webpage, and the content delivery server.

## CDN Architecture

### Basic CDN Structure
```
User Request
    ↓
DNS Resolution → Nearest CDN Edge
    ↓
┌──────────────────────────────────────────────────┐
│                  CDN Network                     │
│                                                  │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐           │
│  │ Edge 1  │  │ Edge 2  │  │ Edge 3  │           │
│  │ (US)    │  │ (EU)    │  │ (Asia)  │           │
│  └─────────┘  └─────────┘  └─────────┘           │
│       ↓            ↓            ↓                │
│  Cache Hit?   Cache Hit?   Cache Hit?            │
│       ↓            ↓            ↓                │
│  Response     Response     Response              │
│       ↓            ↓            ↓                │
│    User         User         User                │
│                                                  │
│  If Cache Miss → Origin Server → Update Cache    │
└──────────────────────────────────────────────────┘
```

### CDN Components
```
┌────────────────────────────────────────────────────┐
│                   CDN Components                   │
│                                                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │   DNS       │  │   Edge      │  │   Origin    │ │
│  │  Resolver   │  │   Servers   │  │   Server    │ │
│  │             │  │             │  │             │ │
│  │ Geo-routing │  │ Caching     │  │ Content     │ │
│  │ Load bal.   │  │ Compression │  │ Generation  │ │
│  └─────────────┘  └─────────────┘  └─────────────┘ │
│                                                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │   Control   │  │ Monitoring  │  │   Security  │ │
│  │   Plane     │  │ & Analytics │  │   Layer     │ │
│  │             │  │             │  │             │ │
│  │ Config Mgmt │  │ Performance │  │ DDoS Prot.  │ │
│  │ Cache Purge │  │ Usage Stats │  │ WAF         │ │
│  └─────────────┘  └─────────────┘  └─────────────┘ │
└────────────────────────────────────────────────────┘
```

## CDN Caching Strategies

### Cache Control Headers
```http
# Strong caching
Cache-Control: public, max-age=31536000, immutable
Expires: Thu, 31 Dec 2030 23:59:59 GMT
ETag: "abc123"

# No caching
Cache-Control: no-cache, no-store, must-revalidate
Pragma: no-cache
Expires: 0

# Conditional caching
Cache-Control: public, max-age=3600
Vary: Accept-Encoding, User-Agent
Last-Modified: Wed, 01 Jan 2024 00:00:00 GMT
```

### Cache Hierarchy
```
Browser Cache
    ↓ (miss)
CDN Edge Cache
    ↓ (miss)
CDN Regional Cache
    ↓ (miss)
Origin Server
```

**Cache Levels:**
1. **Browser Cache**: Client-side storage
2. **Edge Cache**: Nearest CDN node
3. **Regional Cache**: Regional aggregation
4. **Origin Server**: Content source

### Cache Key Generation
```python
import hashlib
from urllib.parse import urlparse, parse_qs

class CDNCacheKey:
    def __init__(self):
        self.vary_headers = ['Accept-Encoding', 'User-Agent', 'Accept-Language']

    def generate_cache_key(self, request_url, headers):
        # Parse URL
        parsed_url = urlparse(request_url)

        # Base key from URL path and query
        base_key = f"{parsed_url.path}"
        if parsed_url.query:
            # Sort query parameters for consistency
            query_params = parse_qs(parsed_url.query)
            sorted_params = sorted(query_params.items())
            base_key += f"?{sorted_params}"

        # Add Vary header values
        vary_values = []
        for header in self.vary_headers:
            if header in headers:
                vary_values.append(f"{header}:{headers[header]}")

        if vary_values:
            base_key += f"|{'|'.join(vary_values)}"

        # Generate hash for cache key
        cache_key = hashlib.sha256(base_key.encode()).hexdigest()

        return {
            'cache_key': cache_key,
            'base_key': base_key,
            'ttl': self.calculate_ttl(headers)
        }

    def calculate_ttl(self, headers):
        # Extract TTL from Cache-Control header
        cache_control = headers.get('Cache-Control', '')

        if 'max-age=' in cache_control:
            max_age = cache_control.split('max-age=')[1].split(',')[0]
            return int(max_age)

        # Default TTL
        return 3600  # 1 hour

# Usage
cache_key_generator = CDNCacheKey()
request_headers = {
    'Accept-Encoding': 'gzip, deflate',
    'User-Agent': 'Mozilla/5.0...',
    'Cache-Control': 'max-age=7200'
}

cache_info = cache_key_generator.generate_cache_key(
    'https://example.com/images/photo.jpg?size=large&format=webp',
    request_headers
)
```

## CDN Configuration

### CloudFlare Configuration
```javascript
// CloudFlare Workers for custom caching
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  const url = new URL(request.url)

  // Custom caching logic
  if (shouldCache(request)) {
    const cache = caches.default
    const cacheKey = new Request(url, request)

    let response = await cache.match(cacheKey)

    if (!response) {
      // Cache miss - fetch from origin
      response = await fetch(request)

      // Cache response with custom TTL
      if (response.ok) {
        response = new Response(response.body, {
          status: response.status,
          statusText: response.statusText,
          headers: {
            ...response.headers,
            'Cache-Control': 'public, max-age=3600',
            'X-Cache-Status': 'MISS'
          }
        })

        event.waitUntil(cache.put(cacheKey, response.clone()))
      }
    } else {
      // Cache hit
      response = new Response(response.body, {
        status: response.status,
        statusText: response.statusText,
        headers: {
          ...response.headers,
          'X-Cache-Status': 'HIT'
        }
      })
    }

    return response
  }

  return fetch(request)
}

function shouldCache(request) {
  const url = new URL(request.url)
  const pathname = url.pathname

  // Cache static assets
  if (pathname.match(/\.(css|js|png|jpg|jpeg|gif|ico|svg|woff|woff2)$/)) {
    return true
  }

  // Cache API responses
  if (pathname.startsWith('/api/')) {
    return request.method === 'GET'
  }

  return false
}
```

### AWS CloudFront Configuration
```python
import boto3

class CloudFrontManager:
    def __init__(self, access_key, secret_key, region='us-east-1'):
        self.client = boto3.client(
            'cloudfront',
            aws_access_key_id=access_key,
            aws_secret_access_key=secret_key,
            region_name=region
        )

    def create_distribution(self, origin_domain_name, caller_reference):
        distribution_config = {
            'CallerReference': caller_reference,
            'Comment': 'Distribution for my website',
            'DefaultRootObject': 'index.html',
            'Origins': {
                'Quantity': 1,
                'Items': [{
                    'Id': 'origin1',
                    'DomainName': origin_domain_name,
                    'CustomOriginConfig': {
                        'HTTPPort': 80,
                        'HTTPSPort': 443,
                        'OriginProtocolPolicy': 'https-only'
                    }
                }]
            },
            'DefaultCacheBehavior': {
                'TargetOriginId': 'origin1',
                'ViewerProtocolPolicy': 'redirect-to-https',
                'TrustedSigners': {
                    'Enabled': False,
                    'Quantity': 0
                },
                'ForwardedValues': {
                    'QueryString': True,
                    'Cookies': {
                        'Forward': 'none'
                    },
                    'Headers': ['Origin', 'Access-Control-Request-Method', 'Access-Control-Request-Headers']
                },
                'MinTTL': 0,
                'Compress': True,
                'AllowedMethods': {
                    'Quantity': 3,
                    'Items': ['GET', 'HEAD', 'OPTIONS']
                },
                'CachedMethods': {
                    'Quantity': 2,
                    'Items': ['GET', 'HEAD']
                },
                'SmoothStreaming': False,
                'DefaultTTL': 86400,
                'MaxTTL': 31536000,
                'FieldLevelEncryptionId': ''
            },
            'CacheBehaviors': {
                'Quantity': 0
            },
            'Enabled': True,
            'ViewerCertificate': {
                'CloudFrontDefaultCertificate': True,
                'SSLSupportMethod': 'sni-only',
                'MinimumProtocolVersion': 'TLSv1.2_2021'
            },
            'PriceClass': 'PriceClass_100',
            'Restrictions': {
                'GeoRestriction': {
                    'RestrictionType': 'none',
                    'Quantity': 0
                }
            }
        }

        response = self.client.create_distribution(
            DistributionConfig=distribution_config
        )

        return response['Distribution']

    def create_invalidation(self, distribution_id, paths):
        invalidation_batch = {
            'Paths': {
                'Quantity': len(paths),
                'Items': paths
            },
            'CallerReference': f'invalidation-{int(time.time())}'
        }

        response = self.client.create_invalidation(
            DistributionId=distribution_id,
            InvalidationBatch=invalidation_batch
        )

        return response['Invalidation']

    def get_distribution_metrics(self, distribution_id):
        # Get CloudWatch metrics for the distribution
        cloudwatch = boto3.client('cloudwatch')

        metrics = cloudwatch.get_metric_statistics(
            Namespace='AWS/CloudFront',
            MetricName='Requests',
            Dimensions=[
                {
                    'Name': 'DistributionId',
                    'Value': distribution_id
                }
            ],
            StartTime=datetime.utcnow() - timedelta(days=7),
            EndTime=datetime.utcnow(),
            Period=3600,
            Statistics=['Sum']
        )

        return metrics['Datapoints']

# Usage
cloudfront = CloudFrontManager('access-key', 'secret-key')
distribution = cloudfront.create_distribution(
    'origin.example.com',
    'my-distribution-v1'
)

# Invalidate cache
invalidation = cloudfront.create_invalidation(
    distribution['Id'],
    ['/images/*', '/css/*', '/js/*']
)
```

## CDN Performance Optimization

### Image Optimization
```python
from PIL import Image
import io
import os

class ImageOptimizer:
    def __init__(self):
        self.supported_formats = ['JPEG', 'PNG', 'WebP']
        self.quality_settings = {
            'low': 60,
            'medium': 75,
            'high': 90
        }

    def optimize_image(self, image_data, format='WebP', quality='medium'):
        try:
            # Open image
            image = Image.open(io.BytesIO(image_data))

            # Convert to RGB if necessary
            if image.mode in ('RGBA', 'LA', 'P'):
                image = image.convert('RGB')

            # Resize if too large
            max_size = (1920, 1080)
            if image.size[0] > max_size[0] or image.size[1] > max_size[1]:
                image.thumbnail(max_size, Image.Resampling.LANCZOS)

            # Optimize based on format
            output = io.BytesIO()

            if format.upper() == 'JPEG':
                image.save(output, format='JPEG',
                         quality=self.quality_settings[quality],
                         optimize=True, progressive=True)

            elif format.upper() == 'PNG':
                image.save(output, format='PNG',
                         optimize=True, compress_level=6)

            elif format.upper() == 'WEBP':
                image.save(output, format='WebP',
                         quality=self.quality_settings[quality],
                         optimize=True, method=6)

            output.seek(0)
            return output.getvalue()

        except Exception as e:
            print(f"Error optimizing image: {e}")
            return image_data

    def generate_thumbnails(self, image_data, sizes):
        thumbnails = {}

        try:
            image = Image.open(io.BytesIO(image_data))

            for size_name, dimensions in sizes.items():
                # Create thumbnail
                thumbnail = image.copy()
                thumbnail.thumbnail(dimensions, Image.Resampling.LANCZOS)

                # Save thumbnail
                output = io.BytesIO()
                thumbnail.save(output, format='WebP', quality=75, optimize=True)
                output.seek(0)

                thumbnails[size_name] = output.getvalue()

        except Exception as e:
            print(f"Error generating thumbnails: {e}")

        return thumbnails

    def get_image_info(self, image_data):
        try:
            image = Image.open(io.BytesIO(image_data))
            return {
                'format': image.format,
                'mode': image.mode,
                'size': image.size,
                'file_size': len(image_data)
            }
        except Exception as e:
            print(f"Error getting image info: {e}")
            return None

# CDN integration
class CDNImageService:
    def __init__(self, cdn_manager, image_optimizer):
        self.cdn = cdn_manager
        self.optimizer = image_optimizer

    def upload_and_optimize(self, bucket, image_key, image_data):
        # Get original image info
        original_info = self.optimizer.get_image_info(image_data)

        # Optimize main image
        optimized_data = self.optimizer.optimize_image(image_data)

        # Upload optimized version
        self.cdn.upload_object(bucket, image_key, optimized_data)

        # Generate and upload thumbnails
        thumbnail_sizes = {
            'small': (150, 150),
            'medium': (300, 300),
            'large': (600, 600)
        }

        thumbnails = self.optimizer.generate_thumbnails(image_data, thumbnail_sizes)

        for size_name, thumbnail_data in thumbnails.items():
            thumbnail_key = f"{os.path.splitext(image_key)[0]}_{size_name}.webp"
            self.cdn.upload_object(bucket, thumbnail_key, thumbnail_data)

        return {
            'original': image_key,
            'thumbnails': {
                size: f"{os.path.splitext(image_key)[0]}_{size}.webp"
                for size in thumbnail_sizes.keys()
            },
            'info': original_info
        }
```

### HTTP/2 and HTTP/3 Optimization
```python
class HTTP2Optimizer:
    def __init__(self):
        self.server_push_resources = [
            '/css/main.css',
            '/js/main.js',
            '/images/logo.png'
        ]

    def create_http2_response(self, content, content_type):
        headers = {
            'content-type': content_type,
            'cache-control': 'public, max-age=31536000',
            'vary': 'accept-encoding',
            'x-content-type-options': 'nosniff',
            'x-frame-options': 'DENY',
            'strict-transport-security': 'max-age=31536000; includeSubDomains'
        }

        # HTTP/2 specific headers
        if content_type.startswith('text/html'):
            headers['link'] = self._generate_preload_links()

        return {
            'status': 200,
            'headers': headers,
            'body': content
        }

    def _generate_preload_links(self):
        preload_links = []

        for resource in self.server_push_resources:
            if resource.endswith('.css'):
                preload_links.append(f'<{resource}>; rel=preload; as=style')
            elif resource.endswith('.js'):
                preload_links.append(f'<{resource}>; rel=preload; as=script')
            elif resource.endswith(('.png', '.jpg', '.jpeg', '.webp')):
                preload_links.append(f'<{resource}>; rel=preload; as=image')

        return ', '.join(preload_links)

    def optimize_for_http3(self, response):
        # HTTP/3 optimizations
        headers = response['headers']

        # Enable QUIC
        headers['alt-svc'] = 'h3=":443"; ma=2592000'

        # Prioritize critical resources
        if 'content-type' in headers and 'text/html' in headers['content-type']:
            headers['early-data'] = '1'

        return response

# CDN edge server implementation
class EdgeServer:
    def __init__(self):
        self.cache = {}
        self.http2_optimizer = HTTP2Optimizer()

    def handle_request(self, request):
        # Check cache
        cache_key = self._generate_cache_key(request)

        if cache_key in self.cache:
            response = self.cache[cache_key]
            response['headers']['x-cache'] = 'HIT'
        else:
            # Fetch from origin
            response = self._fetch_from_origin(request)

            # Optimize response
            response = self.http2_optimizer.create_http2_response(
                response['body'],
                response['headers']['content-type']
            )

            # Cache response
            self.cache[cache_key] = response
            response['headers']['x-cache'] = 'MISS'

        return response

    def _generate_cache_key(self, request):
        # Include relevant headers in cache key
        key_parts = [
            request['url'],
            request.get('accept-encoding', ''),
            request.get('user-agent', '')
        ]

        return hashlib.sha256('|'.join(key_parts).encode()).hexdigest()
```

## CDN Security

### DDoS Protection
```python
class DDoSProtection:
    def __init__(self):
        self.rate_limits = {
            'per_ip': 100,  # requests per minute
            'per_url': 1000,  # requests per minute
            'global': 10000  # requests per minute
        }
        self.blocked_ips = set()
        self.request_counts = {}

    def is_request_allowed(self, client_ip, url):
        current_time = int(time.time())
        minute_key = current_time // 60

        # Check if IP is blocked
        if client_ip in self.blocked_ips:
            return False, "IP blocked due to suspicious activity"

        # Initialize counters
        if minute_key not in self.request_counts:
            self.request_counts[minute_key] = {
                'per_ip': {},
                'per_url': {},
                'global': 0
            }

        counters = self.request_counts[minute_key]

        # Check per-IP rate limit
        ip_count = counters['per_ip'].get(client_ip, 0)
        if ip_count >= self.rate_limits['per_ip']:
            self.blocked_ips.add(client_ip)
            return False, "Rate limit exceeded for IP"

        # Check per-URL rate limit
        url_count = counters['per_url'].get(url, 0)
        if url_count >= self.rate_limits['per_url']:
            return False, "Rate limit exceeded for URL"

        # Check global rate limit
        if counters['global'] >= self.rate_limits['global']:
            return False, "Global rate limit exceeded"

        # Increment counters
        counters['per_ip'][client_ip] = ip_count + 1
        counters['per_url'][url] = url_count + 1
        counters['global'] += 1

        return True, "Request allowed"

    def cleanup_old_counters(self):
        current_time = int(time.time())
        current_minute = current_time // 60

        # Remove counters older than 5 minutes
        old_minutes = [key for key in self.request_counts.keys()
                      if current_minute - key > 5]

        for old_minute in old_minutes:
            del self.request_counts[old_minute]

        # Unblock IPs after 1 hour
        self.blocked_ips = {ip for ip in self.blocked_ips
                           if current_time - self.blocked_ips[ip] < 3600}

# Web Application Firewall (WAF)
class WAF:
    def __init__(self):
        self.rules = [
            self._detect_sql_injection,
            self._detect_xss,
            self._detect_path_traversal,
            self._detect_command_injection
        ]

    def analyze_request(self, request):
        for rule in self.rules:
            result = rule(request)
            if result['blocked']:
                return {
                    'allowed': False,
                    'reason': result['reason'],
                    'rule': result['rule']
                }

        return {'allowed': True}

    def _detect_sql_injection(self, request):
        sql_patterns = [
            r"(\b(SELECT|INSERT|UPDATE|DELETE|DROP|CREATE|ALTER|EXEC|UNION)\b)",
            r"(--|#|/\*|\*/)",
            r"(\bOR\b.*=.*\bOR\b)",
            r"(\bAND\b.*=.*\bAND\b)",
            r"(\bWHERE\b.*\bOR\b)",
            r"(\bWHERE\b.*\bAND\b)"
        ]

        # Check URL and parameters
        check_data = request['url'] + str(request.get('params', ''))

        for pattern in sql_patterns:
            if re.search(pattern, check_data, re.IGNORECASE):
                return {
                    'blocked': True,
                    'reason': 'SQL injection detected',
                    'rule': 'sql_injection'
                }

        return {'blocked': False}

    def _detect_xss(self, request):
        xss_patterns = [
            r"<script.*?>.*?</script>",
            r"javascript:",
            r"on\w+\s*=",
            r"<iframe.*?>",
            r"<object.*?>",
            r"<embed.*?>"
        ]

        check_data = request['url'] + str(request.get('params', ''))

        for pattern in xss_patterns:
            if re.search(pattern, check_data, re.IGNORECASE):
                return {
                    'blocked': True,
                    'reason': 'XSS attack detected',
                    'rule': 'xss'
                }

        return {'blocked': False}

    def _detect_path_traversal(self, request):
        path_traversal_patterns = [
            r"\.\./",
            r"\.\.\\",
            r"%2e%2e%2f",
            r"%2e%2e\\",
            r"\.\.%2f",
            r"\.\.%5c"
        ]

        for pattern in path_traversal_patterns:
            if re.search(pattern, request['url'], re.IGNORECASE):
                return {
                    'blocked': True,
                    'reason': 'Path traversal detected',
                    'rule': 'path_traversal'
                }

        return {'blocked': False}
```

## CDN Monitoring and Analytics

### Performance Monitoring
```python
class CDNPerformanceMonitor:
    def __init__(self):
        self.metrics = {
            'cache_hit_ratio': 0.0,
            'response_times': [],
            'bandwidth_usage': 0,
            'error_rate': 0.0,
            'geographic_distribution': {}
        }

    def record_request(self, request_data):
        # Record cache hit/miss
        if request_data['cache_status'] == 'HIT':
            self.metrics['cache_hit_ratio'] = (
                self.metrics['cache_hit_ratio'] * 0.9 + 1.0 * 0.1
            )
        else:
            self.metrics['cache_hit_ratio'] = (
                self.metrics['cache_hit_ratio'] * 0.9 + 0.0 * 0.1
            )

        # Record response time
        self.metrics['response_times'].append(request_data['response_time'])

        # Keep only last 1000 response times
        if len(self.metrics['response_times']) > 1000:
            self.metrics['response_times'] = self.metrics['response_times'][-1000:]

        # Record bandwidth
        self.metrics['bandwidth_usage'] += request_data['response_size']

        # Record geographic distribution
        country = request_data.get('country', 'Unknown')
        self.metrics['geographic_distribution'][country] = \
            self.metrics['geographic_distribution'].get(country, 0) + 1

    def get_performance_stats(self):
        response_times = self.metrics['response_times']

        if not response_times:
            return {}

        return {
            'cache_hit_ratio': self.metrics['cache_hit_ratio'],
            'avg_response_time': sum(response_times) / len(response_times),
            'p95_response_time': self._percentile(response_times, 95),
            'p99_response_time': self._percentile(response_times, 99),
            'bandwidth_usage_mb': self.metrics['bandwidth_usage'] / (1024 * 1024),
            'geographic_distribution': self.metrics['geographic_distribution']
        }

    def _percentile(self, data, percentile):
        sorted_data = sorted(data)
        index = int(len(sorted_data) * percentile / 100)
        return sorted_data[min(index, len(sorted_data) - 1)]

# Real User Monitoring (RUM)
class RUMCollector:
    def __init__(self):
        self.user_metrics = {}

    def collect_user_metrics(self, user_data):
        session_id = user_data['session_id']

        if session_id not in self.user_metrics:
            self.user_metrics[session_id] = {
                'page_loads': [],
                'resource_loads': [],
                'errors': [],
                'user_agent': user_data['user_agent'],
                'location': user_data['location']
            }

        session_metrics = self.user_metrics[session_id]

        if user_data['type'] == 'page_load':
            session_metrics['page_loads'].append({
                'url': user_data['url'],
                'load_time': user_data['load_time'],
                'timestamp': user_data['timestamp']
            })

        elif user_data['type'] == 'resource_load':
            session_metrics['resource_loads'].append({
                'url': user_data['url'],
                'load_time': user_data['load_time'],
                'size': user_data['size'],
                'timestamp': user_data['timestamp']
            })

        elif user_data['type'] == 'error':
            session_metrics['errors'].append({
                'message': user_data['message'],
                'url': user_data['url'],
                'timestamp': user_data['timestamp']
            })

    def generate_performance_report(self):
        report = {
            'total_sessions': len(self.user_metrics),
            'avg_page_load_time': 0,
            'error_rate': 0,
            'browser_distribution': {},
            'geographic_distribution': {}
        }

        total_page_loads = 0
        total_load_time = 0
        total_errors = 0
        total_resources = 0

        for session_id, metrics in self.user_metrics.items():
            # Page load times
            for page_load in metrics['page_loads']:
                total_page_loads += 1
                total_load_time += page_load['load_time']

            # Errors
            total_errors += len(metrics['errors'])

            # Browser distribution
            user_agent = metrics['user_agent']
            browser = self._extract_browser(user_agent)
            report['browser_distribution'][browser] = \
                report['browser_distribution'].get(browser, 0) + 1

            # Geographic distribution
            location = metrics['location']
            country = location.get('country', 'Unknown')
            report['geographic_distribution'][country] = \
                report['geographic_distribution'].get(country, 0) + 1

        if total_page_loads > 0:
            report['avg_page_load_time'] = total_load_time / total_page_loads

        total_resources = sum(len(metrics['resource_loads'])
                            for metrics in self.user_metrics.values())

        if total_resources > 0:
            report['error_rate'] = total_errors / total_resources

        return report
```

## Real-World Examples

### Netflix CDN (Open Connect)
```
Architecture:
- Dedicated appliances in ISP networks
- 1000+ locations globally
- 100+ Tbps capacity
- Custom caching algorithms

Features:
- Adaptive bitrate streaming
- Local content caching
- Real-time optimization
- ISP partnerships
```

### YouTube CDN
```
Architecture:
- Global edge network
- Smart caching
- Video optimization
- Live streaming support

Features:
- Adaptive streaming
- Video transcoding
- Thumbnail generation
- Analytics integration
```

### Cloudflare CDN
```
Architecture:
- 200+ data centers
- Anycast network
- DDoS protection
- Web Application Firewall

Features:
- HTTP/3 support
- Image optimization
- Smart routing
- Security features
```

## Interview Tips

### Common Questions
1. "How does a CDN work?"
2. "What are the benefits of using a CDN?"
3. "How would you implement caching?"
4. "How do you handle cache invalidation?"
5. "What are CDN security considerations?"

### Answer Framework
1. **Explain Architecture**: DNS routing, edge servers, origin
2. **Discuss Caching**: Strategies, TTL, cache keys
3. **Cover Optimization**: Performance, compression, HTTP/2
4. **Address Security**: DDoS protection, WAF, encryption
5. **Consider Trade-offs**: Cost vs performance, complexity

### Key Points to Emphasize
- Geographic distribution benefits
- Caching strategies and invalidation
- Performance optimization techniques
- Security considerations
- Monitoring and analytics

## Practice Problems

1. **Design a CDN for video streaming**
2. **Implement cache invalidation system**
3. **Create image optimization service**
4. **Design DDoS protection for CDN**
5. **Build CDN analytics dashboard**

## Further Reading

- **Books**: "High Performance Browser Networking" by Ilya Grigorik
- **Documentation**: AWS CloudFront, Cloudflare Developers, Fastly Documentation
- **Concepts**: HTTP/2, HTTP/3, Caching Strategies, Web Performance

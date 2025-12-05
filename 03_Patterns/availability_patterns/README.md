# Availability Patterns

## Definition

Availability patterns are architectural approaches designed to ensure systems remain operational and accessible even when components fail, minimizing downtime and maintaining service continuity.

## High Availability Principles

### Availability Metrics
```
Availability Calculation:
Availability = (Uptime) / (Uptime + Downtime)

Availability Levels:
99%      = 3.65 days/year downtime
99.9%    = 8.76 hours/year downtime
99.99%   = 52.56 minutes/year downtime
99.999%  = 5.26 minutes/year downtime
```

### High Availability Triangle
```
          High Availability
         /        |        \
        /         |         \
   Redundancy   Monitoring   Failover
        |         |         |
        └─────────┴─────────┘
           System Design
```

## Redundancy Patterns

### Active-Active Redundancy
```
Load Balancer
    ↓
┌─────────────────────────────────┐
│                                 │
│  ┌─────────┐  ┌─────────┐       │
│  │Server A │  │Server B │       │
│  │(Active) │  │(Active) │       │
│  └─────────┘  └─────────┘       │
│                                 │
│  Both servers handle traffic    │
│  Load distribution              │
└─────────────────────────────────┘
```

**Implementation:**
```python
import random
import time
from typing import List, Dict, Optional

class ActiveActiveServer:
    def __init__(self, server_id: str, health_check_url: str):
        self.server_id = server_id
        self.health_check_url = health_check_url
        self.is_healthy = True
        self.last_health_check = time.time()
        self.request_count = 0

    def health_check(self) -> bool:
        try:
            # Simulate health check
            response = requests.get(self.health_check_url, timeout=5)
            self.is_healthy = response.status_code == 200
        except:
            self.is_healthy = False

        self.last_health_check = time.time()
        return self.is_healthy

    def handle_request(self, request_data: Dict) -> Dict:
        if not self.is_healthy:
            raise Exception(f"Server {self.server_id} is unhealthy")

        self.request_count += 1
        return {
            'server_id': self.server_id,
            'request_id': request_data.get('request_id'),
            'response': f"Processed by {self.server_id}",
            'timestamp': time.time()
        }

class ActiveActiveLoadBalancer:
    def __init__(self, servers: List[ActiveActiveServer]):
        self.servers = servers
        self.current_index = 0
        self.round_robin_enabled = True

    def select_server(self) -> Optional[ActiveActiveServer]:
        healthy_servers = [s for s in self.servers if s.is_healthy]

        if not healthy_servers:
            return None

        if self.round_robin_enabled:
            # Round-robin selection
            server = healthy_servers[self.current_index % len(healthy_servers)]
            self.current_index += 1
        else:
            # Least connections selection
            server = min(healthy_servers, key=lambda s: s.request_count)

        return server

    def route_request(self, request_data: Dict) -> Dict:
        server = self.select_server()

        if not server:
            return {
                'error': 'No healthy servers available',
                'status_code': 503
            }

        try:
            return server.handle_request(request_data)
        except Exception as e:
            # Mark server as unhealthy and retry
            server.is_healthy = False
            return self.route_request(request_data)

    def health_check_all(self):
        for server in self.servers:
            server.health_check()

# Usage
servers = [
    ActiveActiveServer("server1", "http://server1.health.com/health"),
    ActiveActiveServer("server2", "http://server2.health.com/health"),
    ActiveActiveServer("server3", "http://server3.health.com/health")
]

load_balancer = ActiveActiveLoadBalancer(servers)

# Route requests
for i in range(10):
    request = {'request_id': f'req_{i}', 'data': f'request_data_{i}'}
    response = load_balancer.route_request(request)
    print(f"Request {i}: {response}")

    # Periodic health check
    if i % 3 == 0:
        load_balancer.health_check_all()
```

### Active-Passive Redundancy
```
Load Balancer
    ↓
┌──────────────────────────────────┐
│                                  │
│  ┌─────────┐  ┌───────────┐      │
│  │ Primary │  │ Secondary │      │
│  │ Server  │  │ Server    │      │
│  │ (Active)│  │ (Passive) │      │
│  └─────────┘  └───────────┘      │
│                                  │
│  Primary handles all traffic     │
│  Secondary on standby            │
└──────────────────────────────────┘
```

**Implementation:**
```python
import time
import threading
from enum import Enum

class ServerState(Enum):
    ACTIVE = "active"
    PASSIVE = "passive"
    FAILED = "failed"

class ActivePassiveServer:
    def __init__(self, server_id: str, is_primary: bool):
        self.server_id = server_id
        self.is_primary = is_primary
        self.state = ServerState.ACTIVE if is_primary else ServerState.PASSIVE
        self.last_heartbeat = time.time()
        self.data_store = {}

    def promote_to_primary(self):
        """Promote passive server to primary"""
        if self.state == ServerState.PASSIVE:
            self.state = ServerState.ACTIVE
            print(f"Server {self.server_id} promoted to primary")

    def demote_to_passive(self):
        """Demote primary server to passive"""
        if self.state == ServerState.ACTIVE:
            self.state = ServerState.PASSIVE
            print(f"Server {self.server_id} demoted to passive")

    def handle_write(self, key: str, value: str) -> bool:
        """Handle write operation"""
        if self.state == ServerState.ACTIVE:
            self.data_store[key] = value
            return True
        else:
            return False  # Passive servers don't handle writes

    def handle_read(self, key: str) -> Optional[str]:
        """Handle read operation"""
        return self.data_store.get(key)

    def sync_data(self, primary_data: Dict):
        """Sync data from primary server"""
        if self.state == ServerState.PASSIVE:
            self.data_store = primary_data.copy()
            print(f"Server {self.server_id} synced data from primary")

class FailoverManager:
    def __init__(self, primary: ActivePassiveServer, secondary: ActivePassiveServer):
        self.primary = primary
        self.secondary = secondary
        self.failover_in_progress = False
        self.heartbeat_interval = 5  # seconds
        self.failover_timeout = 15  # seconds

    def start_heartbeat_monitoring(self):
        """Start monitoring heartbeat between servers"""
        def monitor():
            while True:
                self.check_primary_health()
                time.sleep(self.heartbeat_interval)

        thread = threading.Thread(target=monitor, daemon=True)
        thread.start()

    def check_primary_health(self):
        """Check if primary server is healthy"""
        current_time = time.time()

        if current_time - self.primary.last_heartbeat > self.failover_timeout:
            if not self.failover_in_progress:
                self.initiate_failover()

    def initiate_failover(self):
        """Initiate failover to secondary server"""
        self.failover_in_progress = True
        print("Initiating failover to secondary server")

        # Promote secondary to primary
        self.secondary.promote_to_primary()

        # Sync data from primary (if possible)
        try:
            self.secondary.sync_data(self.primary.data_store)
        except Exception as e:
            print(f"Data sync failed: {e}")

        # Demote failed primary
        self.primary.demote_to_passive()

        # Swap roles
        self.primary, self.secondary = self.secondary, self.primary

        self.failover_in_progress = False
        print("Failover completed")

    def handle_request(self, operation: str, key: str = None, value: str = None):
        """Route request to appropriate server"""
        try:
            if operation == "write":
                success = self.primary.handle_write(key, value)
                if success:
                    # Sync to secondary
                    self.secondary.sync_data(self.primary.data_store)
                return {"status": "success", "server": self.primary.server_id}

            elif operation == "read":
                result = self.primary.handle_read(key)
                return {
                    "status": "success",
                    "value": result,
                    "server": self.primary.server_id
                }

        except Exception as e:
            # Try secondary if primary fails
            try:
                if operation == "read":
                    result = self.secondary.handle_read(key)
                    return {
                        "status": "success",
                        "value": result,
                        "server": self.secondary.server_id
                    }
            except Exception as e2:
                return {"status": "error", "message": str(e2)}

# Usage
primary_server = ActivePassiveServer("server1", is_primary=True)
secondary_server = ActivePassiveServer("server2", is_primary=False)

failover_manager = FailoverManager(primary_server, secondary_server)
failover_manager.start_heartbeat_monitoring()

# Simulate operations
print("Writing data to primary...")
result = failover_manager.handle_request("write", "key1", "value1")
print(f"Write result: {result}")

print("Reading data...")
result = failover_manager.handle_request("read", "key1")
print(f"Read result: {result}")

# Simulate primary failure
print("Simulating primary failure...")
primary_server.last_heartbeat = time.time() - 20  # Simulate timeout

time.sleep(10)  # Wait for failover detection

print("Reading data after failover...")
result = failover_manager.handle_request("read", "key1")
print(f"Read result after failover: {result}")
```

## Geographic Redundancy

### Multi-Region Deployment
```
                    Global Load Balancer (DNS)
                           ↓
        ┌─────────────────────────────────────┐
        │                                     │
        │    US-East Region                   │
        │  ┌─────────────────────────────┐    │
        │  │ Load Balancer               │    │
        │  │    ↓                        │    │
        │  │ ┌─────────┐ ┌─────────┐     │    │
        │  │ │Server A │ │Server B │     │    │
        │  │ └─────────┘ └─────────┘     │    │
        │  │    ↓                        │    │
        │  │ Database Cluster            │    │
        │  └─────────────────────────────┘    │
        │                                     │
        │    EU-West Region                   │
        │  ┌─────────────────────────────┐    │
        │  │ Load Balancer               │    │
        │  │    ↓                        │    │
        │  │ ┌─────────┐ ┌─────────┐     │    │
        │  │ │Server C │ │Server D │     │    │
        │  │ └─────────┘ └─────────┘     │    │
        │  │    ↓                        │    │
        │  │ Database Cluster            │    │
        │  └─────────────────────────────┘    │
        └─────────────────────────────────────┘
```

**Implementation:**
```python
import requests
import time
from typing import Dict, List, Optional
from dataclasses import dataclass

@dataclass
class Region:
    name: str
    load_balancer_url: str
    health_check_url: str
    servers: List[str]
    database_url: str

class GeoRedundantSystem:
    def __init__(self, regions: List[Region]):
        self.regions = regions
        self.primary_region = regions[0]  # First region is primary
        self.region_health = {}
        self.dns_records = {}

    def health_check_region(self, region: Region) -> bool:
        """Check health of a region"""
        try:
            response = requests.get(region.health_check_url, timeout=10)
            is_healthy = response.status_code == 200
        except:
            is_healthy = False

        self.region_health[region.name] = {
            'healthy': is_healthy,
            'last_check': time.time(),
            'response_time': response.elapsed.total_seconds() if 'response' in locals() else None
        }

        return is_healthy

    def select_best_region(self, user_location: str = None) -> Optional[Region]:
        """Select best region based on health and location"""
        healthy_regions = [
            region for region in self.regions
            if self.region_health.get(region.name, {}).get('healthy', False)
        ]

        if not healthy_regions:
            return None

        if user_location:
            # Select region closest to user
            return self._find_closest_region(user_location, healthy_regions)
        else:
            # Select region with best response time
            best_region = min(healthy_regions, key=lambda r:
                self.region_health.get(r.name, {}).get('response_time', float('inf')))
            return best_region

    def _find_closest_region(self, user_location: str, regions: List[Region]) -> Region:
        """Find region closest to user location"""
        # Simplified distance calculation
        # In real implementation, use geolocation database
        region_distances = {
            'US-East': {'US': 0, 'EU': 100, 'ASIA': 200},
            'EU-West': {'US': 100, 'EU': 0, 'ASIA': 100},
            'ASIA-East': {'US': 200, 'EU': 100, 'ASIA': 0}
        }

        closest_region = None
        min_distance = float('inf')

        for region in regions:
            distance = region_distances.get(region.name, {}).get(user_location, float('inf'))
            if distance < min_distance:
                min_distance = distance
                closest_region = region

        return closest_region

    def route_request(self, request_data: Dict, user_location: str = None) -> Dict:
        """Route request to best available region"""
        best_region = self.select_best_region(user_location)

        if not best_region:
            return {
                'error': 'No healthy regions available',
                'status_code': 503
            }

        try:
            response = requests.post(
                f"{best_region.load_balancer_url}/api/request",
                json=request_data,
                timeout=30
            )

            return {
                'region': best_region.name,
                'status_code': response.status_code,
                'data': response.json() if response.headers.get('content-type', '').startswith('application/json') else response.text,
                'response_time': response.elapsed.total_seconds()
            }

        except Exception as e:
            # Mark region as unhealthy and retry
            self.region_health[best_region.name] = {
                'healthy': False,
                'last_check': time.time(),
                'error': str(e)
            }

            return self.route_request(request_data, user_location)

    def update_dns_records(self):
        """Update DNS records based on region health"""
        healthy_regions = [
            region.name for region in self.regions
            if self.region_health.get(region.name, {}).get('healthy', False)
        ]

        # Update DNS with healthy regions
        # In real implementation, use DNS provider API
        self.dns_records = {
            'app.example.com': healthy_regions
        }

        print(f"Updated DNS records: {self.dns_records}")

    def start_health_monitoring(self):
        """Start continuous health monitoring"""
        import threading

        def monitor():
            while True:
                for region in self.regions:
                    self.health_check_region(region)

                self.update_dns_records()
                time.sleep(30)  # Check every 30 seconds

        thread = threading.Thread(target=monitor, daemon=True)
        thread.start()

# Usage
regions = [
    Region(
        name="US-East",
        load_balancer_url="https://us-east.example.com",
        health_check_url="https://us-east.example.com/health",
        servers=["server1", "server2"],
        database_url="postgresql://us-east-db.example.com"
    ),
    Region(
        name="EU-West",
        load_balancer_url="https://eu-west.example.com",
        health_check_url="https://eu-west.example.com/health",
        servers=["server3", "server4"],
        database_url="postgresql://eu-west-db.example.com"
    ),
    Region(
        name="ASIA-East",
        load_balancer_url="https://asia-east.example.com",
        health_check_url="https://asia-east.example.com/health",
        servers=["server5", "server6"],
        database_url="postgresql://asia-east-db.example.com"
    )
]

geo_system = GeoRedundantSystem(regions)
geo_system.start_health_monitoring()

# Route requests from different locations
us_request = {'user_id': 'user1', 'action': 'get_data'}
us_response = geo_system.route_request(us_request, 'US')
print(f"US request response: {us_response}")

eu_request = {'user_id': 'user2', 'action': 'get_data'}
eu_response = geo_system.route_request(eu_request, 'EU')
print(f"EU request response: {eu_response}")

asia_request = {'user_id': 'user3', 'action': 'get_data'}
asia_response = geo_system.route_request(asia_request, 'ASIA')
print(f"Asia request response: {asia_response}")
```

## Circuit Breaker Pattern

### Circuit Breaker Implementation
```python
import time
from enum import Enum
from typing import Callable, Any, Optional

class CircuitState(Enum):
    CLOSED = "closed"      # Normal operation
    OPEN = "open"          # Circuit is open
    HALF_OPEN = "half_open"  # Testing if service recovered

class CircuitBreaker:
    def __init__(self,
                 failure_threshold: int = 5,
                 timeout: int = 60,
                 expected_exception: type = Exception):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.expected_exception = expected_exception

        self.failure_count = 0
        self.last_failure_time = None
        self.state = CircuitState.CLOSED

    def call(self, func: Callable, *args, **kwargs) -> Any:
        """Execute function with circuit breaker protection"""
        if self.state == CircuitState.OPEN:
            if self._should_attempt_reset():
                self.state = CircuitState.HALF_OPEN
            else:
                raise Exception("Circuit breaker is OPEN")

        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result

        except self.expected_exception as e:
            self._on_failure()
            raise e

    def _on_success(self):
        """Handle successful call"""
        self.failure_count = 0
        self.state = CircuitState.CLOSED

    def _on_failure(self):
        """Handle failed call"""
        self.failure_count += 1
        self.last_failure_time = time.time()

        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN

    def _should_attempt_reset(self) -> bool:
        """Check if circuit should attempt to reset"""
        return (self.last_failure_time and
                time.time() - self.last_failure_time >= self.timeout)

    def get_state(self) -> CircuitState:
        """Get current circuit state"""
        return self.state

    def get_stats(self) -> dict:
        """Get circuit breaker statistics"""
        return {
            'state': self.state.value,
            'failure_count': self.failure_count,
            'failure_threshold': self.failure_threshold,
            'last_failure_time': self.last_failure_time
        }

# Usage with external service
class ExternalAPIService:
    def __init__(self):
        self.circuit_breaker = CircuitBreaker(
            failure_threshold=3,
            timeout=30,
            expected_exception=ConnectionError
        )

    def call_external_api(self, endpoint: str) -> dict:
        """Call external API with circuit breaker protection"""
        def api_call():
            response = requests.get(f"https://api.example.com{endpoint}", timeout=10)
            if response.status_code == 200:
                return response.json()
            else:
                raise ConnectionError(f"API returned status {response.status_code}")

        return self.circuit_breaker.call(api_call)

# Usage
api_service = ExternalAPIService()

# Make multiple API calls
for i in range(10):
    try:
        result = api_service.call_external_api("/users")
        print(f"Call {i+1}: Success - {result.get('users', [])[:2]}")
    except Exception as e:
        print(f"Call {i+1}: Failed - {e}")

    print(f"  Circuit state: {api_service.circuit_breaker.get_state()}")
    print(f"  Circuit stats: {api_service.circuit_breaker.get_stats()}")

    time.sleep(1)
```

## Graceful Degradation

### Degradation Strategies
```python
from enum import Enum
from typing import Dict, Any, List

class ServiceLevel(Enum):
    FULL = "full"           # All features available
    DEGRADED = "degraded"   # Limited features
    MINIMAL = "minimal"     # Core features only
    OFFLINE = "offline"     # No service available

class GracefulDegradation:
    def __init__(self):
        self.current_level = ServiceLevel.FULL
        self.feature_flags = {
            ServiceLevel.FULL: {
                'search': True,
                'recommendations': True,
                'analytics': True,
                'real_time_updates': True,
                'media_processing': True
            },
            ServiceLevel.DEGRADED: {
                'search': True,
                'recommendations': False,
                'analytics': False,
                'real_time_updates': True,
                'media_processing': False
            },
            ServiceLevel.MINIMAL: {
                'search': True,
                'recommendations': False,
                'analytics': False,
                'real_time_updates': False,
                'media_processing': False
            },
            ServiceLevel.OFFLINE: {
                'search': False,
                'recommendations': False,
                'analytics': False,
                'real_time_updates': False,
                'media_processing': False
            }
        }

    def check_system_health(self) -> Dict[str, Any]:
        """Check system health metrics"""
        # Simulate health checks
        cpu_usage = self._get_cpu_usage()
        memory_usage = self._get_memory_usage()
        database_health = self._check_database_health()
        external_service_health = self._check_external_services()

        return {
            'cpu_usage': cpu_usage,
            'memory_usage': memory_usage,
            'database_health': database_health,
            'external_service_health': external_service_health,
            'overall_health': self._calculate_overall_health(
                cpu_usage, memory_usage, database_health, external_service_health
            )
        }

    def _get_cpu_usage(self) -> float:
        """Get current CPU usage"""
        # Simulate CPU usage check
        import random
        return random.uniform(20, 90)

    def _get_memory_usage(self) -> float:
        """Get current memory usage"""
        import random
        return random.uniform(30, 85)

    def _check_database_health(self) -> bool:
        """Check database connectivity"""
        import random
        return random.choice([True, True, True, False])  # 75% healthy

    def _check_external_services(self) -> Dict[str, bool]:
        """Check external service health"""
        import random
        return {
            'search_service': random.choice([True, True, False]),  # 66% healthy
            'recommendation_service': random.choice([True, False]),  # 50% healthy
            'analytics_service': random.choice([True, True, True, False])  # 75% healthy
        }

    def _calculate_overall_health(self, cpu: float, memory: float,
                               db: bool, external: Dict[str, bool]) -> float:
        """Calculate overall health score (0-100)"""
        health_score = 100

        # CPU impact
        if cpu > 80:
            health_score -= 20
        elif cpu > 60:
            health_score -= 10

        # Memory impact
        if memory > 80:
            health_score -= 20
        elif memory > 60:
            health_score -= 10

        # Database impact
        if not db:
            health_score -= 30

        # External services impact
        healthy_external = sum(external.values()) / len(external)
        health_score -= (1 - healthy_external) * 20

        return max(0, health_score)

    def determine_service_level(self, health_score: float) -> ServiceLevel:
        """Determine service level based on health score"""
        if health_score >= 80:
            return ServiceLevel.FULL
        elif health_score >= 60:
            return ServiceLevel.DEGRADED
        elif health_score >= 40:
            return ServiceLevel.MINIMAL
        else:
            return ServiceLevel.OFFLINE

    def update_service_level(self):
        """Update current service level based on system health"""
        health_metrics = self.check_system_health()
        health_score = health_metrics['overall_health']
        new_level = self.determine_service_level(health_score)

        if new_level != self.current_level:
            print(f"Service level changing from {self.current_level} to {new_level}")
            self.current_level = new_level
            self._notify_level_change(new_level, health_metrics)

    def _notify_level_change(self, new_level: ServiceLevel, metrics: Dict):
        """Notify about service level change"""
        # Send alert to monitoring system
        alert = {
            'type': 'service_level_change',
            'old_level': self.current_level,
            'new_level': new_level,
            'health_metrics': metrics,
            'timestamp': time.time()
        }

        # In real implementation, send to alerting system
        print(f"ALERT: {alert}")

    def is_feature_enabled(self, feature: str) -> bool:
        """Check if feature is enabled at current service level"""
        return self.feature_flags[self.current_level].get(feature, False)

    def get_available_features(self) -> List[str]:
        """Get list of available features at current service level"""
        return [feature for feature, enabled in
                self.feature_flags[self.current_level].items() if enabled]

# Usage
degradation = GracefulDegradation()

# Simulate service with graceful degradation
class ApplicationService:
    def __init__(self):
        self.degradation = GracefulDegradation()

    def search(self, query: str) -> Dict[str, Any]:
        """Search functionality"""
        if not self.degradation.is_feature_enabled('search'):
            return {
                'error': 'Search service temporarily unavailable',
                'service_level': self.degradation.current_level.value
            }

        # Perform search
        results = self._perform_search(query)

        # Add recommendations if available
        if self.degradation.is_feature_enabled('recommendations'):
            results['recommendations'] = self._get_recommendations(query)

        return results

    def get_analytics(self, user_id: str) -> Dict[str, Any]:
        """Get user analytics"""
        if not self.degradation.is_feature_enabled('analytics'):
            return {
                'error': 'Analytics service temporarily unavailable',
                'service_level': self.degradation.current_level.value
            }

        return self._get_user_analytics(user_id)

    def _perform_search(self, query: str) -> List[Dict]:
        """Perform actual search"""
        # Simulate search results
        return [
            {'id': 1, 'title': f'Result for {query}', 'score': 0.9},
            {'id': 2, 'title': f'Another result for {query}', 'score': 0.8}
        ]

    def _get_recommendations(self, query: str) -> List[Dict]:
        """Get recommendations based on query"""
        return [
            {'id': 101, 'title': f'Recommended for {query}', 'type': 'recommendation'}
        ]

    def _get_user_analytics(self, user_id: str) -> Dict[str, Any]:
        """Get user analytics data"""
        return {
            'user_id': user_id,
            'page_views': 1250,
            'session_duration': 450,
            'last_active': time.time()
        }

# Usage
app_service = ApplicationService()

# Update service level based on current health
app_service.degradation.update_service_level()

# Use service with current feature set
print(f"Current service level: {app_service.degradation.current_level}")
print(f"Available features: {app_service.degradation.get_available_features()}")

# Try different features
search_result = app_service.search("python programming")
print(f"Search result: {search_result}")

analytics_result = app_service.get_analytics("user123")
print(f"Analytics result: {analytics_result}")
```

## Health Checking Systems

### Comprehensive Health Check
```python
import time
import threading
from typing import Dict, List, Callable, Any
from dataclasses import dataclass
from enum import Enum

class HealthStatus(Enum):
    HEALTHY = "healthy"
    UNHEALTHY = "unhealthy"
    DEGRADED = "degraded"

@dataclass
class HealthCheck:
    name: str
    check_function: Callable
    timeout: int = 10
    critical: bool = True

class HealthCheckSystem:
    def __init__(self):
        self.health_checks = []
        self.results = {}
        self.last_check_time = None
        self.check_interval = 30  # seconds

    def add_health_check(self, health_check: HealthCheck):
        """Add a health check to the system"""
        self.health_checks.append(health_check)

    def run_health_checks(self) -> Dict[str, Any]:
        """Run all health checks"""
        results = {}

        for check in self.health_checks:
            start_time = time.time()

            try:
                # Run health check with timeout
                result = self._run_with_timeout(
                    check.check_function,
                    check.timeout
                )

                execution_time = time.time() - start_time

                results[check.name] = {
                    'status': HealthStatus.HEALTHY,
                    'message': 'OK',
                    'execution_time': execution_time,
                    'critical': check.critical,
                    'timestamp': start_time
                }

            except Exception as e:
                execution_time = time.time() - start_time

                results[check.name] = {
                    'status': HealthStatus.UNHEALTHY,
                    'message': str(e),
                    'execution_time': execution_time,
                    'critical': check.critical,
                    'timestamp': start_time
                }

        self.results = results
        self.last_check_time = time.time()

        return results

    def _run_with_timeout(self, func: Callable, timeout: int) -> Any:
        """Run function with timeout"""
        import signal

        def timeout_handler(signum, frame):
            raise TimeoutError(f"Health check timed out after {timeout} seconds")

        # Set timeout signal
        signal.signal(signal.SIGALRM, timeout_handler)
        signal.alarm(timeout)

        try:
            result = func()
            signal.alarm(0)  # Cancel alarm
            return result
        except TimeoutError:
            signal.alarm(0)  # Cancel alarm
            raise
        finally:
            signal.signal(signal.SIGALRM, signal.SIG_DFL)  # Reset signal handler

    def get_overall_status(self) -> Dict[str, Any]:
        """Get overall system health status"""
        if not self.results:
            return {
                'status': HealthStatus.UNHEALTHY,
                'message': 'No health check results available'
            }

        critical_checks = []
        non_critical_checks = []

        for check_name, result in self.results.items():
            if result['critical']:
                critical_checks.append(result)
            else:
                non_critical_checks.append(result)

        # Determine overall status
        critical_failed = any(r['status'] != HealthStatus.HEALTHY for r in critical_checks)
        non_critical_failed = any(r['status'] != HealthStatus.HEALTHY for r in non_critical_checks)

        if critical_failed:
            overall_status = HealthStatus.UNHEALTHY
        elif non_critical_failed:
            overall_status = HealthStatus.DEGRADED
        else:
            overall_status = HealthStatus.HEALTHY

        return {
            'status': overall_status,
            'message': self._get_status_message(overall_status, critical_failed, non_critical_failed),
            'last_check_time': self.last_check_time,
            'critical_checks': len(critical_checks),
            'non_critical_checks': len(non_critical_checks),
            'failed_checks': len([r for r in self.results.values() if r['status'] != HealthStatus.HEALTHY])
        }

    def _get_status_message(self, status: HealthStatus, critical_failed: bool, non_critical_failed: bool) -> str:
        """Generate status message"""
        if status == HealthStatus.HEALTHY:
            return "All systems operational"
        elif status == HealthStatus.DEGRADED:
            return "Some non-critical systems degraded"
        elif status == HealthStatus.UNHEALTHY:
            if critical_failed:
                return "Critical systems failed"
            else:
                return "Multiple system failures"
        return "Unknown status"

    def start_continuous_monitoring(self):
        """Start continuous health monitoring"""
        def monitor():
            while True:
                self.run_health_checks()
                time.sleep(self.check_interval)

        thread = threading.Thread(target=monitor, daemon=True)
        thread.start()

    def get_health_endpoint_response(self) -> Dict[str, Any]:
        """Get response suitable for health endpoint"""
        overall_status = self.get_overall_status()

        response = {
            'status': overall_status['status'].value,
            'timestamp': self.last_check_time,
            'checks': self.results
        }

        # Add HTTP status code based on overall health
        if overall_status['status'] == HealthStatus.HEALTHY:
            response['http_status'] = 200
        elif overall_status['status'] == HealthStatus.DEGRADED:
            response['http_status'] = 200  # Still serve traffic
        else:
            response['http_status'] = 503  # Service unavailable

        return response

# Usage
health_system = HealthCheckSystem()

# Define health checks
def database_health_check():
    """Check database connectivity"""
    # Simulate database check
    import random
    if random.random() > 0.1:  # 90% success rate
        return {"connection": "ok", "response_time": 0.05}
    else:
        raise Exception("Database connection failed")

def redis_health_check():
    """Check Redis connectivity"""
    import random
    if random.random() > 0.05:  # 95% success rate
        return {"connection": "ok", "memory_usage": "45%"}
    else:
        raise Exception("Redis connection failed")

def external_api_health_check():
    """Check external API availability"""
    import random
    if random.random() > 0.2:  # 80% success rate
        return {"api_status": "ok", "response_time": 0.2}
    else:
        raise Exception("External API unavailable")

def disk_space_health_check():
    """Check disk space"""
    import random
    usage = random.uniform(10, 90)
    if usage < 80:
        return {"disk_usage": f"{usage}%", "available_space": "sufficient"}
    else:
        raise Exception(f"Disk usage too high: {usage}%")

# Add health checks to system
health_system.add_health_check(HealthCheck("database", database_health_check, critical=True))
health_system.add_health_check(HealthCheck("redis", redis_health_check, critical=True))
health_system.add_health_check(HealthCheck("external_api", external_api_health_check, critical=False))
health_system.add_health_check(HealthCheck("disk_space", disk_space_health_check, critical=True))

# Start monitoring
health_system.start_continuous_monitoring()

# Get health status
time.sleep(2)  # Wait for initial health check
health_response = health_system.get_health_endpoint_response()
print(f"Health response: {health_response}")
```

## Real-World Examples

### Netflix Chaos Engineering
```
Chaos Monkey:
- Randomly terminates instances
- Tests system resilience
- Automated failure injection
- Gradual escalation

Simian Army:
- Multiple chaos monkeys
- Different failure types
- Coordinated attacks
- Real-world scenarios
```

### Amazon AWS High Availability
```
Multi-AZ Deployment:
- Multiple availability zones
- Automatic failover
- Load distribution
- Data replication

Cross-Region:
- Geographic distribution
- Disaster recovery
- Data synchronization
- Global load balancing
```

### Google Site Reliability
```
SRE Principles:
- Error budget management
- Postmortem culture
- Blameless postmortems
- Automation focus

Availability Targets:
- 99.99% for user-facing services
- 99.9% for internal services
- Measured over rolling windows
- Budget-based alerting
```

## Interview Tips

### Common Questions
1. "How would you design a highly available system?"
2. "What's the difference between active-active and active-passive?"
3. "How does circuit breaker pattern work?"
4. "What is graceful degradation?"
5. "How would you implement health checks?"

### Answer Framework
1. **Define Requirements**: Availability targets, RTO/RPO
2. **Choose Redundancy**: Active-active vs active-passive
3. **Design Failover**: Automatic detection and recovery
4. **Implement Monitoring**: Health checks, metrics, alerting
5. **Plan Degradation**: Graceful service reduction

### Key Points to Emphasize
- Availability vs consistency trade-offs
- Failover automation
- Health check strategies
- Circuit breaker benefits
- Geographic redundancy

## Practice Problems

1. **Design highly available database system**
2. **Implement circuit breaker pattern**
3. **Create multi-region deployment**
4. **Design graceful degradation system**
5. **Build health monitoring system**

## Further Reading

- **Books**: "Site Reliability Engineering" by Google, "Release It!" by Michael Nygard
- **Concepts**: Chaos Engineering, Error Budgets, Postmortem Culture
- **Documentation**: AWS High Availability, Azure Availability Zones, Google SRE Practices

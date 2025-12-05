# Caching Strategies

## Definition

Caching is the process of storing frequently accessed data in a faster storage layer to reduce response time, decrease database load, and improve overall system performance.

## Caching Architecture

```
Client → Application → Cache → Database
                   ↑       ↓
                   └─ Cache Miss
```

## Types of Caching

### 1. Client-Side Caching
```
Browser Cache → CDN Cache → Application Cache → Database
```

- **Browser Cache**: Store resources in user's browser
- **CDN Cache**: Geographic distribution of static content
- **Application Cache**: In-memory storage for frequently accessed data

### 2. Server-Side Caching
- **In-Memory Cache**: Redis, Memcached
- **Database Cache**: Query result caching
- **Object Cache**: Pre-computed objects

## Cache Placement Strategies

### 1. Cache-Aside (Lazy Loading)
```
Application → Check Cache
     ↓              ↓
  Cache Hit    Cache Miss → Database → Update Cache
     ↓              ↓
  Return Data   Return Data
```

**Flow:**
1. Application checks cache first
2. If miss, query database
3. Populate cache with result
4. Return data to client

**Pros:**
- Simple to implement
- Cache only contains needed data
- Fault tolerant

**Cons:**
- Cache miss penalty (three round trips)
- Stale data possible
- Thundering herd problem

### 2. Read-Through Cache
```
Application → Cache → Database (on miss)
     ↓           ↓
  Return Data  Update Cache
```

**Flow:**
1. Application always reads from cache
2. Cache handles database queries on misses
3. Cache updates itself and returns data

**Pros:**
- Simpler application code
- Consistent data access pattern
- Reduced cache miss penalty

**Cons:**
- More complex cache implementation
- Cache becomes critical component

### 3. Write-Through Cache
```
Application → Cache → Database (simultaneous)
     ↓           ↓
  Acknowledge   Persist Data
```

**Flow:**
1. Application writes to cache
2. Cache immediately writes to database
3. Cache acknowledges to application

**Pros:**
- Data consistency guaranteed
- Fast read performance
- No data loss on cache failure

**Cons:**
- Higher write latency
- Increased database load

### 4. Write-Behind (Write-Back) Cache
```
Application → Cache (immediate ack)
                ↓
           Database (async)
```

**Flow:**
1. Application writes to cache
2. Cache immediately acknowledges
3. Cache asynchronously writes to database

**Pros:**
- Very low write latency
- Reduced database load
- Can batch writes

**Cons:**
- Risk of data loss
- Complex failure recovery
- Data inconsistency window

## Cache Eviction Policies

### 1. LRU (Least Recently Used)
```
Cache: [A][B][C][D] (capacity=4)
Access: A → B → E
Result: [E][A][B][C] (D evicted)
```

- **Logic**: Evict least recently accessed item
- **Pros**: Good temporal locality
- **Cons**: Requires tracking access order

### 2. LFU (Least Frequently Used)
```
Cache: [A:2][B:3][C:1][D:4] (frequency)
Access: A → B → E
Result: [E:1][A:3][B:4][D:4] (C evicted)
```

- **Logic**: Evict least frequently accessed item
- **Pros**: Good for stable access patterns
- **Cons**: Requires frequency tracking

### 3. FIFO (First In First Out)
```
Cache: [A][B][C][D] (insertion order)
Insert: E
Result: [B][C][D][E] (A evicted)
```

- **Logic**: Evict oldest item
- **Pros**: Simple implementation
- **Cons**: Ignores access patterns

### 4. Random Replacement
- **Logic**: Randomly select item for eviction
- **Pros**: Very simple
- **Cons**: Poor performance

## Cache Invalidation Strategies

### 1. Time-Based Expiration (TTL)
```
Data: {key: "user:123", value: {...}, ttl: 3600}
Time: 0s → 3600s (valid) → 3601s (expired)
```

- **Pros**: Simple, automatic cleanup
- **Cons**: May serve stale data, premature expiration

### 2. Event-Based Invalidation
```
Database Update → Invalidate Cache
User Update → DELETE cache:user:123
```

- **Pros**: Immediate consistency
- **Cons**: Complex implementation, coupling

### 3. Version-Based Invalidation
```
Cache Key: user:123:v5
Database Update: Increment version → v6
New Cache Key: user:123:v6
```

- **Pros**: No race conditions, supports distributed invalidation
- **Cons**: Key management complexity

## Distributed Caching

### 1. Consistent Hashing
```
Ring: 0 ── 50 ── 100 ── 150 ── 200 ── 255
      ↓      ↓       ↓       ↓       ↓
    Node A  Node B  Node C  Node D  Node E

Key "user123" → Hash(123) = 127 → Node C
```

- **Problem**: Standard hashing causes massive remapping
- **Solution**: Consistent hashing minimizes key movement
- **Benefits**: Only adjacent nodes affected by changes

### 2. Replication Strategies
```
Primary: Node A
Replicas: Node B, Node C

Read: Can read from any replica
Write: Must write to primary, replicate to replicas
```

## Cache Implementation Examples

### Redis Configuration
```redis
# Memory management
maxmemory 2gb
maxmemory-policy allkeys-lru

# Persistence
save 900 1
save 300 10
save 60 10000

# Replication
replicaof 192.168.1.100 6379

# Clustering
cluster-enabled yes
cluster-config-file nodes.conf
```

### Memcached Configuration
```bash
# Start memcached with 2GB memory
memcached -m 2048 -p 11211 -u nobody -d

# Connection pool settings
max_connections: 1000
thread_pool_size: 4
```

### Application Cache Implementation (Python)
```python
import redis
import json
from functools import wraps

redis_client = redis.Redis(host='localhost', port=6379, db=0)

def cache_result(ttl=3600):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            cache_key = f"{func.__name__}:{hash(str(args) + str(kwargs))}"
            
            # Try to get from cache
            cached_result = redis_client.get(cache_key)
            if cached_result:
                return json.loads(cached_result)
            
            # Execute function and cache result
            result = func(*args, **kwargs)
            redis_client.setex(
                cache_key, 
                ttl, 
                json.dumps(result, default=str)
            )
            return result
        return wrapper
    return decorator

@cache_result(ttl=1800)
def get_user_profile(user_id):
    # Database query
    return database.get_user(user_id)
```

## Cache Performance Metrics

### Hit Ratio
```
Hit Ratio = Cache Hits / (Cache Hits + Cache Misses)

Good Hit Ratios:
- 90%+ for hot data
- 70%+ for mixed workload
- 50%+ for random access
```

### Latency
```
Cache Latency: ~1ms
Database Latency: ~100ms
Speedup: 100x
```

### Memory Usage
```
Memory Efficiency = Useful Data / Total Memory
Target: >80%
```

## Cache Warming Strategies

### 1. Preloading
```python
def warm_cache():
    popular_items = get_popular_items()
    for item in popular_items:
        cache.set(f"item:{item.id}", item)
```

### 2. Lazy Population
```python
def get_item(item_id):
    cached_item = cache.get(f"item:{item_id}")
    if not cached_item:
        cached_item = database.get_item(item_id)
        cache.set(f"item:{item_id}", cached_item)
    
    # Trigger background refresh for related items
    background_refresh_related_items(item_id)
    return cached_item
```

## Common Cache Problems

### 1. Cache Stampede (Thundering Herd)
```
Problem: Multiple requests miss cache simultaneously
Solution: Request coalescing, cache warming
```

### 2. Cache Penetration
```
Problem: Requests for non-existent data bypass cache
Solution: Cache null results, Bloom filters
```

### 3. Cache Avalanche
```
Problem: Many cache entries expire simultaneously
Solution: Randomized TTL, staggered expiration
```

## Real-World Examples

### Facebook
- **Cache**: TAO (The Association of Objects)
- **Scale**: Billions of objects, petabytes of data
- **Strategy**: Multi-tier caching, geographic distribution

### Twitter
- **Cache**: Redis clusters
- **Use Case**: Timeline caching, user sessions
- **Strategy**: Write-through cache with TTL

### Amazon
- **Cache**: ElastiCache (Redis/Memcached)
- **Use Case**: Product catalog, session management
- **Strategy**: Read-through with write-behind

## Interview Tips

### Common Questions
1. "Where would you place cache in your system?"
2. "How would you handle cache invalidation?"
3. "What's the difference between cache-aside and read-through?"
4. "How would you design a distributed cache?"

### Answer Framework
1. **Identify Hot Data**: What data is frequently accessed?
2. **Choose Strategy**: Cache-aside, read-through, write-through
3. **Design Invalidation**: TTL, event-based, version-based
4. **Handle Failures**: Cache downtime, network issues
5. **Monitor Performance**: Hit ratio, latency, memory usage

### Key Points to Emphasize
- Trade-offs between consistency and performance
- Cache invalidation challenges
- Distributed caching considerations
- Performance monitoring
- Cost implications

## Practice Problems

1. **Design caching for e-commerce product catalog**
2. **Implement cache invalidation for social media feed**
3. **Design distributed cache for global application**
4. **Handle cache stampede for flash sale**
5. **Design caching strategy for video streaming**

## Further Reading

- **Books**: "Redis Essentials" by Maxwell Dayvson Da Silva
- **Papers**: "Memcached: A Distributed Memory Object Caching System"
- **Documentation**: Redis Documentation, Memcached Wiki
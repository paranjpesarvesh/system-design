# Scalability Patterns

## Definition

Scalability patterns are architectural approaches that enable systems to handle increasing load by adding resources. These patterns help design systems that can grow horizontally or vertically to meet demand.

## Horizontal Scaling Patterns

### 1. Microservices Architecture
```
Monolith
├── User Service
├── Order Service
├── Payment Service
└── Notification Service

↓ Decompose ↓

[User Service] [Order Service] [Payment Service] [Notification Service]
     ↓              ↓              ↓                  ↓
Database A     Database B     Database C        Database D
```

**Benefits:**
- Independent scaling of services
- Technology diversity
- Fault isolation
- Team autonomy

**Challenges:**
- Network latency
- Service discovery
- Distributed transactions
- Operational complexity

### 2. Database Sharding
```
Single Database
├── Users 1-1M
├── Users 1M-2M
├── Users 2M-3M
└── Users 3M-4M

↓ Shard by User ID ↓

Shard 0     Shard 1     Shard 2     Shard 3
Users 0     Users 1     Users 2     Users 3
Mod 4       Mod 4       Mod 4       Mod 4
```

**Sharding Strategies:**
- **Range-based**: User ID 1-1M, 1M-2M, etc.
- **Hash-based**: Hash(user_id) % num_shards
- **Directory-based**: Lookup table for shard mapping
- **Geographic**: Shard by user location

**Benefits:**
- Horizontal database scaling
- Improved query performance
- Better resource utilization

**Challenges:**
- Cross-shard queries
- Rebalancing complexity
- Join operations
- Hotspot prevention

### 3. CQRS (Command Query Responsibility Segregation)
```
Write Side (Commands)
├── Validation
├── Business Logic
└── Event Store

↓ Event Stream ↓

Read Side (Queries)
├── Event Handlers
├── Read Models
└── Optimized Queries
```

**Architecture:**
- **Write Model**: Handles commands, validates, stores events
- **Read Model**: Optimized for queries, denormalized data
- **Event Sourcing**: Immutable log of state changes

**Benefits:**
- Independent scaling of reads and writes
- Optimized read models
- Better performance for specific queries
- Audit trail through event log

**Challenges:**
- Eventual consistency
- Complex event handling
- Schema evolution
- Debugging complexity

## Vertical Scaling Patterns

### 1. Caching Layers
```
Client
  ↓
CDN Cache
  ↓
Application Cache
  ↓
Database Cache
  ↓
Database
```

**Cache Hierarchy:**
- **CDN**: Geographic distribution of static content
- **Application Cache**: Redis/Memcached for hot data
- **Database Cache**: Query result caching
- **Browser Cache**: Client-side storage

### 2. Read Replicas
```
Master DB
├── Write Operations
└── Replication → Replica 1, Replica 2, Replica 3

Read Operations → Load Balancer → [Replica 1, Replica 2, Replica 3]
```

**Replication Strategies:**
- **Synchronous**: Strong consistency, higher latency
- **Asynchronous**: Better performance, eventual consistency
- **Semi-synchronous**: Balance between consistency and performance

## Auto-scaling Patterns

### 1. Horizontal Pod Autoscaler (Kubernetes)
```
Metrics Server
     ↓
CPU/Memory Usage
     ↓
HPA Controller
     ↓
Scale Deployment
     ↓
Add/Remove Pods
```

**Scaling Metrics:**
- **CPU Utilization**: Target 70% average
- **Memory Usage**: Target 80% average
- **Custom Metrics**: Queue length, request rate
- **External Metrics**: Database connections

### 2. Serverless Auto-scaling
```
Function as a Service
├── Automatic scaling
├── Pay-per-use
├── Event-driven
└── Cold start handling
```

**Benefits:**
- Zero infrastructure management
- Automatic scaling to zero
- Cost optimization
- High availability

## Load Distribution Patterns

### 1. Consistent Hashing
```
Ring: 0 ── 50 ── 100 ── 150 ── 200 ── 255
      ↓      ↓       ↓       ↓       ↓
    Node A  Node B  Node C  Node D  Node E

Key "user123" → Hash(123) = 127 → Node C
Key "user456" → Hash(456) = 89  → Node B
```

**Virtual Nodes:**
- Each physical node has multiple virtual nodes
- Better load distribution
- Easier rebalancing

### 2. Weighted Load Balancing
```
Server A (Weight 3): A-A-A-
Server B (Weight 2): -B-B--
Server C (Weight 1): ---C--
Sequence: A-A-A-B-B-C
```

## Data Partitioning Patterns

### 1. Time-based Partitioning
```
Table: events
├── events_2023_01
├── events_2023_02
├── events_2023_03
└── events_2023_04
```

**Use Cases:**
- Log data
- Time series data
- Archival strategies

### 2. Geographic Partitioning
```
US Users → us-east-1 Database
EU Users → eu-west-1 Database
APAC Users → ap-southeast-1 Database
```

**Benefits:**
- Reduced latency
- Data sovereignty compliance
- Regional failure isolation

## Performance Optimization Patterns

### 1. Connection Pooling
```
Application
     ↓
Connection Pool
     ↓
Database Server
```

**Benefits:**
- Reduced connection overhead
- Better resource utilization
- Improved response time

### 2. Batch Processing
```
Individual Requests:
Request 1 → Process → Response
Request 2 → Process → Response
Request 3 → Process → Response

Batch Processing:
[Request 1, Request 2, Request 3] → Process → [Response 1, Response 2, Response 3]
```

**Use Cases:**
- Database writes
- Message processing
- API calls

## Scalability Anti-patterns

### 1. Monolithic Bottleneck
```
Problem: Single component limits entire system
Solution: Decompose into independent services
```

### 2. Database Hotspot
```
Problem: Single database table becomes bottleneck
Solution: Sharding, caching, read replicas
```

### 3. Synchronous Dependencies
```
Problem: Tightly coupled services create cascading failures
Solution: Asynchronous communication, circuit breakers
```

## Real-World Examples

### Netflix
- **Pattern**: Microservices + Auto-scaling
- **Scale**: 200+ million users globally
- **Architecture**: Cloud-native, regional distribution

### Amazon
- **Pattern**: Service-oriented architecture
- **Scale**: Billions of transactions daily
- **Architecture**: Distributed services, database sharding

### Twitter
- **Pattern**: Timeline caching + Fan-out on write
- **Scale**: 500+ million tweets per day
- **Architecture**: Redis clusters, distributed cache

## Implementation Examples

### Kubernetes HPA Configuration
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### Database Sharding Implementation
```python
class ShardManager:
    def __init__(self, num_shards):
        self.num_shards = num_shards
        self.shards = [DatabaseConnection(i) for i in range(num_shards)]
    
    def get_shard(self, key):
        shard_id = hash(key) % self.num_shards
        return self.shards[shard_id]
    
    def write(self, key, value):
        shard = self.get_shard(key)
        return shard.write(key, value)
    
    def read(self, key):
        shard = self.get_shard(key)
        return shard.read(key)
```

## Monitoring Scalability

### Key Metrics
- **Throughput**: Requests per second
- **Latency**: Response time percentiles
- **Error Rate**: Failed requests percentage
- **Resource Utilization**: CPU, memory, network
- **Queue Length**: Pending requests

### Scaling Triggers
```python
def should_scale_up(metrics):
    return (
        metrics.cpu_utilization > 70 or
        metrics.memory_utilization > 80 or
        metrics.queue_length > 100
    )

def should_scale_down(metrics):
    return (
        metrics.cpu_utilization < 30 and
        metrics.memory_utilization < 40 and
        metrics.queue_length < 10
    )
```

## Interview Tips

### Common Questions
1. "How would you scale a database to handle 1 billion users?"
2. "What's the difference between vertical and horizontal scaling?"
3. "How would you design auto-scaling for a microservices application?"
4. "When would you use sharding vs replication?"

### Answer Framework
1. **Understand Requirements**: Growth rate, traffic patterns
2. **Choose Scaling Strategy**: Vertical vs horizontal
3. **Design Architecture**: Components, data flow, dependencies
4. **Plan for Growth**: Auto-scaling, monitoring, optimization
5. **Consider Trade-offs**: Cost, complexity, consistency

### Key Points to Emphasize
- Start with requirements and constraints
- Consider both technical and business factors
- Plan for monitoring and observability
- Think about cost optimization
- Consider team capabilities

## Practice Problems

1. **Design auto-scaling for e-commerce platform**
2. **Scale messaging system to 1 billion users**
3. **Design sharding strategy for social media database**
4. **Implement CQRS pattern for financial system**
5. **Design caching layers for video streaming service**

## Further Reading

- **Books**: "Designing Data-Intensive Applications" by Martin Kleppmann
- **Papers**: "The Google File System", "Dynamo: Amazon's Highly Available Key-value Store"
- **Documentation**: Kubernetes Autoscaling, AWS Auto Scaling
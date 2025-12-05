# Data Management Patterns

## Definition

Data management patterns are architectural approaches for organizing, storing, accessing, and maintaining data in distributed systems to ensure consistency, performance, and scalability.

## Data Partitioning Patterns

### Horizontal Partitioning (Sharding)
```
Single Database:
┌─────────────────────────────┐
│ Users Table (1M records)    │
└─────────────────────────────┘

Sharded by User ID:
Shard 0: Users 0-249,999
Shard 1: Users 250,000-499,999
Shard 2: Users 500,000-749,999
Shard 3: Users 750,000-999,999
```

**Sharding Strategies:**
- **Range-based**: ID ranges (0-999, 1000-1999)
- **Hash-based**: Hash(key) % num_shards
- **Directory-based**: Lookup table for shard mapping

### Vertical Partitioning
```
Single Database:
┌─────────────────────────────────────┐
│ Users │ Posts │ Comments │ Likes    │
└─────────────────────────────────────┘

Partitioned by Functionality:
Users DB: Users table
Posts DB: Posts table
Comments DB: Comments table
Likes DB: Likes table
```

## Consistency Patterns

### Eventual Consistency
```
Write Operation:
Client → Primary → Replicas (async)
    ↓         ↓
  Success   Eventual consistency

Read Operation:
Client → Replica → May return stale data
    ↓
  Eventually consistent
```

### Strong Consistency
```
Write Operation:
Client → Primary → Replicas (sync)
    ↓         ↓
  Success   All replicas updated

Read Operation:
Client → Primary → Always latest data
    ↓
  Strongly consistent
```

## Data Synchronization Patterns

### Change Data Capture (CDC)
```
Database Changes:
INSERT/UPDATE/DELETE
    ↓
CDC Tool (Debezium, Canal)
    ↓
Event Stream → Kafka → Consumers
```

### Dual Write Pattern
```
Client → Database A + Database B
    ↓              ↓
  Success        Success

Sync Process:
Database A ↔ Database B
    ↓
  Consistency Check
```

## Cache-Aside Pattern
```
Read Flow:
Application → Cache → Hit? → Return Data
    ↓           ↓
  Miss        Database → Cache → Return Data

Write Flow:
Application → Database → Invalidate Cache
    ↓
  Success
```

## Read-Through Pattern
```
Read Flow:
Application → Cache → Hit? → Return Data
    ↓           ↓
  Miss        Database → Cache → Return Data
    ↓
  Always through cache
```

## Write-Through Pattern
```
Write Flow:
Application → Cache → Database
    ↓           ↓
  Success     Success
    ↓
  Both updated
```

## Write-Behind Pattern
```
Write Flow:
Application → Cache (immediate ack)
    ↓
  Success
    ↓
Cache → Database (async)
    ↓
  Batched writes
```

## Data Archival Patterns

### Cold-Hot Data Separation
```
Hot Data (Recent):
- Frequently accessed
- Fast storage (SSD)
- High availability

Cold Data (Historical):
- Rarely accessed
- Cheap storage (HDD/Tape)
- Lower availability
```

### Time-Based Partitioning
```
Table Partitioning:
posts_2024_01
posts_2024_02
posts_2024_03
...

Query Optimization:
SELECT * FROM posts_2024_01 WHERE user_id = ?
-- Only scans relevant partition
```

## Data Versioning Patterns

### Immutable Data with Versioning
```
User Record v1:
{id: 1, name: "John", email: "john@old.com", version: 1}

User Record v2:
{id: 1, name: "John", email: "john@new.com", version: 2}

Query:
SELECT * FROM users WHERE id = 1 AND version = 2
-- Always get specific version
```

### Append-Only Storage
```
Events Stream:
[{type: "created", data: {...}, timestamp: t1},
 {type: "updated", data: {...}, timestamp: t2},
 {type: "updated", data: {...}, timestamp: t3}]

Current State:
Replay events to build current state
```

## Data Validation Patterns

### Schema Validation
```
Input Validation:
- Type checking
- Format validation
- Business rules
- Constraint validation

Database Constraints:
- Primary keys
- Foreign keys
- Unique constraints
- Check constraints
```

### Data Quality Checks
```
Quality Metrics:
- Completeness: Required fields present
- Accuracy: Data is correct
- Consistency: No contradictions
- Timeliness: Data is up-to-date
```

## Real-World Examples

### Facebook Data Management
```
Features:
- Sharding by user ID
- Read replicas for scaling
- Memcached for caching
- Cassandra for time-series
- Multi-region replication
```

### LinkedIn Data Management
```
Features:
- Voldemort for key-value storage
- Espresso for document storage
- Kafka for event streaming
- Change data capture
- Real-time synchronization
```

### Netflix Data Management
```
Features:
- EVCache for distributed caching
- S3 for media storage
- Cassandra for user data
- Event-driven architecture
- Multi-AZ deployment
```

## Interview Tips

### Common Questions
1. "How would you partition a database?"
2. "What's the difference between strong and eventual consistency?"
3. "When would you use different caching patterns?"
4. "How would you handle data synchronization?"
5. "What are the trade-offs between different data management approaches?"

### Answer Framework
1. **Understand Requirements**: Scale, consistency, latency
2. **Choose Partitioning**: Horizontal vs vertical
3. **Select Consistency**: Strong vs eventual based on needs
4. **Design Caching**: Pattern based on access patterns
5. **Plan Synchronization**: CDC, dual writes, conflict resolution

### Key Points to Emphasize
- Trade-offs between consistency and performance
- Partitioning strategies and their implications
- Caching patterns and their use cases
- Data synchronization challenges
- Monitoring and maintenance considerations

## Practice Problems

1. **Design data partitioning for social media platform**
2. **Implement caching strategy for e-commerce**
3. **Create data synchronization system**
4. **Design data archival strategy**
5. **Build data validation system**

## Further Reading

- **Books**: "Designing Data-Intensive Applications" by Martin Kleppmann
- **Concepts**: CAP Theorem, Event Sourcing, CQRS, Data Partitioning
- **Documentation**: Kafka Documentation, Redis Patterns, Database Sharding Guides

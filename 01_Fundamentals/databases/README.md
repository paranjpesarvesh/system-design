# Database Fundamentals

## Definition

Databases are organized collections of structured information, or data, typically stored electronically in a computer system. They are the foundation of most applications and critical for system design decisions.

## Database Types

### Relational Databases (SQL)
```
Table: Users
┌─────┬─────────────┬─────────────────┐
│ ID  │ Username    │ Email           │
├─────┼─────────────┼─────────────────┤
│ 1   │ john_doe    │ john@ex.com     │
│ 2   │ jane_smith  │ jane@ex.com     │
└─────┴─────────────┴─────────────────┘

Table: Posts
┌─────┬─────────┬─────────────┬─────────────┐
│ ID  │ User_ID │ Title       │ Content     │
├─────┼─────────┼─────────────┼─────────────┤
│ 1   │ 1       │ My First    │ Hello world │
│ 2   │ 2       │ Hello       │ Hi there    │
└─────┴─────────┴─────────────┴─────────────┘
```

**Characteristics:**
- **Structured Data**: Fixed schema with defined data types
- **ACID Properties**: Atomicity, Consistency, Isolation, Durability
- **Relationships**: Foreign keys and joins between tables
- **SQL Language**: Standardized query language

**Examples:**
- **PostgreSQL**: Open-source, feature-rich, extensible
- **MySQL**: Popular, web-focused, high performance
- **Oracle**: Enterprise-grade, commercial, advanced features
- **SQL Server**: Microsoft ecosystem, Windows integration

### NewSQL Databases
```
NewSQL combines SQL and NoSQL benefits:
- ACID transactions like SQL
- Horizontal scaling like NoSQL
- Distributed architecture
- Cloud-native design

CockroachDB Example:
┌────────────────────────────────────────────────────────┐
│                    CockroachDB Cluster                 │
│                                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   Node 1    │  │   Node 2    │  │   Node 3    │     │
│  │  ┌─────────┐│  │  ┌─────────┐│  │  ┌─────────┐│     │
│  │  │ Range 1 ││  │  │ Range 2 ││  │  │ Range 3 ││     │
│  │  │ Range 4 ││  │  │ Range 5 ││  │  │ Range 6 ││     │
│  │  └─────────┘│  │  └─────────┘│  │  └─────────┘│     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└────────────────────────────────────────────────────────┘
```

**Characteristics:**
- **Geo-distributed**: Multi-region replication
- **Strong consistency**: Serializable isolation
- **Horizontal scaling**: Automatic rebalancing
- **SQL interface**: Familiar query language

**Examples:**
- **CockroachDB**: Cloud-native, geo-distributed
- **TiDB**: MySQL-compatible, HTAP (Hybrid Transactional/Analytical Processing)
- **YugabyteDB**: PostgreSQL-compatible, multi-cloud
- **Google Spanner**: Globally distributed, external consistency

### NoSQL Databases
```
Document Database (MongoDB):
{
  "_id": "user123",
  "username": "john_doe",
  "email": "john@example.com",
  "posts": [
    {"title": "First Post", "content": "Hello"},
    {"title": "Second Post", "content": "World"}
  ],
  "preferences": {
    "theme": "dark",
    "notifications": true
  }
}

Key-Value Database (Redis):
user:123 → {"username": "john", "email": "john@ex.com"}
post:456 → {"title": "Hello", "content": "World"}

Column-Family Database (Cassandra):
Row Key: user123
┌─────────────┬─────────────┬─────────────┐
│ username    │ email       │ posts       │
│ john_doe    │ john@ex.com │ [1,2,3]     │
└─────────────┴─────────────┴─────────────┘

Time-Series Database (InfluxDB):
measurement: cpu_usage
tags: host=server1, region=us-east
fields: value=85.5
timestamp: 1640995200

Vector Database (Pinecone):
vectors: [
  [0.1, 0.2, 0.3, ...],  // Embedding for "cat"
  [0.4, 0.5, 0.6, ...],  // Embedding for "dog"
]
metadata: {"text": "cat", "category": "animal"}
```

**Types:**
- **Document**: MongoDB, Couchbase, DynamoDB (document mode)
- **Key-Value**: Redis, Riak, etcd
- **Column-Family**: Cassandra, HBase, ScyllaDB
- **Graph**: Neo4j, Amazon Neptune, JanusGraph
- **Time-Series**: InfluxDB, TimescaleDB, Prometheus
- **Vector**: Pinecone, Weaviate, Milvus (for AI/ML embeddings)

## Database Selection Criteria

### Use SQL When:
- **Data Structure**: Well-defined, consistent schema
- **ACID Requirements**: Strong consistency needed
- **Complex Queries**: Joins, aggregations, complex operations
- **Data Integrity**: Foreign keys, constraints important
- **Mature Technology**: Proven solutions, extensive tooling

**Examples:**
- Financial systems (transactions)
- E-commerce (orders, inventory)
- HR systems (employee data)
- Analytics (structured reporting)

### Use NoSQL When:
- **Flexible Schema**: Evolving data structure
- **High Scalability**: Horizontal scaling required
- **High Performance**: Low latency, high throughput
- **Unstructured Data**: Documents, key-value pairs
- **Cloud Native**: Designed for distributed systems

**Examples:**
- Social media feeds
- IoT data collection
- Real-time analytics
- Content management systems

## Database Design Principles

### Normalization
```
1NF (First Normal Form):
- Each cell contains atomic values
- No repeating groups

2NF (Second Normal Form):
- 1NF + No partial dependencies
- All non-key attributes depend on entire primary key

3NF (Third Normal Form):
- 2NF + No transitive dependencies
- Non-key attributes don't depend on other non-key attributes
```

**Example:**
```sql
-- Unnormalized
Orders(order_id, customer_name, customer_address, product_name, product_price, quantity)

-- 3NF Normalized
Customers(customer_id, customer_name, customer_address)
Products(product_id, product_name, product_price)
Orders(order_id, customer_id, order_date)
Order_Items(order_id, product_id, quantity, price)
```

### Denormalization
```
Purpose: Improve read performance at cost of write complexity

Normalized:
Users(user_id, name, email)
Posts(post_id, user_id, title, content)
Comments(comment_id, post_id, user_id, content)

Denormalized:
Posts(post_id, user_id, user_name, title, content, comment_count)
```

**When to Denormalize:**
- Read-heavy workloads
- Performance-critical queries
- Reporting and analytics
- Caching layers

## Indexing Strategies

### B-Tree Index
```
Structure:
        [Root]
       /      \
   [Internal] [Internal]
   /    \      /    \
[Leaf] [Leaf] [Leaf] [Leaf]

Use Cases:
- Equality queries (=, IN)
- Range queries (>, <, BETWEEN)
- Sort operations
- Prefix searches
```

### Hash Index
```
Structure:
Hash Function → Bucket → Data Pointer

Use Cases:
- Equality queries only
- Memory-based tables
- Fast lookups
- No range queries
```

### Composite Index
```sql
-- Single column index
CREATE INDEX idx_email ON users(email);

-- Composite index
CREATE INDEX idx_user_status_date ON users(status, created_at);

-- Covering index
CREATE INDEX idx_post_details ON posts(user_id, created_at, title);
```

**Index Selection Strategy:**
1. **Identify Query Patterns**: Most frequent queries
2. **Analyze WHERE Clauses**: Filter conditions
3. **Consider JOIN Columns**: Foreign key relationships
4. **Evaluate ORDER BY**: Sorting requirements
5. **Monitor Performance**: Query execution plans

## Database Scaling

### Vertical Scaling (Scale Up)
```
Single Server:
┌───────────────────────────────────┐
│     CPU: 16 cores                 │
│     RAM: 64GB                     │
│     Storage: 1TB SSD              │
└───────────────────────────────────┘

Scale Up:
┌───────────────────────────────────┐
│     CPU: 32 cores                 │
│     RAM: 128GB                    │
│     Storage: 2TB SSD              │
└───────────────────────────────────┘
```

**Pros:**
- Simple implementation
- No application changes
- Strong consistency
- Lower complexity

**Cons:**
- Physical limits
- Single point of failure
- Expensive scaling
- Vendor lock-in

### Horizontal Scaling (Scale Out)
```
Multiple Servers:
Server 1     Server 2     Server 3
┌─────────────┐  ┌─────────────┐  ┌────────────┐
│ CPU: 8      │  │ CPU: 8      │  │ CPU: 8     │
│ RAM: 32GB   │  │ RAM: 32GB   │  │ RAM: 32GB  │
│ 256GB SSD   │  │ 256GB SSD   │  │ 256GB SSD  │
└─────────────┘  └─────────────┘  └────────────┘
```

**Pros:**
- Unlimited scaling
- Better fault tolerance
- Cost-effective
- Geographic distribution

**Cons:**
- Complex architecture
- Consistency challenges
- Network overhead
- Operational complexity

## Database Replication

### Master-Slave Replication
```
Master (Write)
    ↓
┌─────────────────┐
│   Write Query   │
│   Binary Log    │
└─────────────────┘
    ↓
Slave 1 (Read)  Slave 2 (Read)  Slave 3 (Read)
```

**Implementation:**
```sql
-- Master configuration
server-id = 1
log-bin = mysql-bin
binlog-format = ROW

-- Slave configuration
server-id = 2
relay-log = relay-bin
read-only = 1

-- Start replication
CHANGE MASTER TO
  MASTER_HOST='master-ip',
  MASTER_USER='replication-user',
  MASTER_PASSWORD='password',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=154;

START SLAVE;
```

### Master-Master Replication
```
Master 1 ↔ Master 2
    ↓         ↓
Slave 1    Slave 2
```

**Challenges:**
- Conflict resolution
- Data consistency
- Write conflicts
- Complex topology

## Database Sharding

### Horizontal Sharding
```
Users Table (Sharded by user_id):
Shard 0: user_id % 4 = 0
Shard 1: user_id % 4 = 1
Shard 2: user_id % 4 = 2
Shard 3: user_id % 4 = 3

Query Routing:
SELECT * FROM users WHERE user_id = 123;
→ Shard 3 (123 % 4 = 3)
```

### Vertical Sharding
```
Single Database:
┌─────────────────┐
│   Users         │
│   Posts         │
│   Comments      │
│   Likes         │
└─────────────────┘

Vertical Shards:
Users DB        Posts DB
┌─────────┐    ┌─────────┐
│ Users   │    │ Posts   │
└─────────┘    └─────────┘

Comments DB    Likes DB
┌─────────┐    ┌─────────┐
│Comments │    │ Likes   │
└─────────┘    └─────────┘
```

### Sharding Strategies

#### Range-based Sharding
```python
def get_shard_range(user_id):
    if user_id < 1000000:
        return "shard_0"
    elif user_id < 2000000:
        return "shard_1"
    elif user_id < 3000000:
        return "shard_2"
    else:
        return "shard_3"
```

#### Hash-based Sharding
```python
def get_shard_hash(user_id, num_shards=4):
    return hash(user_id) % num_shards

def get_shard_connection(user_id):
    shard_id = get_shard_hash(user_id)
    return connections[f"shard_{shard_id}"]
```

#### Directory-based Sharding
```python
class ShardDirectory:
    def __init__(self):
        self.shard_map = {}

    def add_user_shard(self, user_id, shard_id):
        self.shard_map[user_id] = shard_id

    def get_user_shard(self, user_id):
        return self.shard_map.get(user_id)

    def move_user(self, user_id, new_shard_id):
        old_shard_id = self.shard_map[user_id]
        # Move data from old to new shard
        self.move_data(user_id, old_shard_id, new_shard_id)
        self.shard_map[user_id] = new_shard_id
```

## Database Performance Optimization

### Query Optimization
```sql
-- Slow query
SELECT * FROM orders WHERE customer_id = 12345;

-- Optimized with index
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- Explain plan
EXPLAIN SELECT * FROM orders WHERE customer_id = 12345;

-- Covering index
CREATE INDEX idx_orders_covering ON orders(customer_id, order_date, total);
```

### Connection Pooling
```python
import psycopg2
from psycopg2 import pool

class DatabasePool:
    def __init__(self, min_conn=1, max_conn=20):
        self.pool = psycopg2.pool.SimpleConnectionPool(
            minconn=min_conn,
            maxconn=max_conn,
            host="localhost",
            database="myapp",
            user="user",
            password="password"
        )

    def get_connection(self):
        return self.pool.getconn()

    def return_connection(self, conn):
        self.pool.putconn(conn)

    def execute_query(self, query, params=None):
        conn = self.get_connection()
        try:
            cursor = conn.cursor()
            cursor.execute(query, params)
            result = cursor.fetchall()
            return result
        finally:
            self.return_connection(conn)

# Usage
db_pool = DatabasePool()
result = db_pool.execute_query("SELECT * FROM users WHERE id = %s", (123,))
```

### Caching Layer
```python
import redis
import json
from datetime import timedelta

class DatabaseWithCache:
    def __init__(self, db_connection, redis_client):
        self.db = db_connection
        self.cache = redis_client
        self.default_ttl = 3600  # 1 hour

    def get_user(self, user_id):
        cache_key = f"user:{user_id}"

        # Try cache first
        cached_user = self.cache.get(cache_key)
        if cached_user:
            return json.loads(cached_user)

        # Cache miss - query database
        user = self.db.execute_query(
            "SELECT * FROM users WHERE id = %s",
            (user_id,)
        )

        if user:
            # Cache the result
            self.cache.setex(
                cache_key,
                self.default_ttl,
                json.dumps(user[0])
            )
            return user[0]

        return None

    def update_user(self, user_id, user_data):
        # Update database
        self.db.execute_query(
            "UPDATE users SET name = %s, email = %s WHERE id = %s",
            (user_data['name'], user_data['email'], user_id)
        )

        # Invalidate cache
        cache_key = f"user:{user_id}"
        self.cache.delete(cache_key)
```

## Database Migration Strategies

### Schema Migration
```python
# Alembic (SQLAlchemy migration tool)
from alembic import op
import sqlalchemy as sa

def upgrade():
    # Add new column
    op.add_column('users', sa.Column('phone', sa.String(20), nullable=True))

    # Create index
    op.create_index('idx_users_phone', 'users', ['phone'])

    # Migrate existing data
    op.execute("""
        UPDATE users SET phone = NULL WHERE phone IS NULL
    """)

def downgrade():
    # Remove column
    op.drop_column('users', 'phone')
```

**Migration Patterns:**
- **Versioned Migrations**: Track schema changes with versions
- **Blue-Green Deployment**: Deploy new schema alongside old
- **Canary Releases**: Gradual rollout with rollback capability
- **Feature Flags**: Enable/disable new features independently

### Data Migration
```python
class DataMigration:
    def __init__(self, source_db, target_db):
        self.source = source_db
        self.target = target_db
        self.batch_size = 1000

    def migrate_users_table(self):
        """Migrate users table with transformation"""
        offset = 0

        while True:
            # Fetch batch from source
            users = self.source.execute_query("""
                SELECT * FROM users ORDER BY id
                LIMIT ? OFFSET ?
            """, (self.batch_size, offset))

            if not users:
                break

            # Transform data
            transformed_users = []
            for user in users:
                transformed_user = {
                    'id': user['id'],
                    'username': user['username'].lower(),  # Transform
                    'email': user['email'],
                    'created_at': user['created_at']
                }
                transformed_users.append(transformed_user)

            # Insert batch to target
            self.target.bulk_insert('users', transformed_users)

            offset += self.batch_size

            # Log progress
            print(f"Migrated {offset} users")

    def validate_migration(self):
        """Validate data integrity after migration"""
        source_count = self.source.execute_query("SELECT COUNT(*) FROM users")[0][0]
        target_count = self.target.execute_query("SELECT COUNT(*) FROM users")[0][0]

        if source_count != target_count:
            raise ValueError(f"Count mismatch: {source_count} vs {target_count}")

        print("Migration validation successful")
```

**Migration Strategies:**
- **Big Bang**: Migrate everything at once (high risk)
- **Phased Migration**: Migrate in stages (lower risk)
- **Parallel Run**: Run old and new systems simultaneously
- **Strangler Pattern**: Gradually replace old system

## Modern Database Patterns

### Database as a Service (DBaaS)
```
Cloud Database Services:
┌────────────────────────────────────────────────────────┐
│                    AWS RDS                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   Aurora    │  │   MySQL     │  │ PostgreSQL  │     │
│  │   (MySQL)   │  │             │  │             │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│                                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   DynamoDB  │  │   Redshift  │  │   Neptune   │     │
│  │   (NoSQL)   │  │   (DW)      │  │   (Graph)   │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└────────────────────────────────────────────────────────┘
```

**Benefits:**
- **Managed Service**: No infrastructure management
- **Auto-Scaling**: Automatic capacity adjustment
- **High Availability**: Built-in redundancy
- **Security**: Enterprise-grade security features
- **Monitoring**: Built-in metrics and alerting

### Multi-Cloud Database Strategy
```
Hybrid Cloud Architecture:
┌────────────────────────────────────────────────────────┐
│                    Application                         │
│                                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   AWS       │  │   Azure     │  │   GCP       │     │
│  │   Aurora    │  │   Database  │  │   Cloud SQL │     │
│  │             │  │             │  │             │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│                                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   Read      │  │   Read      │  │   Read      │     │
│  │   Replica   │  │   Replica   │  │   Replica   │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└────────────────────────────────────────────────────────┘
```

**Strategies:**
- **Active-Active**: Read/write to multiple clouds
- **Active-Passive**: One primary, others backup
- **Data Replication**: Cross-cloud data synchronization
- **Traffic Routing**: Global load balancing

### Data Mesh Architecture
```
Domain-Oriented Data Architecture:
┌────────────────────────────────────────────────────────┐
│                    Data Mesh                           │
│                                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ User Domain │  │ Order       │  │ Product     │     │
│  │ Data Product│  │ Domain      │  │ Domain      │     │
│  │             │  │ Data        │  │ Data        │     │
│  │ ┌─────────┐ │  │ Product     │  │ Product     │     │
│  │ │ API     │ │  │             │  │             │     │
│  │ │ Schema  │ │  │             │  │             │     │
│  │ │ Quality │ │  │             │  │             │     │
│  │ └─────────┘ │  └─────────────┘  └─────────────┘     │
│  └─────────────┘                                       │
│                                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ Data        │  │ Self-serve  │  │ Federated   │     │
│  │ Catalog     │  │ Analytics   │  │ Governance  │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└────────────────────────────────────────────────────────┘
```

**Principles:**
- **Domain Ownership**: Teams own their data
- **Data as Product**: Treat data as product with APIs
- **Self-Serve Platform**: Enable data discovery and access
- **Federated Governance**: Distributed data governance

## Database Security

### Authentication and Authorization
```sql
-- Create users with specific permissions
CREATE USER 'app_user'@'%' IDENTIFIED BY 'secure_password';
CREATE USER 'readonly_user'@'%' IDENTIFIED BY 'readonly_password';

-- Grant specific permissions
GRANT SELECT, INSERT, UPDATE ON myapp.users TO 'app_user'@'%';
GRANT SELECT ON myapp.* TO 'readonly_user'@'%';

-- Row-level security
CREATE POLICY user_policy ON users
    FOR ALL TO app_user
    USING (user_id = current_user_id());

ALTER TABLE users ENABLE ROW LEVEL SECURITY;
```

### Encryption
```python
from cryptography.fernet import Fernet

class EncryptedDatabase:
    def __init__(self, db_connection, encryption_key):
        self.db = db_connection
        self.cipher = Fernet(encryption_key)

    def encrypt_sensitive_data(self, data):
        return self.cipher.encrypt(data.encode()).decode()

    def decrypt_sensitive_data(self, encrypted_data):
        return self.cipher.decrypt(encrypted_data.encode()).decode()

    def insert_user(self, user_data):
        # Encrypt sensitive fields
        encrypted_email = self.encrypt_sensitive_data(user_data['email'])
        encrypted_phone = self.encrypt_sensitive_data(user_data['phone'])

        query = """
        INSERT INTO users (name, email, phone, created_at)
        VALUES (%s, %s, %s, NOW())
        """

        self.db.execute_query(query, (
            user_data['name'],
            encrypted_email,
            encrypted_phone
        ))

    def get_user(self, user_id):
        query = "SELECT * FROM users WHERE id = %s"
        result = self.db.execute_query(query, (user_id,))

        if result:
            user = result[0]
            # Decrypt sensitive fields
            user['email'] = self.decrypt_sensitive_data(user['email'])
            user['phone'] = self.decrypt_sensitive_data(user['phone'])
            return user

        return None
```

## Database Monitoring

### Key Metrics
```python
import time
import psycopg2
from prometheus_client import Counter, Histogram, Gauge

# Metrics
QUERY_COUNT = Counter('db_queries_total', 'Total database queries', ['query_type'])
QUERY_DURATION = Histogram('db_query_duration_seconds', 'Database query duration')
ACTIVE_CONNECTIONS = Gauge('db_active_connections', 'Active database connections')

class MonitoredDatabase:
    def __init__(self, connection_string):
        self.connection_string = connection_string
        self.connection = None

    def connect(self):
        self.connection = psycopg2.connect(self.connection_string)
        ACTIVE_CONNECTIONS.inc()

    def execute_query(self, query, params=None, query_type='select'):
        start_time = time.time()

        try:
            cursor = self.connection.cursor()
            cursor.execute(query, params)
            result = cursor.fetchall()

            # Record metrics
            QUERY_COUNT.labels(query_type=query_type).inc()
            QUERY_DURATION.observe(time.time() - start_time)

            return result
        except Exception as e:
            QUERY_COUNT.labels(query_type='error').inc()
            raise e

    def get_connection_stats(self):
        query = """
        SELECT
            count(*) as active_connections,
            count(*) FILTER (WHERE state = 'active') as active_queries
        FROM pg_stat_activity
        """
        return self.execute_query(query)[0]
```

## Real-World Examples

### Facebook Database Architecture
```
Tiers:
- Web Tier: Application servers
- Cache Tier: Memcached, TAO
- Database Tier: MySQL (sharded)
- Storage Tier: Haystack (photos)

Sharding Strategy:
- User data sharded by user_id
- Each shard handles ~1M users
- Read replicas for scaling reads
- Multi-region replication
```

### Uber Database Architecture
```
Services:
- Trips Service: PostgreSQL
- Users Service: MySQL
- Payments Service: PostgreSQL
- Geospatial: PostgreSQL + PostGIS
- Real-time: Redis

Replication:
- Master-slave for reads
- Multi-master for writes
- Cross-region replication
- Automatic failover
```

### Netflix Database Architecture
```
Data Stores:
- User data: Cassandra (NoSQL)
- Catalog: Elasticsearch
- Recommendations: DynamoDB
- Analytics: S3 + Redshift

Strategy:
- Polyglot persistence
- Event-driven architecture
- Microservices per domain
- Global distribution
```

## Interview Tips

### Common Questions
1. "SQL vs NoSQL vs NewSQL - when to use which?"
2. "How would you scale a database to handle millions of users?"
3. "What are database indexes and how do they work?"
4. "Explain ACID vs BASE properties"
5. "How would you handle database migrations in production?"
6. "What's the difference between replication and sharding?"
7. "How do you choose between different NoSQL types?"

### Answer Framework
1. **Understand Requirements**: Data structure, scale, consistency, query patterns
2. **Choose Database Type**: SQL vs NoSQL vs NewSQL vs specialized databases
3. **Design Schema**: Normalization/denormalization, relationships, indexes
4. **Plan Scaling**: Replication, sharding, caching, multi-cloud strategies
5. **Handle Operations**: Migrations, monitoring, backup/recovery
6. **Consider Trade-offs**: Performance vs consistency vs complexity vs cost

### Key Points to Emphasize
- Business requirements drive database choice
- Data consistency and query patterns are critical
- Scaling strategies depend on read/write ratios
- Migration and operational concerns are important
- Security and compliance requirements
- Cost optimization in cloud environments
- Modern patterns like data mesh and DBaaS

## Practice Problems

1. **Design database for e-commerce platform with microservices**
2. **Scale social media database to 1B users with multi-region**
3. **Design banking transaction system with ACID compliance**
4. **Create real-time analytics database with time-series data**
5. **Design multi-tenant SaaS database with data isolation**
6. **Implement database migration strategy for zero-downtime deployment**
7. **Design IoT data platform with time-series and document databases**
8. **Create AI/ML feature store with vector database**

## Further Reading

- **Books**: "Database System Concepts" by Silberschatz, "Designing Data-Intensive Applications" by Kleppmann
- **Papers**: "The Google File System", "Dynamo: Amazon's Highly Available Key-value Store"
- **Documentation**: PostgreSQL Documentation, MongoDB Manual

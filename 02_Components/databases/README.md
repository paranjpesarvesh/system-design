# Database Systems Deep Dive

## Advanced Database Concepts

### Database Normalization Forms

#### First Normal Form (1NF)
```
Unnormalized:
┌─────────────────────────────────────┐
│ Order_ID │ Product_IDs │ Quantities │
├─────────────────────────────────────┤
│ 1        │ 1,2,3        │ 2,1,3     │
│ 2        │ 4,5          │ 1,2       │
└─────────────────────────────────────┘

1NF Normalized:
┌───────────────────────────────────┐
│ Order_ID │ Product_ID │ Quantity  │
├───────────────────────────────────┤
│ 1        │ 1          │ 2         │
│ 1        │ 2          │ 1         │
│ 1        │ 3          │ 3         │
│ 2        │ 4          │ 1         │
│ 2        │ 5          │ 2         │
└───────────────────────────────────┘
```

#### Second Normal Form (2NF)
```
1NF but not 2NF:
┌─────────────────────────────────────────────────┐
│ Order_ID │ Product_ID │ Quantity │ Product_Name │
├─────────────────────────────────────────────────┤
│ 1        │ 1          │ 2         │ "Laptop"    │
│ 1        │ 2          │ 1         │ "Mouse"     │
│ 2        │ 1          │ 1         │ "Laptop"    │
│ 2        │ 3          │ 3         │ "Keyboard"  │
└─────────────────────────────────────────────────┘

2NF Normalized:
Products:
┌─────────────────────────┐
│ Product_ID │ Name       │
├─────────────────────────┤
│ 1          │ "Laptop"   │
│ 2          │ "Mouse"    │
│ 3          │ "Keyboard" │
└─────────────────────────┘

Order_Items:
┌───────────────────────────────────┐
│ Order_ID │ Product_ID │ Quantity  │
├───────────────────────────────────┤
│ 1        │ 1          │ 2         │
│ 1        │ 2          │ 1         │
│ 2        │ 1          │ 1         │
│ 2        │ 3          │ 3         │
└───────────────────────────────────┘
```

#### Third Normal Form (3NF)
```
2NF but not 3NF:
┌─────────────────────────────────────────────┐
│ Employee_ID │ Department_ID │ Dept_Name     │
├─────────────────────────────────────────────┤
│ 1           │ 10            │ "Sales"       │
│ 2           │ 10            │ "Sales"       │
│ 3           │ 20            │ "IT"          │
│ 4           │ 20            │ "IT"          │
└─────────────────────────────────────────────┘

3NF Normalized:
Departments:
┌─────────────────────┐
│ Dept_ID │ Name      │
├─────────────────────┤
│ 10      │ "Sales"   │
│ 20      │ "IT"      │
└─────────────────────┘

Employees:
┌─────────────────────────┐
│ Employee_ID │ Dept_ID   │
├─────────────────────────┤
│ 1           │ 10        │
│ 2           │ 10        │
│ 3           │ 20        │
│ 4           │ 20        │
└─────────────────────────┘
```

### Database Indexing Strategies

#### Composite Indexes
```sql
-- Single column indexes
CREATE INDEX idx_email ON users(email);
CREATE INDEX idx_created_at ON posts(created_at);

-- Composite index for common query patterns
CREATE INDEX idx_user_status_date ON users(status, created_at);
CREATE INDEX idx_post_user_date ON posts(user_id, created_at DESC);

-- Covering index (includes all columns needed for query)
CREATE INDEX idx_order_customer_status ON orders(customer_id, status)
INCLUDE (order_date, total_amount);

-- Partial index for frequently accessed subset
CREATE INDEX idx_active_users ON users(id) WHERE status = 'active';

-- Expression index for computed values
CREATE INDEX idx_lower_email ON users(LOWER(email));
```

#### Index Usage Analysis
```sql
-- Analyze index usage
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'john@example.com';

-- Index statistics
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan DESC;

-- Unused indexes
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
AND idx_scan = 0;
```

### Query Optimization

#### Query Execution Plans
```sql
-- Simple query with EXPLAIN
EXPLAIN SELECT * FROM users WHERE email = 'john@example.com';

-- Detailed execution plan with ANALYZE
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.*, p.title
FROM users u
JOIN posts p ON u.id = p.user_id
WHERE u.status = 'active'
AND p.created_at >= '2024-01-01'
ORDER BY p.created_at DESC
LIMIT 10;

-- Common execution plan operators:
-- Seq Scan: Sequential table scan
-- Index Scan: Using index to find rows
-- Index Only Scan: Using index only (covering index)
-- Bitmap Heap Scan: Using bitmap index
-- Hash Join: Hash-based join
-- Nested Loop: Nested loop join
-- Merge Join: Sorted merge join
```

#### Query Optimization Techniques
```sql
-- 1. Proper JOIN order (small table first)
SELECT /*+ LEADING(u p) */ *
FROM users u
JOIN posts p ON u.id = p.user_id
WHERE u.status = 'active';

-- 2. Use EXISTS instead of IN for subqueries
SELECT u.*
FROM users u
WHERE EXISTS (
    SELECT 1 FROM posts p
    WHERE p.user_id = u.id
    AND p.status = 'published'
);

-- 3. Avoid SELECT *
SELECT id, username, email, created_at
FROM users
WHERE status = 'active';

-- 4. Use appropriate data types
-- Use VARCHAR instead of TEXT for short strings
-- Use TIMESTAMP instead of VARCHAR for dates
-- Use INTEGER instead of VARCHAR for IDs

-- 5. Pagination optimization
-- Bad: OFFSET for large offsets
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 10000;

-- Good: Keyset pagination
SELECT * FROM posts
WHERE created_at < '2024-01-01 12:00:00'
ORDER BY created_at DESC
LIMIT 20;
```

## Advanced Database Features

### Window Functions
```sql
-- Running total
SELECT
    order_date,
    customer_id,
    amount,
    SUM(amount) OVER (
        PARTITION BY customer_id
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM orders;

-- Row number
SELECT
    employee_id,
    salary,
    ROW_NUMBER() OVER (
        PARTITION BY department_id
        ORDER BY salary DESC
    ) AS salary_rank
FROM employees;

-- Moving average
SELECT
    date,
    sales,
    AVG(sales) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7_days
FROM daily_sales;

-- LAG/LEAD functions
SELECT
    date,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY date) AS prev_day_revenue,
    LEAD(revenue, 1) OVER (ORDER BY date) AS next_day_revenue
FROM daily_revenue;
```

### Common Table Expressions (CTEs)
```sql
-- Simple CTE
WITH active_users AS (
    SELECT id, username, email
    FROM users
    WHERE status = 'active'
)
SELECT
    a.username,
    p.title,
    p.created_at
FROM active_users a
JOIN posts p ON a.id = p.user_id;

-- Recursive CTE for hierarchical data
WITH RECURSIVE employee_hierarchy AS (
    -- Base case: top-level managers
    SELECT
        id,
        name,
        manager_id,
        1 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive case: subordinates
    SELECT
        e.id,
        e.name,
        e.manager_id,
        eh.level + 1 AS level
    FROM employees e
    JOIN employee_hierarchy eh ON e.manager_id = eh.id
)
SELECT
    id,
    name,
    level,
    REPEAT('  ', level - 1) || name AS formatted_name
FROM employee_hierarchy
ORDER BY level, name;
```

### Materialized Views
```sql
-- Create materialized view
CREATE MATERIALIZED VIEW user_post_stats AS
SELECT
    u.id AS user_id,
    u.username,
    COUNT(p.id) AS post_count,
    MAX(p.created_at) AS last_post_date
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
GROUP BY u.id, u.username;

-- Refresh materialized view
REFRESH MATERIALIZED VIEW user_post_stats;

-- Concurrent refresh (allows queries during refresh)
REFRESH MATERIALIZED VIEW CONCURRENTLY user_post_stats;

-- Create unique index for fast lookups
CREATE UNIQUE INDEX idx_user_post_stats_user_id
ON user_post_stats(user_id);

-- Query materialized view
SELECT * FROM user_post_stats
WHERE post_count > 10
ORDER BY post_count DESC;
```

## Database Performance Tuning

### Configuration Optimization

#### PostgreSQL Configuration
```postgresql
# Memory settings
shared_buffers = 256MB                    # 25% of RAM
effective_cache_size = 1GB                  # 75% of RAM
work_mem = 4MB                           # Per operation memory
maintenance_work_mem = 64MB                # Maintenance operations

# Connection settings
max_connections = 200
shared_preload_libraries = 'pg_stat_statements'

# WAL settings
wal_buffers = 16MB
checkpoint_completion_target = 0.9
wal_writer_delay = 200ms

# Query planner settings
random_page_cost = 1.1                  # SSD optimization
effective_io_concurrency = 200             # SSD concurrent I/O

# Logging
log_min_duration_statement = 1000          # Log slow queries
log_checkpoints = on
log_connections = on
log_disconnections = on
```

#### MySQL Configuration
```ini
[mysqld]
# Memory settings
innodb_buffer_pool_size = 2G              # 70-80% of RAM
innodb_log_file_size = 256M
innodb_log_buffer_size = 16M
key_buffer_size = 32M
sort_buffer_size = 2M

# Connection settings
max_connections = 300
max_connect_errors = 1000

# InnoDB settings
innodb_file_per_table = 1
innodb_flush_log_at_trx_commit = 2
innodb_flush_method = O_DIRECT

# Query cache
query_cache_type = 1
query_cache_size = 64M
query_cache_limit = 2M

# Slow query log
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2
```

### Connection Pooling

#### PgBouncer Configuration
```ini
[databases]
myapp = host=localhost port=5432 dbname=myapp

[pgbouncer]
listen_port = 6432
listen_addr = 127.0.0.1
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
logfile = /var/log/pgbouncer/pgbouncer.log
pidfile = /var/run/pgbouncer/pgbouncer.pid
admin_users = stats
stats_users = stats, pgbouncer

# Pool settings
pool_mode = transaction
max_client_conn = 200
default_pool_size = 20
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 5
max_db_connections = 50
max_user_connections = 50

# Timeouts
server_reset_query = DISCARD ALL
server_check_delay = 30
server_check_query = select 1
server_lifetime = 3600
server_idle_timeout = 600
```

#### Application Connection Pool
```python
import psycopg2
from psycopg2 import pool
from contextlib import contextmanager

class DatabasePool:
    def __init__(self, min_conn=5, max_conn=20, **db_kwargs):
        self.pool = psycopg2.pool.ThreadedConnectionPool(
            minconn=min_conn,
            maxconn=max_conn,
            **db_kwargs
        )

    @contextmanager
    def get_connection(self):
        conn = self.pool.getconn()
        try:
            yield conn
        finally:
            self.pool.putconn(conn)

    def close_all(self):
        self.pool.closeall()

# Usage
db_pool = DatabasePool(
    host='localhost',
    database='myapp',
    user='user',
    password='password',
    min_conn=10,
    max_conn=50
)

def get_user_posts(user_id):
    with db_pool.get_connection() as conn:
        with conn.cursor() as cursor:
            cursor.execute("""
                SELECT p.*, u.username
                FROM posts p
                JOIN users u ON p.user_id = u.id
                WHERE p.user_id = %s
                ORDER BY p.created_at DESC
                LIMIT 10
            """, (user_id,))

            return cursor.fetchall()
```

## Database Replication

### Master-Slave Replication Setup

#### PostgreSQL Streaming Replication
```bash
# Master configuration (postgresql.conf)
wal_level = replica
max_wal_senders = 3
wal_keep_segments = 64
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/archive/%f'

# pg_hba.conf (allow replication)
host    replication     replicator     10.0.0.2/32     md5

# Create replication user
CREATE USER replicator REPLICATION LOGIN CONNECTION LIMIT 3 ENCRYPTED PASSWORD 'replicator_password';

# Slave setup
pg_basebackup -h master_ip -D /var/lib/postgresql/data -U replicator -v -P -W

# recovery.conf (on slave)
standby_mode = 'on'
primary_conninfo = 'host=master_ip port=5432 user=replicator'
trigger_file = '/tmp/postgresql.trigger'
```

#### MySQL Replication
```bash
# Master configuration (my.cnf)
[mysqld]
server-id = 1
log-bin = mysql-bin
binlog-format = ROW
binlog-do-db = myapp
expire_logs_days = 7
max_binlog_size = 100M

# Create replication user
CREATE USER 'repl_user'@'%' IDENTIFIED BY 'repl_password';
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'%';

# Get master position
FLUSH TABLES WITH READ LOCK;
SHOW MASTER STATUS;
UNLOCK TABLES;

# Slave configuration
[mysqld]
server-id = 2
relay-log = relay-bin
read-only = 1

# Configure slave
CHANGE MASTER TO
    MASTER_HOST='master_ip',
    MASTER_USER='repl_user',
    MASTER_PASSWORD='repl_password',
    MASTER_LOG_FILE='mysql-bin.000001',
    MASTER_LOG_POS=154;

START SLAVE;
```

### Database Clustering

#### PostgreSQL Patroni Cluster
```yaml
# patroni.yml
restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.0.1:8008

etcd:
  hosts: 10.0.0.10:2379,10.0.0.11:2379,10.0.0.12:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
  pg_hba:
    - host all all 0.0.0.0/0 md5
  users:
    admin:
      password: admin
      options:
        - createrole
        - createdb

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.0.1:5432
  data_dir: /var/lib/postgresql/data
  pgpass: /tmp/pgpass
  authentication:
    replication:
      username: replicator
      password: replicator_password
    superuser:
      username: postgres
      password: postgres
  parameters:
    max_connections: 200
    shared_buffers: 256MB
    effective_cache_size: 1GB
    work_mem: 4MB
    maintenance_work_mem: 64MB
    wal_level: replica
    max_wal_senders: 5
    wal_keep_segments: 32
    hot_standby: on
    max_replication_slots: 5
    wal_log_hints: on
```

#### MySQL Group Replication
```ini
# Primary node configuration
[mysqld]
server-id = 1
gtid_mode = ON
enforce_gtid_consistency = ON
log_bin = mysql-bin
log_slave_updates = ON
binlog_format = ROW
binlog_checksum = NONE

# Replica node configuration
[mysqld]
server-id = 2
gtid_mode = ON
enforce_gtid_consistency = ON
log_bin = mysql-bin
log_slave_updates = ON
binlog_format = ROW
binlog_checksum = NONE
read_only = ON

# Configure group replication
-- On primary
SET GLOBAL group_replication_bootstrap_group = ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group = OFF;

-- On replicas
CHANGE MASTER TO
    MASTER_HOST='primary_ip',
    MASTER_PORT=3306,
    MASTER_USER='repl_user',
    MASTER_PASSWORD='repl_password'
    FOR CHANNEL 'group_replication_recovery';

START GROUP_REPLICATION;
```

## Database Monitoring

### Performance Monitoring Queries

#### PostgreSQL Monitoring
```sql
-- Active connections
SELECT
    count(*) AS total_connections,
    count(*) FILTER (WHERE state = 'active') AS active_connections,
    count(*) FILTER (WHERE state = 'idle') AS idle_connections
FROM pg_stat_activity;

-- Long-running queries
SELECT
    pid,
    now() - pg_stat_activity.query_start AS duration,
    query,
    state
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes'
ORDER BY duration DESC;

-- Table sizes
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) -
                   pg_relation_size(schemaname||'.'||tablename)) AS index_size
FROM pg_tables
WHERE schemaname NOT IN ('information_schema', 'pg_catalog')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Index usage
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Cache hit ratio
SELECT
    sum(heap_blks_read) AS heap_read,
    sum(heap_blks_hit) AS heap_hit,
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) AS ratio
FROM pg_stat_database
WHERE datname = 'myapp';
```

#### MySQL Monitoring
```sql
-- Process list
SELECT
    id,
    user,
    host,
    db,
    command,
    time,
    state,
    info
FROM information_schema.processlist
WHERE time > 5
ORDER BY time DESC;

-- Table sizes
SELECT
    table_schema,
    table_name,
    ROUND(((data_length + index_length) / 1024 / 1024), 2) AS size_mb,
    ROUND((data_length / 1024 / 1024), 2) AS data_mb,
    ROUND((index_length / 1024 / 1024), 2) AS index_mb
FROM information_schema.tables
WHERE table_schema NOT IN ('information_schema', 'mysql', 'performance_schema')
ORDER BY (data_length + index_length) DESC;

-- InnoDB buffer pool usage
SELECT
    variable_name,
    variable_value
FROM performance_schema.global_status
WHERE variable_name IN (
    'Innodb_buffer_pool_pages_data',
    'Innodb_buffer_pool_pages_dirty',
    'Innodb_buffer_pool_pages_total',
    'Innodb_buffer_pool_read_requests',
    'Innodb_buffer_pool_reads'
);

-- Query cache statistics
SHOW STATUS LIKE 'Qcache%';
```

### Database Metrics Collection
```python
import psycopg2
import time
import json
from prometheus_client import Gauge, Counter, Histogram

# Prometheus metrics
DB_CONNECTIONS = Gauge('db_connections_total', 'Total database connections')
DB_QUERY_DURATION = Histogram('db_query_duration_seconds', 'Database query duration')
DB_SLOW_QUERIES = Counter('db_slow_queries_total', 'Total slow queries')

class DatabaseMonitor:
    def __init__(self, connection_string):
        self.connection_string = connection_string
        self.metrics = {}

    def collect_metrics(self):
        conn = psycopg2.connect(self.connection_string)

        try:
            with conn.cursor() as cursor:
                # Active connections
                cursor.execute("""
                    SELECT count(*) FROM pg_stat_activity
                    WHERE state = 'active'
                """)
                active_connections = cursor.fetchone()[0]
                DB_CONNECTIONS.set(active_connections)

                # Database size
                cursor.execute("""
                    SELECT pg_size_pretty(pg_database_size(current_database()))
                """)
                db_size = cursor.fetchone()[0]

                # Cache hit ratio
                cursor.execute("""
                    SELECT
                        sum(heap_blks_hit)::float /
                        (sum(heap_blks_hit) + sum(heap_blks_read)) AS ratio
                    FROM pg_stat_database
                    WHERE datname = current_database()
                """)
                cache_ratio = cursor.fetchone()[0]

                # Slow queries
                cursor.execute("""
                    SELECT count(*) FROM pg_stat_activity
                    WHERE (now() - query_start) > interval '5 minutes'
                """)
                slow_queries = cursor.fetchone()[0]
                DB_SLOW_QUERIES.inc(slow_queries)

                self.metrics = {
                    'timestamp': time.time(),
                    'active_connections': active_connections,
                    'database_size': db_size,
                    'cache_hit_ratio': cache_ratio,
                    'slow_queries': slow_queries
                }

        finally:
            conn.close()

        return self.metrics

    def get_query_stats(self, query):
        """Analyze specific query performance"""
        conn = psycopg2.connect(self.connection_string)

        try:
            with conn.cursor() as cursor:
                start_time = time.time()

                # Execute with EXPLAIN ANALYZE
                explain_query = f"EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) {query}"
                cursor.execute(explain_query)
                plan = cursor.fetchone()[0]

                execution_time = time.time() - start_time

                # Extract execution plan details
                execution_plan = json.loads(plan)
                total_cost = execution_plan[0]['Plan']['Total Cost']
                actual_time = execution_plan[0]['Execution Time']

                DB_QUERY_DURATION.observe(execution_time)

                return {
                    'query': query,
                    'execution_time': execution_time,
                    'total_cost': total_cost,
                    'actual_time': actual_time,
                    'plan': execution_plan
                }

        finally:
            conn.close()

# Usage
monitor = DatabaseMonitor('postgresql://user:pass@localhost/myapp')

# Collect metrics periodically
while True:
    metrics = monitor.collect_metrics()
    print(f"Database metrics: {json.dumps(metrics, indent=2)}")
    time.sleep(60)  # Collect every minute
```

## Database Backup and Recovery

### Backup Strategies

#### PostgreSQL Backup
```bash
# Logical backup with pg_dump
pg_dump -h localhost -U postgres -d myapp -f backup.sql

# Custom format backup (parallel)
pg_dump -h localhost -U postgres -d myapp -Fc -f backup.dump

# Directory format backup (parallel)
pg_dump -h localhost -U postgres -d myapp -Fd -j 4 -f backup_dir/

# Physical backup with pg_basebackup
pg_basebackup -h localhost -U postgres -D /backup/base -Ft -z -P 4 -v

# Point-in-time recovery backup
pg_basebackup -h localhost -U postgres -D /backup/base -Ft -z -x -v

# Archive WAL files
archive_command = 'cp %p /backup/wal_archive/%f'
```

#### MySQL Backup
```bash
# Logical backup with mysqldump
mysqldump -h localhost -u root -p --single-transaction --routines --triggers myapp > backup.sql

# Compressed backup
mysqldump -h localhost -u root -p --single-transaction --routines --triggers myapp | gzip > backup.sql.gz

# All databases backup
mysqldump -h localhost -u root -p --all-databases --single-transaction > all_databases.sql

# Physical backup with mysqlhotcopy
mysqlhotcopy --user=root --password=pass myapp /backup/myapp

# Binary backup (MySQL Enterprise Backup)
mysqlbackup --user=root --password=pass --backup-dir=/backup/mysql
```

### Recovery Procedures

#### PostgreSQL Point-in-Time Recovery
```bash
# Stop PostgreSQL
sudo systemctl stop postgresql

# Restore base backup
rm -rf /var/lib/postgresql/data/*
pg_basebackup -h localhost -U postgres -D /var/lib/postgresql/data -Ft -z -v

# Create recovery.conf
cat > /var/lib/postgresql/data/recovery.conf << EOF
restore_command = 'cp /backup/wal_archive/%f %p'
recovery_target_time = '2024-01-15 10:30:00'
standby_mode = off
EOF

# Start PostgreSQL
sudo systemctl start postgresql

# Monitor recovery progress
tail -f /var/log/postgresql/postgresql.log
```

#### MySQL Point-in-Time Recovery
```bash
# Stop MySQL
sudo systemctl stop mysql

# Restore backup
rm -rf /var/lib/mysql/*
mysql -u root -p < backup.sql

# Configure binary log recovery
cat >> /etc/mysql/my.cnf << EOF
[mysqld]
log-bin=mysql-bin
server-id=1
EOF

# Start MySQL with skip-slave-start
sudo systemctl start mysql --skip-slave-start

# Apply binary logs to point in time
mysqlbinlog --start-datetime="2024-01-15 10:00:00" \
           --stop-datetime="2024-01-15 10:30:00" \
           /var/log/mysql/mysql-bin.000001 | mysql -u root -p

# Restart MySQL normally
sudo systemctl restart mysql
```

## Real-World Database Architectures

### Facebook Database Architecture
```
Components:
- MySQL (user data, relationships)
- Cassandra (messaging, notifications)
- HBase (real-time analytics)
- HDFS (data warehouse)

Features:
- Sharding by user ID
- Read replicas for scaling
- Multi-region replication
- Custom caching layers
```

### Uber Database Architecture
```
Components:
- MySQL (core business data)
- PostgreSQL (geospatial data)
- Redis (caching, sessions)
- Schemaless (flexible data)

Features:
- Geographic sharding
- Real-time location tracking
- Trip data partitioning
- Multi-master replication
```

### Netflix Database Architecture
```
Components:
- Cassandra (user data, viewing history)
- MySQL (billing, account info)
- Elasticsearch (search)
- S3 (media storage)

Features:
- Multi-region deployment
- Eventual consistency
- Custom data modeling
- Automated failover
```

## Interview Tips

### Advanced Database Questions
1. "How would you design a database for 1 billion users?"
2. "What's the difference between vertical and horizontal partitioning?"
3. "How do you optimize slow queries?"
4. "Explain database isolation levels"
5. "How would you implement database replication?"

### Answer Framework
1. **Understand Requirements**: Scale, consistency, availability
2. **Choose Database Type**: SQL vs NoSQL based on needs
3. **Design Schema**: Normalization, indexing, constraints
4. **Plan Scaling**: Sharding, replication, caching
5. **Optimize Performance**: Query tuning, configuration, monitoring

### Key Points to Emphasize
- Trade-offs between consistency and performance
- Indexing strategies and their impact
- Partitioning and sharding approaches
- Backup and recovery procedures
- Monitoring and maintenance practices

## Practice Problems

1. **Design database for social media platform**
2. **Optimize slow database queries**
3. **Implement database replication**
4. **Design database for e-commerce**
5. **Create database monitoring system**

## Further Reading

- **Books**: "Database System Concepts" by Silberschatz, "SQL Performance Explained" by Markus Winand
- **Documentation**: PostgreSQL Documentation, MySQL Reference Manual
- **Concepts**: ACID Properties, CAP Theorem, Database Normalization

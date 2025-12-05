# System Design Interview Guide

## Overview

This guide provides a comprehensive framework for tackling system design interviews, covering the entire process from requirements gathering to final implementation details.

## Interview Framework

### Step 1: Requirements Clarification (5-10 minutes)

#### Functional Requirements
- **Core Features**: What should the system do?
- **User Actions**: What can users accomplish?
- **Data Operations**: Create, read, update, delete operations
- **Business Logic**: Rules and constraints

#### Non-Functional Requirements
- **Scale**: Number of users, requests per second, data volume
- **Performance**: Latency requirements, throughput targets
- **Availability**: Uptime requirements (99.9%, 99.99%, etc.)
- **Consistency**: Strong vs eventual consistency needs

#### Questions to Ask
```
Functional:
- What are the main features?
- Who are the users?
- What operations are supported?

Non-Functional:
- How many users do we expect?
- What's the expected QPS?
- What are the latency requirements?
- What's the availability target?

Constraints:
- What's the budget?
- What's the timeline?
- What's the team size?
- Any technology preferences?
```

### Step 2: System Estimation (5 minutes)

#### Back-of-the-Envelope Calculations
```
Example: Twitter-like Service
- Daily Active Users: 100 million
- Tweets per user per day: 5
- Total tweets per day: 500 million
- Average tweet size: 280 bytes
- Storage per day: 500M × 280B = 140GB
- Storage per year: 140GB × 365 = 51TB

QPS Calculation:
- Tweets per second: 500M / 86400 ≈ 5,787
- Peak QPS (3x): 5,787 × 3 ≈ 17,361
- Read QPS (10x writes): 173,610
```

#### Storage Estimation
```
Data Types:
- User profiles: 1KB per user
- Posts: 1KB per post
- Media: 500KB per post
- Comments: 200B per comment

Total Storage:
- Users: 100M × 1KB = 100GB
- Posts: 500M × 1KB = 500GB
- Media: 500M × 500KB = 250TB
- Comments: 1B × 200B = 200GB
```

### Step 3: High-Level Design (10-15 minutes)

#### System Components
```
Client
  ↓
Load Balancer
  ↓
API Gateway
  ↓
[Service 1] [Service 2] [Service 3]
  ↓           ↓           ↓
Database 1  Database 2  Database 3
```

#### Key Design Decisions
- **Architecture**: Monolithic vs Microservices
- **Database**: SQL vs NoSQL vs Hybrid
- **Caching**: In-memory, distributed, CDN
- **Communication**: Synchronous vs Asynchronous

### Step 4: Component Deep Dive (15-20 minutes)

#### Database Design
```sql
Users Table:
- id (BIGINT, PRIMARY KEY)
- username (VARCHAR, UNIQUE)
- email (VARCHAR, UNIQUE)
- password_hash (VARCHAR)
- created_at (TIMESTAMP)
- updated_at (TIMESTAMP)

Posts Table:
- id (BIGINT, PRIMARY KEY)
- user_id (BIGINT, FOREIGN KEY)
- content (TEXT)
- media_urls (JSON)
- created_at (TIMESTAMP)
- like_count (INT)
- comment_count (INT)
```

#### API Design
```
POST /api/v1/posts
GET /api/v1/posts/{post_id}
GET /api/v1/users/{user_id}/posts
PUT /api/v1/posts/{post_id}
DELETE /api/v1/posts/{post_id}
```

#### Caching Strategy
```
L1: Application Cache (Redis)
- User sessions
- Frequently accessed posts
- API responses

L2: CDN Cache
- Static assets
- Media files
- API responses (GET)

L3: Database Cache
- Query results
- Index pages
- Computed aggregates
```

### Step 5: Bottleneck Analysis (5-10 minutes)

#### Common Bottlenecks
1. **Database**: Single point, query performance
2. **Network**: Latency, bandwidth limitations
3. **Cache**: Miss ratio, eviction policies
4. **Load Balancer**: Connection limits, health checks

#### Solutions
```
Database Bottlenecks:
- Read replicas
- Sharding
- Indexing
- Connection pooling

Network Bottlenecks:
- Compression
- Batching
- CDN
- Edge computing

Cache Bottlenecks:
- Consistent hashing
- Multi-level caching
- Cache warming
- Optimized TTL
```

### Step 6: Trade-off Discussion (5 minutes)

#### Common Trade-offs
- **Consistency vs Availability**: CAP theorem
- **Performance vs Cost**: Resource allocation
- **Latency vs Throughput**: System optimization
- **Security vs Usability**: User experience

#### Example Discussion
```
Database Choice:
SQL (PostgreSQL):
Pros: ACID properties, complex queries, consistency
Cons: Vertical scaling limits, single point of failure

NoSQL (MongoDB):
Pros: Horizontal scaling, flexible schema, high availability
Cons: Eventual consistency, limited transactions

Decision: Use SQL for user data (consistency critical)
Use NoSQL for posts (scalability critical)
```

## Common System Design Patterns

### Scalability Patterns
1. **Horizontal Scaling**: Add more servers
2. **Vertical Scaling**: Increase server capacity
3. **Microservices**: Decompose into smaller services
4. **Caching**: Store frequently accessed data
5. **Load Balancing**: Distribute traffic

### Availability Patterns
1. **Redundancy**: Multiple instances
2. **Failover**: Automatic switching
3. **Circuit Breaker**: Prevent cascading failures
4. **Health Checks**: Monitor system health
5. **Graceful Degradation**: Reduced functionality

### Consistency Patterns
1. **Strong Consistency**: All nodes see same data
2. **Eventual Consistency**: Nodes converge over time
3. **Read Repair**: Fix inconsistencies on read
4. **Write Repair**: Fix inconsistencies on write
5. **Quorum**: Majority agreement

## Design Templates

### URL Shortener
```
Requirements:
- Generate short URL from long URL
- Redirect short URL to original
- Custom alias support
- Analytics tracking

Scale:
- 100M URLs created per day
- 1B redirects per day
- 99.9% availability
- <100ms redirect time

Design:
- Base62 encoding for short codes
- Distributed ID generation
- Redis for caching redirects
- PostgreSQL for URL metadata
- CDN for global performance
```

### Social Media Feed
```
Requirements:
- Post updates (text, media)
- Follow/unfollow users
- Generate personalized feed
- Real-time updates

Scale:
- 500M daily active users
- 500M posts per day
- 100K feed requests per second
- <200ms feed generation

Design:
- Fan-out on write for normal users
- Pull on read for celebrities
- Redis for timeline caching
- Elasticsearch for search
- Kafka for real-time updates
```

### E-commerce Platform
```
Requirements:
- Product catalog
- Shopping cart
- Order processing
- Payment processing
- Inventory management

Scale:
- 10M products
- 1M daily orders
- 99.99% availability
- Strong consistency for orders

Design:
- Microservices architecture
- PostgreSQL for orders (ACID)
- MongoDB for catalog (flexible)
- Redis for shopping cart
- Message queue for order processing
```

## Interview Tips

### Communication Skills
1. **Think Aloud**: Explain your thought process
2. **Ask Questions**: Clarify requirements early
3. **Draw Diagrams**: Visualize your design
4. **Explain Trade-offs**: Justify your decisions
5. **Handle Feedback**: Adapt to interviewer suggestions

### Common Mistakes to Avoid
1. **Jumping to Solutions**: Understand requirements first
2. **Ignoring Constraints**: Consider budget, timeline, team
3. **Over-engineering**: Keep it simple initially
4. **Silent Periods**: Keep the conversation going
5. **Defensive Attitude**: Be open to feedback

### Time Management
```
0-5 min: Requirements clarification
5-10 min: System estimation
10-25 min: High-level design
25-45 min: Component deep dive
45-55 min: Bottleneck analysis
55-60 min: Trade-offs and conclusion
```

## Practice Problems

### Beginner Level
1. **URL Shortener**: TinyURL, bit.ly
2. **Pastebin**: Text sharing service
3. **Web Crawler**: Google search basics
4. **Twitter Clone**: Simple social media
5. **Tiny Web Server**: Basic HTTP server

### Intermediate Level
1. **Facebook News Feed**: Social media feed
2. **YouTube/Netflix**: Video streaming
3. **Uber/Lyft**: Ride-sharing
4. **Airbnb**: Booking platform
5. **Dropbox**: File storage

### Advanced Level
1. **Google Search**: Web search engine
2. **WhatsApp**: Messaging platform
3. **Instagram**: Photo sharing
4. **Amazon**: E-commerce platform
5. **Netflix**: Video streaming with recommendations

## Technical Deep Dives

### Database Design
```sql
Normalization vs Denormalization:
Normalized:
- Reduces data redundancy
- Ensures data integrity
- Better for write-heavy workloads

Denormalized:
- Improves read performance
- Reduces join operations
- Better for read-heavy workloads

Example: Social Media
Normalized:
Users(id, name, email)
Posts(id, user_id, content)
Likes(id, user_id, post_id)

Denormalized:
Posts(id, user_id, content, user_name, like_count)
```

### Caching Strategies
```
Cache-Aside:
App → Check Cache → Miss → DB → Update Cache → Return

Read-Through:
App → Cache → DB (on miss) → Update Cache → Return

Write-Through:
App → Cache → DB (simultaneous) → Acknowledge

Write-Behind:
App → Cache (immediate ack) → DB (async)
```

### Load Balancing Algorithms
```
Round Robin:
Requests distributed evenly
Good for similar server capacity

Least Connections:
Route to least busy server
Good for variable request duration

IP Hash:
Consistent routing based on client IP
Good for session persistence

Weighted Round Robin:
Distribute based on server capacity
Good for heterogeneous servers
```

## Real-World Examples

### Netflix Architecture
```
Components:
- CDN (Open Connect)
- Microservices
- Chaos Engineering
- Real-time monitoring

Key Decisions:
- Cloud-native architecture
- Multi-region deployment
- Active-active configuration
- Automated failover
```

### Amazon Architecture
```
Components:
- Service-oriented architecture
- Microservices
- Database sharding
- Global infrastructure

Key Decisions:
- Two-pizza teams
- Decentralized architecture
- API-first design
- Continuous deployment
```

### Google Architecture
```
Components:
- Distributed file system
- MapReduce
- Bigtable
- Spanner

Key Decisions:
- Custom hardware
- Global scale
- Strong consistency
- Machine learning integration
```

## Further Preparation

### Study Resources
- **Books**: "Designing Data-Intensive Applications" by Martin Kleppmann
- **Courses**: "System Design Interview" on educative.io, "Grokking the System Design Interview"
- **Practice**: LeetCode system design problems, mock interviews
- **Blogs**: High Scalability, Netflix TechBlog, AWS Architecture Blog

### Mock Interview Checklist
```
Before Interview:
- Review common patterns
- Practice whiteboarding
- Prepare questions to ask
- Test your setup (if remote)

During Interview:
- Start with requirements
- Draw clear diagrams
- Explain your decisions
- Handle feedback gracefully

After Interview:
- Note what went well
- Identify areas for improvement
- Follow up if appropriate
```

## Final Tips

1. **Start Simple**: Begin with basic design, then add complexity
2. **Communicate Clearly**: Explain your thought process
3. **Be Flexible**: Adapt to new requirements
4. **Show Your Work**: Justify your decisions
5. **Stay Calm**: It's a conversation, not an exam

Remember: The goal is to demonstrate your thinking process, not to find the perfect solution. Good luck!
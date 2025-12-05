# Consistency Models

## Definition

Consistency in distributed systems refers to the guarantee that every read receives the most recent write or an error. Different consistency models provide different guarantees about when and how updates become visible to different parts of the system.

## Consistency Spectrum

```
Strong Consistency ←→ Weak Consistency
     ↑                           ↓
Linearizability               Eventual
Sequential Consistency         Causal
```

## Strong Consistency Models

### 1. Linearizability
- **Definition**: All operations appear to occur instantaneously at some point between invocation and response
- **Guarantee**: Most recent write is always visible
- **Use Case**: Financial transactions, database locks
- **Trade-off**: High latency, limited availability

```
Time →
Client A: Write(X=1) ──────✓
Client B:           Read(X) → 1
Client C:           Read(X) → 1
```

### 2. Sequential Consistency
- **Definition**: Operations are seen in a total order that respects program order
- **Guarantee**: All processes see same order of operations
- **Use Case**: Collaborative editing, version control
- **Trade-off**: Better performance than linearizability

### 3. Causal Consistency
- **Definition**: Causally related operations are seen in order by all processes
- **Guarantee**: Preserves cause-effect relationships
- **Use Case**: Social media feeds, messaging systems
- **Trade-off**: Good balance between consistency and performance

```
User A posts → User B comments → User C likes
All users see: Post → Comment → Like
```

## Weak Consistency Models

### 1. Eventual Consistency
- **Definition**: If no new updates are made, all replicas converge to same value
- **Guarantee**: Eventually consistent given no updates
- **Use Case**: DNS, social media, caching
- **Trade-off**: High availability, low latency

```
Time →
Replica 1: X=1 ── X=2 ── X=3
Replica 2: X=1 ──────── X=3
Replica 3: X=1 ── X=2 ──────
Eventually: All replicas have X=3
```

### 2. Read Your Writes
- **Definition**: Client sees its own writes
- **Guarantee**: Session consistency for individual client
- **Use Case**: User profiles, shopping carts
- **Trade-off**: Client-centric consistency

### 3. Monotonic Reads
- **Definition**: Client never sees older data after seeing newer data
- **Guarantee**: No time travel for individual client
- **Use Case**: News feeds, timeline views
- **Trade-off**: Per-client ordering guarantee

## Consistency in Databases

### ACID Properties
- **Atomicity**: All or nothing transaction execution
- **Consistency**: Database remains in valid state
- **Isolation**: Concurrent transactions don't interfere
- **Durability**: Committed transactions persist

### BASE Properties (NoSQL)
- **Basically Available**: System remains operational despite failures
- **Soft State**: State may change over time (eventual consistency)
- **Eventually Consistent**: Replicas converge given no updates

### Isolation Levels
| Level | Dirty Reads | Non-repeatable Reads | Phantom Reads |
|-------|-------------|---------------------|---------------|
| Read Uncommitted | ✓ | ✓ | ✓ |
| Read Committed | ✗ | ✓ | ✓ |
| Repeatable Read | ✗ | ✗ | ✓ |
| Serializable | ✗ | ✗ | ✗ |

### Consistency in Microservices

#### Saga Pattern
```
Orchestration-based Saga:
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ Order       │───▶│ Payment     │───▶│ Inventory   │
│ Service     │    │ Service     │    │ Service     │
│             │◀───│             │◀───│             │
│ Compensate  │    │ Compensate  │    │ Compensate  │
└─────────────┘    └─────────────┘    └─────────────┘
```

**Types:**
- **Choreography**: Services communicate directly via events
- **Orchestration**: Central coordinator manages the saga

**Benefits:**
- **Distributed Transactions**: Without 2PC overhead
- **Fault Tolerance**: Compensating transactions for rollback
- **Scalability**: Avoids blocking locks

#### Event Sourcing
```
Traditional: State → Database
Event Sourcing: Events → State → Database

Event Stream:
├── UserCreated(id=123, name="John")
├── UserUpdated(id=123, email="john@example.com")
├── UserDeleted(id=123)
└── State Rebuilt: User(id=123, deleted=true)
```

**Benefits:**
- **Audit Trail**: Complete history of changes
- **Temporal Queries**: State at any point in time
- **Event Replay**: Rebuild state from events
- **CQRS Integration**: Separate read/write models

#### CQRS (Command Query Responsibility Segregation)
```
Command Side (Write):
├── Commands: CreateOrder, UpdateOrder, CancelOrder
├── Validation: Business rules, constraints
├── Events: OrderCreated, OrderUpdated, OrderCancelled
└── Storage: Event store, write-optimized

Query Side (Read):
├── Queries: GetOrder, ListOrders, SearchOrders
├── Models: Denormalized views, optimized for reads
├── Projections: Event handlers build read models
└── Storage: Read-optimized databases, caches
```

**Benefits:**
- **Performance**: Optimized read/write paths
- **Scalability**: Scale reads/writes independently
- **Flexibility**: Multiple read models for different needs
- **Eventual Consistency**: Read models lag behind writes

## Consistency Protocols

### 1. Two-Phase Commit (2PC)
```
Phase 1: Prepare
Coordinator → Participants: "Can you commit?"
Participants → Coordinator: "Yes/No"

Phase 2: Commit/Abort
Coordinator → Participants: "Commit/Abort"
Participants → Coordinator: "ACK"
```

- **Pros**: Strong consistency, distributed transactions
- **Cons**: Blocking, single point of failure, high latency

### 2. Paxos Algorithm
- **Purpose**: Achieving consensus in distributed systems
- **Roles**: Proposer, Acceptor, Learner
- **Guarantee**: Safety and liveness under certain conditions

### 3. Raft Algorithm
- **Purpose**: More understandable consensus algorithm
- **Components**: Leader election, log replication, safety
- **Guarantee**: Strong consistency, leader-based approach

```
Term 1: Leader A
Term 2: Leader B (after A fails)
Term 3: Leader C (after B fails)
```

## Consistency Patterns

### 1. Quorum-based Replication
```
N = Total replicas
W = Write quorum
R = Read quorum

Strong consistency if: W + R > N
```

- **Example**: N=3, W=2, R=2 ensures strong consistency
- **Trade-off**: Latency vs consistency guarantee

### 2. Leader-Follower Replication
```
Client → Leader → Followers
         ↓
    Response to Client
```

- **Write Path**: Leader processes, replicates to followers
- **Read Path**: Can read from leader or followers
- **Consistency**: Strong if reading from leader

### 3. Multi-Leader Replication
```
Client A → Leader A ↔ Leader B ← Client B
         ↓           ↓
    Followers A   Followers B
```

- **Challenge**: Conflict resolution
- **Resolution**: Last-write-wins, vector clocks, CRDTs

## Conflict Resolution Strategies

### 1. Last-Write-Wins (LWW)
- **Strategy**: Use timestamp to determine winner
- **Pros**: Simple, deterministic
- **Cons**: Can lose updates

### 2. Vector Clocks
- **Strategy**: Track causality between operations
- **Pros**: Preserve causal relationships
- **Cons**: Complex implementation

### 3. Conflict-free Replicated Data Types (CRDTs)
- **Strategy**: Data structures that automatically merge
- **Types**: G-Counter, PN-Counter, OR-Set, LWW-Element-Set

### 4. Version Vectors and Vector Clocks
```
Vector Clock Example:
Node A: [1, 0, 0] - Write operation 1
Node B: [1, 1, 0] - Write operation 2
Node C: [1, 0, 1] - Concurrent write operation 3

Comparison:
[1,1,0] > [1,0,0] - Node B has seen A's write
[1,0,1] > [1,0,0] - Node C has seen A's write
[1,1,0] || [1,0,1] - Concurrent operations
```

**Benefits:**
- **Causality Detection**: Identify concurrent vs causal operations
- **Conflict Resolution**: Determine which operations to merge
- **Distributed Coordination**: Without central authority

### Consistency in Cloud Architectures

#### Strong Consistency Services
```
AWS:
├── DynamoDB (Strongly Consistent Reads)
├── Aurora (ACID transactions)
└── RDS (Traditional RDBMS)

Azure:
├── Cosmos DB (Strong consistency option)
├── SQL Database (ACID)
└── Database for PostgreSQL

GCP:
├── Spanner (External consistency)
├── Cloud SQL (ACID)
└── Firestore (Strong consistency)
```

#### Eventual Consistency Services
```
AWS:
├── DynamoDB (Default eventual consistency)
├── S3 (Eventual consistency for metadata)
└── ElastiCache (Cache consistency)

Azure:
├── Cosmos DB (Session/Eventual consistency)
├── Table Storage (Eventual consistency)
└── Redis Cache

GCP:
├── Bigtable (Eventual consistency)
├── Datastore (Eventual consistency)
└── Memorystore (Cache consistency)
```

### Serverless Consistency

#### Function State Management
```
Stateless Functions:
├── No local state persistence
├── External state in databases/caches
├── Event-driven consistency
└── Idempotent operations

Stateful Functions:
├── Durable Functions (Azure)
├── Step Functions (AWS)
├── Workflows (GCP)
└── Strong consistency guarantees
```

**Challenges:**
- **Cold Starts**: Initialization consistency
- **Execution Time Limits**: Transaction timeouts
- **Coordination**: Distributed function calls
- **Observability**: Tracing across function boundaries

## Real-World Examples

### Google Spanner
- **Consistency**: External consistency (strong)
- **Technology**: TrueTime API, Paxos replication
- **Use Case**: Global financial transactions

### Amazon DynamoDB
- **Consistency**: Eventually consistent (default), strongly consistent (optional)
- **Technology**: Quorum-based replication, vector clocks
- **Use Case**: Shopping carts, user sessions

### Cassandra
- **Consistency**: Tunable consistency per operation
- **Technology**: Quorum-based, leaderless replication
- **Use Case**: Time series, logging, analytics

### MongoDB
- **Consistency**: Strong consistency for primary reads
- **Technology**: Primary-secondary replication
- **Use Case**: Content management, user profiles

## Interview Tips

### Common Questions
1. "Explain CAP theorem with examples"
2. "What's the difference between strong and eventual consistency?"
3. "How would you design a globally distributed database?"
4. "When would you use eventual consistency?"
5. "What's the saga pattern and when would you use it?"
6. "Explain CQRS and event sourcing"
7. "How do CRDTs work?"

### Answer Framework
1. **Understand Requirements**: Consistency needs vs availability vs performance
2. **Choose Model**: ACID vs BASE, strong vs eventual consistency
3. **Design Architecture**: Replication strategy, conflict resolution, partitioning
4. **Implement Patterns**: Saga, CQRS, event sourcing for complex systems
5. **Handle Conflicts**: Vector clocks, CRDTs, last-write-wins strategies
6. **Consider Trade-offs**: Latency, complexity, cost, scalability
7. **Monitor and Tune**: Adjust consistency levels based on usage patterns

### Key Points to Emphasize
- No one-size-fits-all solution - depends on business requirements
- Trade-offs between consistency, availability, and performance (CAP theorem)
- Business requirements drive consistency choices (financial vs social media)
- Consider both read and write patterns when choosing models
- Plan for conflict resolution in distributed systems
- Modern patterns like CQRS and event sourcing for complex domains
- Cloud services offer different consistency guarantees

## Practice Problems

1. **Design a globally consistent counter with CRDTs**
2. **Implement conflict resolution for collaborative editing with operational transforms**
3. **Design a shopping cart with eventual consistency and conflict resolution**
4. **Create a messaging system with causal consistency using vector clocks**
5. **Design a financial system with strong consistency using sagas**
6. **Implement CQRS for an e-commerce platform**
7. **Design event sourcing for a banking system**
8. **Create a distributed leaderboard with eventual consistency**

## Further Reading

- **Books**: "Designing Data-Intensive Applications" by Martin Kleppmann
- **Papers**: "Dynamo: Amazon's Highly Available Key-value Store", "Spanner: Google's Globally-Distributed Database"
- **Concepts**: CRDTs, Vector Clocks, Consensus Algorithms
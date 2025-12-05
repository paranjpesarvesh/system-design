# Consistency Patterns

## Definition

Consistency patterns are architectural approaches that ensure data remains consistent across distributed systems, balancing between availability, performance, and correctness requirements.

## Consistency Models

### Strong Consistency
```
Client A → Database → Response (Latest)
    ↓                ↓
Client B → Database → Response (Latest)
    ↓                ↓
Client C → Database → Response (Latest)

All clients see the same data at the same time
```

**Characteristics:**
- **Linearizability**: Operations appear to execute atomically
- **Read-Your-Writes**: Clients see their own writes
- **Monotonic Reads**: Clients never see data go backward in time
- **High Latency**: Requires coordination between replicas

### Eventual Consistency
```
Client A → Database → Response (May be stale)
    ↓                ↓
Client B → Database → Response (May be different)
    ↓                ↓
Client C → Database → Response (May be different)

All replicas eventually converge to the same state
```

**Characteristics:**
- **High Availability**: System remains operational during partitions
- **Low Latency**: No coordination required for reads
- **Stale Reads**: Clients may see outdated data
- **Convergence**: All replicas eventually agree

### Weak Consistency
```
Client A → Replica 1 → Response (Local state)
    ↓
Client B → Replica 2 → Response (Local state)
    ↓
Client C → Replica 3 → Response (Local state)

Different replicas may return different values
```

**Characteristics:**
- **Highest Availability**: System tolerates network partitions
- **Lowest Latency**: No coordination required
- **Inconsistent Reads**: Clients may see conflicting data
- **No Convergence Guarantee**: May remain inconsistent

## Consistency Protocols

### Two-Phase Commit (2PC)
```
Coordinator
    ↓
┌───────────────────────────────────────────────┐
│              Phase 1: Prepare                 │
│                                               │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  │
│  │Resource A │  │Resource B │  │Resource C │  │
│  └───────────┘  └───────────┘  └───────────┘  │
│         ↓           ↓           ↓             │
│    Prepare      Prepare      Prepare          │
│    Response     Response     Response         │
│  └─────────┘  └─────────┘  └─────────┘        │
│                                               │
│         All Prepared? ──────────────┘         │
│                ↓                              │
│              Phase 2: Commit                  │
│                                               │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  │
│  │Resource A │  │Resource B │  │Resource C │  │
│  └───────────┘  └───────────┘  └───────────┘  │
│         ↓           ↓           ↓             │
│    Commit       Commit       Commit           │
│  └─────────┘  └─────────┘  └─────────┘        │
└───────────────────────────────────────────────┘
```

**Implementation:**
```python
import time
import threading
from enum import Enum
from typing import Dict, Any, Callable

class TransactionState(Enum):
    PREPARING = "preparing"
    PREPARED = "prepared"
    COMMITTING = "committing"
    COMMITTED = "committed"
    ABORTED = "aborted"

class TwoPhaseCommit:
    def __init__(self, participants: List[str]):
        self.participants = participants
        self.transaction_id = f"tx_{int(time.time())}"
        self.state = TransactionState.PREPARING
        self.prepared_responses = {}
        self.lock = threading.Lock()

    def prepare_phase(self, transaction_data: Dict[str, Any]) -> bool:
        """Execute prepare phase on all participants"""
        self.state = TransactionState.PREPARING

        with self.lock:
            for participant in self.participants:
                try:
                    response = self._send_prepare_request(participant, transaction_data)
                    self.prepared_responses[participant] = response

                    if not response['success']:
                        print(f"Prepare failed for {participant}: {response['error']}")
                        self.abort_phase()
                        return False

                except Exception as e:
                    print(f"Prepare error for {participant}: {e}")
                    self.abort_phase()
                    return False

        # Check if all participants prepared successfully
        all_prepared = all(
            response['success']
            for response in self.prepared_responses.values()
        )

        if all_prepared:
            self.state = TransactionState.PREPARED
            return True
        else:
            self.state = TransactionState.ABORTED
            return False

    def commit_phase(self) -> bool:
        """Execute commit phase on all participants"""
        if self.state != TransactionState.PREPARED:
            return False

        self.state = TransactionState.COMMITTING

        with self.lock:
            for participant in self.participants:
                try:
                    response = self._send_commit_request(participant)

                    if not response['success']:
                        print(f"Commit failed for {participant}: {response['error']}")
                        # In real implementation, would need compensation logic
                        return False

                except Exception as e:
                    print(f"Commit error for {participant}: {e}")
                    return False

        self.state = TransactionState.COMMITTED
        return True

    def abort_phase(self):
        """Execute abort phase on all participants"""
        self.state = TransactionState.ABORTED

        with self.lock:
            for participant in self.participants:
                try:
                    response = self._send_abort_request(participant)
                    if not response['success']:
                        print(f"Abort failed for {participant}: {response['error']}")
                except Exception as e:
                    print(f"Abort error for {participant}: {e}")

    def _send_prepare_request(self, participant: str, data: Dict[str, Any]) -> Dict[str, Any]:
        """Send prepare request to participant"""
        # Simulate network request
        time.sleep(0.1)  # Network latency

        # Simulate participant logic
        if participant == "resource_a":
            return {'success': True, 'message': 'Prepared'}
        elif participant == "resource_b":
            return {'success': True, 'message': 'Prepared'}
        elif participant == "resource_c":
            # Simulate failure for demonstration
            if data.get('force_failure', False):
                return {'success': True, 'message': 'Prepared'}
            else:
                return {'success': False, 'error': 'Resource unavailable'}

        return {'success': False, 'error': 'Unknown participant'}

    def _send_commit_request(self, participant: str) -> Dict[str, Any]:
        """Send commit request to participant"""
        time.sleep(0.1)  # Network latency

        if participant == "resource_a":
            return {'success': True, 'message': 'Committed'}
        elif participant == "resource_b":
            return {'success': True, 'message': 'Committed'}
        elif participant == "resource_c":
            return {'success': True, 'message': 'Committed'}

        return {'success': False, 'error': 'Commit failed'}

    def _send_abort_request(self, participant: str) -> Dict[str, Any]:
        """Send abort request to participant"""
        time.sleep(0.1)  # Network latency

        return {'success': True, 'message': 'Aborted'}

# Usage
coordinator = TwoPhaseCommit(['resource_a', 'resource_b', 'resource_c'])

# Transaction data
transaction_data = {
    'operation': 'transfer',
    'amount': 100,
    'from_account': 'account_a',
    'to_account': 'account_b'
}

# Execute 2PC
if coordinator.prepare_phase(transaction_data):
    print("All participants prepared, committing...")
    if coordinator.commit_phase():
        print("Transaction committed successfully")
    else:
        print("Transaction commit failed")
else:
    print("Transaction aborted during prepare phase")
```

### Three-Phase Commit (3PC)
```
Coordinator
    ↓
┌─────────────────────────────────────────┐
│              Phase 1: CanCommit         │
│                                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  │
│  │Resource A│  │Resource B│  │Resource C│ │
│  └─────────┘  └─────────┘  └─────────┘ │
│         ↓           ↓           ↓         │
│    CanCommit   CanCommit   CanCommit   │
│    Response    Response    Response    │
│  └─────────┘  └─────────┘  └─────────┘ │
│                                     │
│         All CanCommit? ──────────────┘ │
│                ↓                   │
│              Phase 2: PreCommit       │
│                                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │Resource A│  │Resource B│  │Resource C│ │
│  └─────────┘  └─────────┘  └─────────┘ │
│         ↓           ↓           ↓         │
│    PreCommit    PreCommit    PreCommit    │
│    Response    Response    Response    │
│  └─────────┘  └─────────┘  └─────────┘ │
│                                     │
│         All PreCommitted? ──────────────┘ │
│                ↓                   │
│              Phase 3: Commit          │
│                                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │Resource A│  │Resource B│  │Resource C│ │
│  └─────────┘  └─────────┘  └─────────┘ │
│         ↓           ↓           ↓         │
│    Commit       Commit       Commit       │
│  └─────────┘  └─────────┘  └─────────┘ │
└─────────────────────────────────────────┘
```

**Benefits over 2PC:**
- **Pre-commit phase**: Reduces blocking during commit
- **Can-commit check**: Early detection of conflicts
- **Better failure handling**: More granular control

## Consistency Patterns

### Quorum-Based Replication
```
Write Request
    ↓
Primary Node
    ↓
┌─────────────────────────────────────────┐
│                                     │
│  Quorum: Write to majority (N/2 + 1) │
│                                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │Replica 1│  │Replica 2│  │Replica 3│ │
│  └─────────┘  └─────────┘  └─────────┘ │
│         ↓           ↓           ↓         │
│    Acknowledge   Acknowledge   Acknowledge   │
│  └─────────┘  └─────────┘  └─────────┘ │
│                                     │
│         Majority Acknowledged? ──────────────┘ │
│                ↓                   │
│            Commit Success           │
└─────────────────────────────────────────┘
```

**Implementation:**
```python
import time
import threading
from typing import List, Dict, Any

class QuorumReplication:
    def __init__(self, replicas: List[str], quorum_size: int):
        self.replicas = replicas
        self.quorum_size = quorum_size
        self.write_logs = {}  # transaction_id -> list of acknowledgments
        self.lock = threading.Lock()

    def write_with_quorum(self, data: Dict[str, Any]) -> bool:
        """Write data to quorum of replicas"""
        transaction_id = f"tx_{int(time.time())"
        self.write_logs[transaction_id] = []

        with self.lock:
            # Send write to all replicas
            for replica in self.replicas:
                try:
                    ack = self._send_write_request(replica, data)
                    if ack['success']:
                        self.write_logs[transaction_id].append(replica)

                    print(f"Write to {replica}: {'ACK' if ack['success'] else 'NACK'}")

                except Exception as e:
                    print(f"Write error to {replica}: {e}")

            # Check if quorum achieved
            if len(self.write_logs[transaction_id]) >= self.quorum_size:
                print(f"Quorum achieved ({len(self.write_logs[transaction_id])}/{len(self.replicas)})")
                return True
            else:
                print(f"Quorum not achieved ({len(self.write_logs[transaction_id])}/{len(self.replicas)})")
                return False

    def _send_write_request(self, replica: str, data: Dict[str, Any]) -> Dict[str, Any]:
        """Send write request to replica"""
        # Simulate network request
        time.sleep(0.05)  # Network latency

        # Simulate replica behavior
        if replica == "replica_1":
            return {'success': True, 'message': 'Written'}
        elif replica == "replica_2":
            return {'success': True, 'message': 'Written'}
        elif replica == "replica_3":
            return {'success': True, 'message': 'Written'}
        elif replica == "replica_4":
            return {'success': True, 'message': 'Written'}
        elif replica == "replica_5":
            # Simulate occasional failure
            import random
            if random.random() > 0.8:  # 20% failure rate
                return {'success': False, 'error': 'Write failed'}
            else:
                return {'success': True, 'message': 'Written'}

        return {'success': False, 'error': 'Unknown replica'}

# Usage
replicas = ['replica_1', 'replica_2', 'replica_3', 'replica_4', 'replica_5']
quorum_size = 3  # Majority of 5

quorum_system = QuorumReplication(replicas, quorum_size)

# Test quorum writes
for i in range(10):
    data = {'key': f'value_{i}', 'timestamp': time.time()}
    success = quorum_system.write_with_quorum(data)
    print(f"Write {i+1}: {'Success' if success else 'Failed'}")

    time.sleep(0.5)
```

### Leader Election (Raft)
```
Cluster Nodes
    ↓
┌─────────────────────────────────────────┐
│                                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │  Node A  │  │  Node B  │  │  Node C  │ │
│  └─────────┘  └─────────┘  └─────────┘ │
│                                     │
│              Leader Election Process      │
│                                     │
│  ┌─────────────────────────────────┐ │
│  │         Leader: Node B         │ │
│  │         Followers: A, C          │ │
│  │         Term: 5                  │ │
│  └─────────────────────────────────┘ │
│                                     │
│         Log Replication              │
│                                     │
│  Leader → Followers: Log Entries   │
└─────────────────────────────────────────┘
```

**Raft Implementation:**
```python
import time
import threading
import random
from enum import Enum
from typing import List, Dict, Any

class NodeState(Enum):
    FOLLOWER = "follower"
    CANDIDATE = "candidate"
    LEADER = "leader"

class LogEntry:
    def __init__(self, term: int, index: int, command: str):
        self.term = term
        self.index = index
        self.command = command
        self.timestamp = time.time()

class RaftNode:
    def __init__(self, node_id: str, peers: List[str]):
        self.node_id = node_id
        self.peers = peers
        self.state = NodeState.FOLLOWER
        self.current_term = 0
        self.voted_for = None
        self.log = []
        self.commit_index = 0
        self.leader_id = None
        self.election_timeout = 5.0  # seconds
        self.heartbeat_interval = 1.0  # seconds
        self.lock = threading.Lock()

    def start_election(self):
        """Start leader election"""
        self.current_term += 1
        self.state = NodeState.CANDIDATE
        self.voted_for = self.node_id
        self.leader_id = None

        print(f"{self.node_id}: Starting election for term {self.current_term}")

        # Request votes from peers
        votes = 1  # Vote for self
        for peer in self.peers:
            if self._request_vote(peer):
                votes += 1

        # Check if won election
        if votes > len(self.peers) // 2:
            self.become_leader()
        else:
            # Wait for election timeout or new leader
            time.sleep(self.election_timeout)
            if self.state == NodeState.CANDIDATE:
                self.state = NodeState.FOLLOWER

    def _request_vote(self, peer: str) -> bool:
        """Request vote from peer"""
        # Simulate vote request
        # In real implementation, this would be an RPC call
        return random.choice([True, True, False])  # 66% chance of getting vote

    def become_leader(self):
        """Become leader"""
        self.state = NodeState.LEADER
        self.leader_id = self.node_id
        print(f"{self.node_id}: Became leader for term {self.current_term}")

        # Start sending heartbeats
        threading.Thread(target=self._send_heartbeats, daemon=True).start()

    def _send_heartbeats(self):
        """Send heartbeats to followers"""
        while self.state == NodeState.LEADER:
            for peer in self.peers:
                heartbeat = {
                    'term': self.current_term,
                    'leader_id': self.node_id,
                    'commit_index': self.commit_index,
                    'timestamp': time.time()
                }

                # Send heartbeat to peer
                print(f"{self.node_id}: Sending heartbeat to {peer}")
                time.sleep(self.heartbeat_interval)

    def append_log(self, command: str):
        """Append entry to log"""
        if self.state == NodeState.LEADER:
            entry = LogEntry(self.current_term, len(self.log), command)
            self.log.append(entry)

            # Replicate to followers
            for peer in self.peers:
                print(f"{self.node_id}: Replicating log entry {entry.index} to {peer}")
                # In real implementation, send log entry to peer

    def commit_log_entry(self, index: int):
        """Commit log entry"""
        if index < len(self.log):
            self.commit_index = index
            entry = self.log[index]
            print(f"{self.node_id}: Committed log entry {index}: {entry.command}")

# Usage
nodes = ['node_a', 'node_b', 'node_c', 'node_d', 'node_e']

# Create Raft nodes
raft_nodes = []
for i, node_id in enumerate(nodes):
    peers = [n for n in nodes if n != node_id]
    node = RaftNode(node_id, peers)
    raft_nodes.append(node)

# Simulate election
print("Starting Raft cluster...")
raft_nodes[0].start_election()  # Node A starts election

# Let election complete
time.sleep(6)

# Check states
for node in raft_nodes:
    print(f"{node.node_id}: State = {node.state.value}, Term = {node.current_term}")
```

## Conflict Resolution

### Last-Write-Wins
```
Concurrent Writes:
Client A → Replica 1: Write X = 5
Client B → Replica 2: Write X = 10
    ↓                ↓
Replica 1: X = 5 (stored)
Replica 2: X = 10 (stored)
    ↓                ↓
Client C → Replica 1: Read X → Returns 5
Client D → Replica 2: Read X → Returns 10

Result: X = 10 (last write wins)
```

### Vector Clocks
```
Client A: [1, 1, 1] → Server A
Client B: [2, 2, 2] → Server B
    ↓                ↓
Server A: [1, 1, 1] → Client B
Server B: [2, 2, 2] → Client A

Comparison:
Client A: [1, 1, 1] < [2, 2, 2] (Client B's data is newer)
Client B: [2, 2, 2] > [1, 1, 1] (Client A's data is older)

Result: Both clients see each other's updates
```

**Vector Clock Implementation:**
```python
import time
from typing import Dict, List

class VectorClock:
    def __init__(self):
        self.clock = {}  # node_id -> logical_time

    def update(self, node_id: str):
        """Update logical time for node"""
        self.clock[node_id] = self.clock.get(node_id, 0) + 1

    def get_time(self, node_id: str) -> int:
        """Get logical time for node"""
        return self.clock.get(node_id, 0)

    def compare(self, other_clock: Dict[str, int]) -> bool:
        """Compare with another vector clock"""
        for node_id in set(self.clock.keys()) | set(other_clock.keys()):
            my_time = self.get_time(node_id)
            their_time = other_clock.get(node_id, 0)

            if my_time < their_time:
                return False  # Other clock is ahead
        return True  # My clock is ahead or equal

    def merge(self, other_clock: Dict[str, int]):
        """Merge with another vector clock"""
        for node_id, time in other_clock.items():
            current_time = self.get_time(node_id)
            new_time = max(current_time, time)
            self.clock[node_id] = new_time

class Event:
    def __init__(self, event_id: str, node_id: str, data: Any):
        self.event_id = event_id
        self.node_id = node_id
        self.data = data
        self.vector_clock = VectorClock()
        self.timestamp = time.time()

class EventStore:
    def __init__(self):
        self.events = []
        self.vector_clock = VectorClock()

    def add_event(self, event: Event):
        """Add event to store"""
        # Update vector clock for this event
        self.vector_clock.update(event.node_id)
        event.vector_clock = self.vector_clock.copy()

        self.events.append(event)
        print(f"Event {event.event_id} added by {event.node_id}")

    def get_events_since(self, client_clock: Dict[str, int]) -> List[Event]:
        """Get events that client hasn't seen"""
        new_events = []

        for event in self.events:
            if event.vector_clock.compare(client_clock):
                new_events.append(event)

        return new_events

# Usage
event_store = EventStore()

# Simulate events from different nodes
event1 = Event("evt_1", "node_a", {"key": "value1"})
event2 = Event("evt_2", "node_b", {"key": "value2"})
event3 = Event("evt_3", "node_a", {"key": "value3"})

event_store.add_event(event1)
event_store.add_event(event2)
event_store.add_event(event3)

# Client clocks
client_a_clock = {"node_a": 2, "node_b": 1}
client_b_clock = {"node_a": 1, "node_b": 2}

# Get events for each client
events_for_a = event_store.get_events_since(client_a_clock)
events_for_b = event_store.get_events_since(client_b_clock)

print(f"Events for Client A: {len(events_for_a)}")
print(f"Events for Client B: {len(events_for_b)}")
```

## Real-World Examples

### Google Spanner
```
Features:
- External consistency
- Global transactions
- TrueTime API for global clock
- Automatic sharding
- Paxos-based replication

Consistency Model:
- Strong consistency with bounded staleness
- Global transaction support
- Cross-region transactions
```

### Amazon DynamoDB
```
Features:
- Eventually consistent reads
- Strong consistent reads (optional)
- Conditional writes
- Automatic conflict resolution

Consistency Models:
- Eventually consistent reads (default)
- Strong consistent reads (DynamoDB Transactions)
- Read-after-write consistency
```

### Cassandra
```
Features:
- Eventual consistency
- Tunable consistency per operation
- Last-write-wins conflict resolution
- Anti-entropy repairs

Consistency Models:
- ONE (strongest consistency)
- QUORUM (majority consistency)
- LOCAL_QUORUM (datacenter consistency)
- EACH_QUORUM (rack consistency)
- LOCAL_ONE (eventual consistency)
```

## Interview Tips

### Common Questions
1. "What's the difference between strong and eventual consistency?"
2. "How does two-phase commit work?"
3. "What is the CAP theorem?"
4. "How would you implement leader election?"
5. "How would you resolve conflicts in distributed system?"

### Answer Framework
1. **Understand Requirements**: Consistency vs availability needs
2. **Choose Model**: Strong vs eventual based on use case
3. **Design Protocol**: 2PC, 3PC, Raft, or custom
4. **Handle Conflicts**: LWW, vector clocks, CRDTs
5. **Consider Trade-offs**: Latency, complexity, cost

### Key Points to Emphasize
- CAP theorem implications
- Consistency vs availability trade-offs
- Network partition handling
- Conflict resolution strategies
- Performance considerations

## Practice Problems

1. **Design distributed transaction system**
2. **Implement leader election algorithm**
3. **Create conflict resolution mechanism**
4. **Design eventually consistent data store**
5. **Build strongly consistent key-value store**

## Further Reading

- **Books**: "Designing Data-Intensive Applications" by Martin Kleppmann
- **Papers**: "Paxos Made Simple" by Leslie Lamport, "In Search of an Understandable Consensus Algorithm" by Diego Ongaro
- **Concepts**: CAP Theorem, Vector Clocks, CRDTs

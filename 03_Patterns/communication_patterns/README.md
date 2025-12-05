# Communication Patterns

## Definition

Communication patterns define how services and components in distributed systems exchange information, coordinate actions, and maintain consistency across networks.

## Synchronous vs Asynchronous Communication

### Synchronous Communication
```
Client → Request → Server
    ↓                ↓
  Wait            Process
    ↓                ↓
Response ← Server ← Client

Characteristics:
- Immediate response required
- Blocking operation
- Tight coupling
- Strong consistency
```

### Asynchronous Communication
```
Client → Message Queue → Worker
    ↓                      ↓
Continue            Process
    ↓                      ↓
Response ← Worker ← Client (later)

Characteristics:
- No immediate response
- Non-blocking operation
- Loose coupling
- Eventual consistency
```

## Request-Response Patterns

### REST API Pattern
```python
import requests
import time
from typing import Dict, Any

class RESTClient:
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.timeout = 30
    
    def get(self, endpoint: str, params: Dict = None) -> Dict[str, Any]:
        """GET request"""
        url = f"{self.base_url}{endpoint}"
        if params:
            url += f"?{requests.compat.urlencode(params)}"
        
        try:
            response = requests.get(url, timeout=self.timeout)
            return {
                'success': response.status_code == 200,
                'data': response.json() if response.headers.get('content-type', '').startswith('application/json') else response.text,
                'status_code': response.status_code,
                'response_time': response.elapsed.total_seconds()
            }
        except requests.exceptions.RequestException as e:
            return {
                'success': False,
                'error': str(e),
                'response_time': 0
            }
    
    def post(self, endpoint: str, data: Dict[str, Any]) -> Dict[str, Any]:
        """POST request"""
        url = f"{self.base_url}{endpoint}"
        
        try:
            response = requests.post(url, json=data, timeout=self.timeout)
            return {
                'success': response.status_code == 200,
                'data': response.json() if response.headers.get('content-type', '').startswith('application/json') else response.text,
                'status_code': response.status_code,
                'response_time': response.elapsed.total_seconds()
            }
        except requests.exceptions.RequestException as e:
            return {
                'success': False,
                'error': str(e),
                'response_time': 0
            }

# Usage
client = RESTClient('https://api.example.com')

# GET request
response = client.get('/users/123')
print(f"GET response: {response}")

# POST request
data = {'name': 'John', 'email': 'john@example.com'}
response = client.post('/users', data)
print(f"POST response: {response}")
```

### GraphQL Pattern
```python
import requests
from typing import Dict, Any, Optional

class GraphQLClient:
    def __init__(self, endpoint: str):
        self.endpoint = endpoint
        self.headers = {'Content-Type': 'application/json'}
    
    def query(self, query: str, variables: Dict = None) -> Dict[str, Any]:
        """Execute GraphQL query"""
        payload = {
            'query': query,
            'variables': variables
        }
        
        try:
            response = requests.post(
                self.endpoint,
                json=payload,
                headers=self.headers,
                timeout=30
            )
            
            return {
                'success': response.status_code == 200,
                'data': response.json(),
                'response_time': response.elapsed.total_seconds()
            }
        except requests.exceptions.RequestException as e:
            return {
                'success': False,
                'error': str(e),
                'response_time': 0
            }
    
    def mutation(self, mutation: str, variables: Dict = None) -> Dict[str, Any]:
        """Execute GraphQL mutation"""
        payload = {
            'query': mutation,
            'variables': variables
        }
        
        try:
            response = requests.post(
                self.endpoint,
                json=payload,
                headers=self.headers,
                timeout=30
            )
            
            return {
                'success': response.status_code == 200,
                'data': response.json(),
                'response_time': response.elapsed.total_seconds()
            }
        except requests.exceptions.RequestException as e:
            return {
                'success': False,
                'error': str(e),
                'response_time': 0
            }

# Usage
client = GraphQLClient('https://api.example.com/graphql')

# Query
query = """
query GetUser($id: ID!) {
  user(id: $id) {
    id
    name
    email
    posts {
      id
      title
      content
    }
  }
}
"""
variables = {'id': '123'}
response = client.query(query, variables)
print(f"GraphQL query response: {response}")

# Mutation
mutation = """
mutation CreateUser($input: CreateUserInput!) {
  createUser(input: $input) {
    id
    name
    email
  }
}
"""
input_data = {'name': 'Jane', 'email': 'jane@example.com'}
response = client.mutation(mutation, input_data)
print(f"GraphQL mutation response: {response}")
```

## Message-Based Patterns

### Publish-Subscribe Pattern
```python
import threading
import time
import queue
from typing import Dict, List, Callable, Any
from dataclasses import dataclass

@dataclass
class Message:
    topic: str
    data: Any
    publisher_id: str = None
    timestamp: float

class MessageBroker:
    def __init__(self):
        self.topics = {}  # topic -> list of subscribers
        self.message_history = []
    
    def subscribe(self, topic: str, callback: Callable[[Message], None]):
        """Subscribe to a topic"""
        if topic not in self.topics:
            self.topics[topic] = []
        
        self.topics[topic].append(callback)
    
    def publish(self, message: Message):
        """Publish message to all subscribers"""
        if message.topic not in self.topics:
            return
        
        self.message_history.append(message)
        
        # Deliver to all subscribers
        for callback in self.topics[message.topic]:
            try:
                callback(message)
            except Exception as e:
                print(f"Error delivering message to subscriber: {e}")
    
    def get_topic_stats(self) -> Dict[str, int]:
        """Get statistics for each topic"""
        stats = {}
        
        for topic, subscribers in self.topics.items():
            stats[topic] = len(subscribers)
        
        return stats

# Usage
broker = MessageBroker()

# Define subscribers
def user_notification_handler(message: Message):
    print(f"User received notification: {message.data}")

def analytics_handler(message: Message):
    print(f"Analytics received event: {message.data}")

# Subscribe to topics
broker.subscribe('user.notifications', user_notification_handler)
broker.subscribe('user.events', analytics_handler)

# Publish messages
notification_msg = Message(
    topic="user.notifications",
    data={"type": "welcome", "message": "Welcome to our platform!"},
    publisher_id="system"
)

event_msg = Message(
    topic="user.events",
    data={"type": "login", "user_id": "user123", "timestamp": time.time()},
    publisher_id="auth_service"
)

broker.publish(notification_msg)
broker.publish(event_msg)

# Get statistics
stats = broker.get_topic_stats()
print(f"Topic statistics: {stats}")
```

### Message Queue Pattern
```python
import threading
import time
import queue
from typing import Dict, Any, List, Callable
from dataclasses import dataclass

@dataclass
class Task:
    task_id: str
    task_type: str
    data: Dict[str, Any]
    priority: int = 0
    created_at: float = None

class TaskQueue:
    def __init__(self):
        self.queue = queue.PriorityQueue()
        self.workers = []
        self.task_results = {}
    
    def add_worker(self, worker_id: str, task_handler: Callable[[Task], Any]):
        """Add a worker to the queue"""
        self.workers.append({
            'worker_id': worker_id,
            'handler': task_handler,
            'current_task': None,
            'completed_tasks': 0,
            'failed_tasks': 0
        })
    
    def enqueue_task(self, task: Task):
        """Add task to the queue"""
        # Use negative priority for max-heap behavior
        priority = -task.priority
        self.queue.put((priority, task.created_at, task))
    
    def start_processing(self):
        """Start task processing"""
        def worker_loop(worker: Dict):
            worker['handler'] = worker['handler']
            worker['running'] = True
            
            while worker['running']:
                try:
                    # Get next task from queue
                    try:
                        priority, created_at, task = self.queue.get(timeout=1)
                        task.priority = -priority  # Convert back to positive
                    except queue.Empty:
                        continue
                    
                    worker['current_task'] = task
                    
                    # Process task
                    result = worker['handler'](task)
                    
                    if result:
                        worker['completed_tasks'] += 1
                        self.task_results[task.task_id] = {
                            'status': 'completed',
                            'result': result,
                            'worker_id': worker['worker_id'],
                            'completed_at': time.time()
                        }
                    else:
                        worker['failed_tasks'] += 1
                        self.task_results[task.task_id] = {
                            'status': 'failed',
                            'error': 'Task processing failed',
                            'worker_id': worker['worker_id'],
                            'failed_at': time.time()
                        }
                    
                    worker['current_task'] = None
                
                except Exception as e:
                    worker['failed_tasks'] += 1
                    self.task_results[task.task_id] = {
                        'status': 'failed',
                        'error': str(e),
                        'worker_id': worker['worker_id'],
                        'failed_at': time.time()
                    }
        
        # Start all workers
        worker_threads = []
        for worker in self.workers:
            thread = threading.Thread(target=worker_loop, args=(worker,), daemon=True)
            thread.start()
            worker_threads.append(thread)
        
        return worker_threads
    
    def get_queue_stats(self) -> Dict[str, Any]:
        """Get queue statistics"""
        total_tasks = len(self.task_results)
        completed_tasks = sum(1 for result in self.task_results.values() if result['status'] == 'completed')
        failed_tasks = sum(1 for result in self.task_results.values() if result['status'] == 'failed')
        
        worker_stats = []
        for worker in self.workers:
            worker_stats.append({
                'worker_id': worker['worker_id'],
                'completed_tasks': worker['completed_tasks'],
                'failed_tasks': worker['failed_tasks'],
                'success_rate': worker['completed_tasks'] / (worker['completed_tasks'] + worker['failed_tasks']) if (worker['completed_tasks'] + worker['failed_tasks']) > 0 else 0,
                'current_task': worker['current_task'].task_id if worker['current_task'] else None
            })
        
        return {
            'queue_size': self.queue.qsize(),
            'total_tasks': total_tasks,
            'completed_tasks': completed_tasks,
            'failed_tasks': failed_tasks,
            'success_rate': completed_tasks / total_tasks if total_tasks > 0 else 0,
            'workers': worker_stats
        }

# Task handlers
def process_email_task(task: Task) -> Dict[str, Any]:
    """Process email sending task"""
    email_data = task.data
    
    # Simulate email sending
    print(f"Sending email to {email_data['to']}: {email_data['subject']}")
    time.sleep(1)  # Simulate processing time
    
    return {
        'sent': True,
        'recipient': email_data['to'],
        'subject': email_data['subject']
    }

def process_image_task(task: Task) -> Dict[str, Any]:
    """Process image resizing task"""
    image_data = task.data
    
    # Simulate image processing
    print(f"Resizing image {image_data['filename']} to {image_data['size']}")
    time.sleep(2)  # Simulate processing time
    
    return {
        'resized': True,
        'original_size': image_data['original_size'],
        'new_size': image_data['size'],
        'output_file': f"resized_{image_data['filename']}"
    }

# Usage
task_queue = TaskQueue()

# Add workers
task_queue.add_worker('worker_1', process_email_task)
task_queue.add_worker('worker_2', process_email_task)
task_queue.add_worker('worker_3', process_image_task)

# Enqueue tasks
email_task = Task(
    task_id="email_1",
    task_type="email",
    data={"to": "user1@example.com", "subject": "Welcome!", "body": "Welcome to our service"},
    priority=1
)

image_task = Task(
    task_id="image_1",
    task_type="image_resize",
    data={"filename": "photo.jpg", "size": "800x600", "original_size": "1920x1080"},
    priority=2
)

task_queue.enqueue_task(email_task)
task_queue.enqueue_task(image_task)

# Start processing
worker_threads = task_queue.start_processing()

# Let tasks process
time.sleep(5)

# Get statistics
stats = task_queue.get_queue_stats()
print(f"Queue statistics: {stats}")

# Stop processing
for worker in task_queue.workers:
    worker['running'] = False

for thread in worker_threads:
    thread.join()
```

## Event-Driven Architecture

### Event Sourcing Pattern
```python
import json
import time
from typing import Dict, List, Any
from dataclasses import dataclass, asdict
from enum import Enum

class EventType(Enum):
    USER_CREATED = "user_created"
    USER_UPDATED = "user_updated"
    USER_DELETED = "user_deleted"
    ORDER_CREATED = "order_created"
    ORDER_UPDATED = "order_updated"
    ORDER_CANCELLED = "order_cancelled"

@dataclass
class Event:
    event_id: str
    event_type: EventType
    aggregate_id: str
    aggregate_type: str
    data: Dict[str, Any]
    timestamp: float
    version: int
    
    def to_dict(self) -> Dict[str, Any]:
        return asdict(self)

class EventStore:
    def __init__(self):
        self.events = []  # In production, use persistent storage
        self.snapshots = {}  # aggregate_id -> snapshot
    
    def save_event(self, event: Event):
        """Save event to event store"""
        self.events.append(event)
        
        # Update snapshot if needed
        self._update_snapshot(event)
    
    def get_events(self, aggregate_id: str, from_version: int = 0) -> List[Event]:
        """Get events for an aggregate from specific version"""
        return [
            event for event in self.events
            if event.aggregate_id == aggregate_id and event.version > from_version
        ]
    
    def get_snapshot(self, aggregate_id: str) -> Dict[str, Any]:
        """Get latest snapshot of an aggregate"""
        return self.snapshots.get(aggregate_id, {})
    
    def _update_snapshot(self, event: Event):
        """Update snapshot based on event"""
        if event.aggregate_type == "user":
            self._update_user_snapshot(event)
        elif event.aggregate_type == "order":
            self._update_order_snapshot(event)
    
    def _update_user_snapshot(self, event: Event):
        """Update user snapshot based on event"""
        if event.aggregate_id not in self.snapshots:
            self.snapshots[event.aggregate_id] = {}
        
        snapshot = self.snapshots[event.aggregate_id]
        
        if event.event_type == EventType.USER_CREATED:
            snapshot.update(event.data)
            snapshot['version'] = event.version
        elif event.event_type == EventType.USER_UPDATED:
            snapshot.update(event.data)
            snapshot['version'] = event.version
        elif event.event_type == EventType.USER_DELETED:
            snapshot['deleted'] = True
            snapshot['deleted_at'] = event.timestamp
            snapshot['version'] = event.version
    
    def _update_order_snapshot(self, event: Event):
        """Update order snapshot based on event"""
        if event.aggregate_id not in self.snapshots:
            self.snapshots[event.aggregate_id] = {}
        
        snapshot = self.snapshots[event.aggregate_id]
        
        if event.event_type == EventType.ORDER_CREATED:
            snapshot.update(event.data)
            snapshot['status'] = 'created'
            snapshot['version'] = event.version
        elif event.event_type == EventType.ORDER_UPDATED:
            snapshot.update(event.data)
            snapshot['version'] = event.version
        elif event.event_type == EventType.ORDER_CANCELLED:
            snapshot['status'] = 'cancelled'
            snapshot['cancelled_at'] = event.timestamp
            snapshot['version'] = event.version

class Aggregate:
    def __init__(self, aggregate_id: str, event_store: EventStore):
        self.aggregate_id = aggregate_id
        self.event_store = event_store
        self.version = 0
        self.state = {}
    
    def load_from_history(self):
        """Rebuild aggregate state from event history"""
        events = self.event_store.get_events(self.aggregate_id)
        
        for event in events:
            self.apply_event(event)
    
    def load_from_snapshot(self):
        """Load aggregate from latest snapshot"""
        snapshot = self.event_store.get_snapshot(self.aggregate_id)
        
        if snapshot:
            self.state = snapshot.copy()
            self.version = snapshot.get('version', 0)
        else:
            self.load_from_history()
    
    def apply_event(self, event: Event):
        """Apply event to aggregate state"""
        if event.aggregate_id != self.aggregate_id:
            raise ValueError("Event aggregate ID mismatch")
        
        if event.version != self.version + 1:
            raise ValueError(f"Event version mismatch. Expected {self.version + 1}, got {event.version}")
        
        # Apply event based on type
        if event.event_type == EventType.USER_CREATED:
            self._apply_user_created(event)
        elif event.event_type == EventType.USER_UPDATED:
            self._apply_user_updated(event)
        elif event.event_type == EventType.ORDER_CREATED:
            self._apply_order_created(event)
        elif event.event_type == EventType.ORDER_UPDATED:
            self._apply_order_updated(event)
    
    def _apply_user_created(self, event: Event):
        """Apply user created event"""
        self.state.update(event.data)
    
    def _apply_user_updated(self, event: Event):
        """Apply user updated event"""
        self.state.update(event.data)
    
    def _apply_order_created(self, event: Event):
        """Apply order created event"""
        self.state.update(event.data)
        self.state['status'] = 'created'
    
    def _apply_order_updated(self, event: Event):
        """Apply order updated event"""
        self.state.update(event.data)

# Usage
event_store = EventStore()

# Create events
user_created_event = Event(
    event_id="evt_1",
    event_type=EventType.USER_CREATED,
    aggregate_id="user_123",
    aggregate_type="user",
    data={"name": "John Doe", "email": "john@example.com", "age": 30},
    timestamp=time.time(),
    version=1
)

user_updated_event = Event(
    event_id="evt_2",
    event_type=EventType.USER_UPDATED,
    aggregate_id="user_123",
    aggregate_type="user",
    data={"age": 31},  # User had birthday
    timestamp=time.time(),
    version=2
)

order_created_event = Event(
    event_id="evt_3",
    event_type=EventType.ORDER_CREATED,
    aggregate_id="order_456",
    aggregate_type="order",
    data={"user_id": "user_123", "total": 99.99, "items": ["item1", "item2"]},
    timestamp=time.time(),
    version=1
)

# Save events
event_store.save_event(user_created_event)
event_store.save_event(user_updated_event)
event_store.save_event(order_created_event)

# Rebuild aggregates
user_aggregate = Aggregate("user_123", event_store)
user_aggregate.load_from_snapshot()

order_aggregate = Aggregate("order_456", event_store)
order_aggregate.load_from_snapshot()

print(f"User aggregate state: {user_aggregate.state}")
print(f"User aggregate version: {user_aggregate.version}")

print(f"Order aggregate state: {order_aggregate.state}")
print(f"Order aggregate version: {order_aggregate.version}")
```

## Real-World Examples

### Netflix Communication Architecture
```
Components:
- Event-driven architecture
- Kafka for event streaming
- gRPC for synchronous calls
- GraphQL for API queries
- WebSocket for real-time updates

Patterns:
- Event sourcing for playback
- CQRS for read/write separation
- Circuit breakers for resilience
- Bulkheading for isolation
```

### Uber Communication Architecture
```
Components:
- Real-time location sharing
- Pub/sub for driver-passenger matching
- WebSocket for live tracking
- Message queues for trip events
- REST APIs for booking

Patterns:
- Geospatial pub/sub
- Command/query separation
- Event-driven notifications
- Retry mechanisms with exponential backoff
```

### Amazon Communication Architecture
```
Components:
- Service mesh (AWS App Mesh)
- EventBridge for event routing
- SQS for message queues
- SNS for pub/sub messaging
- API Gateway for REST APIs

Patterns:
- Async microservices communication
- Event-driven order processing
- Fan-out notifications
- Dead letter queues
```

## Interview Tips

### Common Questions
1. "When would you use synchronous vs asynchronous communication?"
2. "What's the difference between pub/sub and message queues?"
3. "How does event sourcing work?"
4. "What is CQRS and when would you use it?"
5. "How would you handle message ordering?"

### Answer Framework
1. **Understand Requirements**: Consistency, latency, scalability
2. **Choose Pattern**: Synchronous vs asynchronous based on needs
3. **Design Communication**: Protocols, formats, error handling
4. **Consider Reliability**: Retry, circuit breakers, dead letters
5. **Plan for Scale**: Partitioning, load balancing, monitoring

### Key Points to Emphasize
- Trade-offs between consistency and availability
- Communication pattern selection criteria
- Error handling and retry strategies
- Message ordering guarantees
- Monitoring and observability

## Practice Problems

1. **Design real-time chat system**
2. **Implement event sourcing for e-commerce**
3. **Create CQRS system for banking**
4. **Design pub/sub notification system**
5. **Build message queue for job processing**

## Further Reading

- **Books**: "Enterprise Integration Patterns" by Hohpe and Woolf
- **Concepts**: Event Sourcing, CQRS, Message Queues, Pub/Sub Systems
- **Documentation**: Kafka Documentation, RabbitMQ Guide, AWS SQS/SNS Documentation
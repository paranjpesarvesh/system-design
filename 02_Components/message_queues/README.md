# Message Queues and Event Systems

## Definition

Message queues are asynchronous communication mechanisms that enable different components of a distributed system to communicate and coordinate by sending messages through intermediaries, decoupling senders from receivers.

## Core Concepts

### Producer-Consumer Pattern
```
Producer → Message Queue → Consumer
    ↓                           ↓
Send Message              Receive Message
                           Process Message
                           Acknowledge
```

**Key Characteristics:**
- **Asynchronous**: Producer doesn't wait for consumer
- **Decoupled**: Producer and consumer don't know about each other
- **Durable**: Messages persist until consumed
- **Scalable**: Multiple producers/consumers possible

### Message Queue Components
```
┌─────────────────────────────────────────────────────────┐
│                 Message Queue System                    │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │   Producer  │  │    Queue    │  │   Consumer  │      │
│  │             │  │             │  │             │      │
│  │ Send Message│→ │   Store     │→ │Get Message  │      │
│  │             │  │   Messages  │  │             │      │
│  └─────────────┘  │             │  │Process      │      │
│                   │   Forward   │  │Message      │      │
│  ┌─────────────┐  │   Messages  │  │             │      │
│  │   Producer  │  │             │  │Acknowledge  │      │
│  │             │  └─────────────┘  │             │      │
│  │ Send Message│                   └─────────────┘      │
│  └─────────────┘                                        │
└─────────────────────────────────────────────────────────┘
```

## Types of Message Queues

### Point-to-Point (Queue)
```
Producer A → Queue → Consumer A
Producer B → Queue → Consumer B
Producer C → Queue → Consumer C

Each message consumed by exactly one consumer
```

**Use Cases:**
- Task processing
- Order processing
- Email sending
- Data processing pipelines

### Publish-Subscribe (Topic)
```
Producer → Topic
    ↓
┌─────────────────────────────────┐
│         Subscribers             │
│                                 │
│  Consumer A (gets all)          │
│  Consumer B (filtered)          │
│  Consumer C (selected topics)   │
└─────────────────────────────────┘

Each message can be consumed by multiple subscribers
```

**Use Cases:**
- Event notifications
- Real-time updates
- News feeds
- System monitoring

## Popular Message Queue Systems

### RabbitMQ
```
Features:
- AMQP protocol
- Flexible routing
- Message acknowledgments
- Clustering support
- Management UI

Architecture:
Exchange → Queue → Consumer
   ↓         ↓       ↓
Routing   Message  Processing
```

**Implementation:**
```python
import pika

class RabbitMQProducer:
    def __init__(self, host='localhost'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()

    def send_message(self, queue_name, message):
        self.channel.queue_declare(queue=queue_name, durable=True)

        self.channel.basic_publish(
            exchange='',
            routing_key=queue_name,
            body=message,
            properties=pika.BasicProperties(
                delivery_mode=2,  # make message persistent
            )
        )
        print(f" [x] Sent '{message}' to queue '{queue_name}'")

    def close(self):
        self.connection.close()

class RabbitMQConsumer:
    def __init__(self, host='localhost'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()

    def consume_messages(self, queue_name, callback):
        self.channel.queue_declare(queue=queue_name, durable=True)

        self.channel.basic_qos(prefetch_count=1)
        self.channel.basic_consume(
            queue=queue_name,
            on_message_callback=callback,
            auto_ack=False
        )

        print(f" [*] Waiting for messages in queue '{queue_name}'")
        self.channel.start_consuming()

    def close(self):
        self.connection.close()

# Usage
def message_callback(ch, method, properties, body):
    print(f" [x] Received '{body.decode()}'")
    # Process message
    ch.basic_ack(delivery_tag=method.delivery_tag)

producer = RabbitMQProducer()
producer.send_message('task_queue', 'Hello World!')
producer.close()

consumer = RabbitMQConsumer()
consumer.consume_messages('task_queue', message_callback)
```

### Apache Kafka
```
Features:
- Distributed streaming platform
- High throughput
- Partitioned topics
- Replication
- Exactly-once semantics

Architecture:
Producer → Kafka Cluster → Consumer
    ↓         ↓           ↓
  Topic   Partitions   Consumer Groups
```

**Implementation:**
```python
from kafka import KafkaProducer, KafkaConsumer
import json

class KafkaProducer:
    def __init__(self, bootstrap_servers=['localhost:9092']):
        self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            key_serializer=lambda k: k.encode('utf-8') if k else None
        )

    def send_message(self, topic, message, key=None):
        future = self.producer.send(topic, value=message, key=key)

        # Block until message is sent
        record_metadata = future.get(timeout=10)

        print(f"Message sent to {record_metadata.topic} "
              f"partition {record_metadata.partition} "
              f"offset {record_metadata.offset}")

    def close(self):
        self.producer.flush()
        self.producer.close()

class KafkaConsumer:
    def __init__(self, topics, group_id, bootstrap_servers=['localhost:9092']):
        self.consumer = KafkaConsumer(
            *topics,
            group_id=group_id,
            bootstrap_servers=bootstrap_servers,
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            key_deserializer=lambda k: k.decode('utf-8') if k else None,
            auto_offset_reset='earliest',
            enable_auto_commit=False
        )

    def consume_messages(self, callback):
        print(f"Starting consumer for group: {self.consumer.config['group_id']}")

        try:
            for message in self.consumer:
                print(f"Received message from {message.topic}: {message.value}")

                # Process message
                try:
                    callback(message)
                    # Commit offset after successful processing
                    self.consumer.commit()
                except Exception as e:
                    print(f"Error processing message: {e}")
                    # Handle error, don't commit offset

        except KeyboardInterrupt:
            print("Stopping consumer...")
        finally:
            self.consumer.close()

# Usage
def process_order(message):
    order_data = message.value
    print(f"Processing order: {order_data['order_id']}")
    # Business logic here

# Producer
producer = KafkaProducer()
producer.send_message('orders', {
    'order_id': '12345',
    'customer_id': 'cust_789',
    'amount': 99.99,
    'items': ['item1', 'item2']
}, key='order_12345')

# Consumer
consumer = KafkaConsumer(['orders'], 'order_processor')
consumer.consume_messages(process_order)
```

### Redis Pub/Sub
```
Features:
- In-memory
- Fast
- Simple API
- Pattern matching
- No persistence (by default)

Architecture:
Publisher → Redis Channel → Subscriber
```

**Implementation:**
```python
import redis
import json
import threading

class RedisPublisher:
    def __init__(self, host='localhost', port=6379):
        self.redis_client = redis.Redis(host=host, port=port, decode_responses=True)

    def publish_message(self, channel, message):
        message_json = json.dumps(message)
        result = self.redis_client.publish(channel, message_json)
        print(f"Published to {result} subscribers on channel '{channel}'")
        return result

class RedisSubscriber:
    def __init__(self, host='localhost', port=6379):
        self.redis_client = redis.Redis(host=host, port=port, decode_responses=True)
        self.pubsub = self.redis_client.pubsub()

    def subscribe_to_channel(self, channel, callback):
        self.pubsub.subscribe(channel)

        print(f"Subscribed to channel: {channel}")

        for message in self.pubsub.listen():
            if message['type'] == 'message':
                try:
                    data = json.loads(message['data'])
                    callback(message['channel'], data)
                except json.JSONDecodeError:
                    callback(message['channel'], message['data'])

    def subscribe_to_pattern(self, pattern, callback):
        self.pubsub.psubscribe(pattern)

        print(f"Subscribed to pattern: {pattern}")

        for message in self.pubsub.listen():
            if message['type'] == 'pmessage':
                try:
                    data = json.loads(message['data'])
                    callback(message['channel'], data)
                except json.JSONDecodeError:
                    callback(message['channel'], message['data'])

# Usage
def handle_notification(channel, data):
    print(f"Received notification on {channel}: {data}")

# Publisher
publisher = RedisPublisher()
publisher.publish_message('notifications', {
    'type': 'order_completed',
    'order_id': '12345',
    'user_id': 'user_789'
})

# Subscriber
subscriber = RedisSubscriber()
# Run in separate thread
thread = threading.Thread(
    target=subscriber.subscribe_to_channel,
    args=('notifications', handle_notification)
)
thread.start()
```

## Message Queue Patterns

### Work Queue Pattern
```
Producer → Queue → [Worker 1, Worker 2, Worker 3]
    ↓
Distribute work among multiple workers
```

**Implementation:**
```python
class WorkQueue:
    def __init__(self, queue_name, num_workers=3):
        self.queue_name = queue_name
        self.num_workers = num_workers
        self.workers = []

    def add_task(self, task_data):
        # Add task to queue
        message_queue.put((self.queue_name, task_data))

    def start_workers(self):
        for i in range(self.num_workers):
            worker = Worker(f"worker_{i}", self.queue_name)
            worker.start()
            self.workers.append(worker)

    def stop_workers(self):
        for worker in self.workers:
            worker.stop()

class Worker:
    def __init__(self, worker_id, queue_name):
        self.worker_id = worker_id
        self.queue_name = queue_name
        self.running = False

    def start(self):
        self.running = True
        thread = threading.Thread(target=self._process_tasks)
        thread.start()

    def stop(self):
        self.running = False

    def _process_tasks(self):
        while self.running:
            try:
                # Get task from queue
                task = message_queue.get(timeout=1)
                if task:
                    queue_name, task_data = task
                    if queue_name == self.queue_name:
                        self._execute_task(task_data)
                        message_queue.task_done()
            except:
                continue

    def _execute_task(self, task_data):
        print(f"{self.worker_id} processing task: {task_data}")
        # Simulate work
        time.sleep(random.uniform(0.1, 1.0))
        print(f"{self.worker_id} completed task: {task_data['task_id']}")
```

### Publish-Subscribe Pattern
```
Publisher → Topic → [Subscriber 1, Subscriber 2, Subscriber 3]
    ↓
Broadcast message to all interested subscribers
```

**Implementation:**
```python
class EventBroker:
    def __init__(self):
        self.subscribers = {}
        self.message_queue = queue.Queue()

    def subscribe(self, event_type, callback):
        if event_type not in self.subscribers:
            self.subscribers[event_type] = []
        self.subscribers[event_type].append(callback)

    def unsubscribe(self, event_type, callback):
        if event_type in self.subscribers:
            self.subscribers[event_type].remove(callback)

    def publish(self, event_type, event_data):
        event = {
            'type': event_type,
            'data': event_data,
            'timestamp': time.time()
        }
        self.message_queue.put(event)

    def start_processing(self):
        while True:
            try:
                event = self.message_queue.get(timeout=1)
                self._process_event(event)
            except:
                continue

    def _process_event(self, event):
        event_type = event['type']
        if event_type in self.subscribers:
            for callback in self.subscribers[event_type]:
                try:
                    callback(event)
                except Exception as e:
                    print(f"Error in subscriber callback: {e}")

# Usage
def handle_user_created(event):
    user_data = event['data']
    print(f"New user created: {user_data['username']}")

def handle_user_updated(event):
    user_data = event['data']
    print(f"User updated: {user_data['username']}")

broker = EventBroker()
broker.subscribe('user.created', handle_user_created)
broker.subscribe('user.updated', handle_user_updated)

# Start event processing in background
thread = threading.Thread(target=broker.start_processing)
thread.start()

# Publish events
broker.publish('user.created', {
    'user_id': '123',
    'username': 'john_doe',
    'email': 'john@example.com'
})
```

### Request-Reply Pattern
```
Client → Request Queue → Server
    ↓                      ↓
Response Queue ← Server ← Process Request
    ↓
Client receives response
```

**Implementation:**
```python
import uuid

class RPCClient:
    def __init__(self, connection):
        self.connection = connection
        self.callback_queue = ''
        self.response = None
        self.corr_id = None
        self.setup_reply_queue()

    def setup_reply_queue(self):
        result = self.connection.channel.queue_declare(queue='', exclusive=True)
        self.callback_queue = result.method.queue

        self.connection.channel.basic_consume(
            queue=self.callback_queue,
            on_message_callback=self.on_response,
            auto_ack=True
        )

    def on_response(self, ch, method, props, body):
        if self.corr_id == props.correlation_id:
            self.response = body

    def call(self, routing_key, message):
        self.response = None
        self.corr_id = str(uuid.uuid4())

        self.connection.channel.basic_publish(
            exchange='',
            routing_key=routing_key,
            properties=pika.BasicProperties(
                reply_to=self.callback_queue,
                correlation_id=self.corr_id,
            ),
            body=str(message)
        )

        while self.response is None:
            self.connection.process_data_events(time_limit=None)

        return self.response.decode()

class RPCServer:
    def __init__(self, connection):
        self.connection = connection

    def start_server(self, queue_name):
        self.connection.channel.queue_declare(queue=queue_name)
        self.connection.channel.basic_qos(prefetch_count=1)
        self.connection.channel.basic_consume(
            queue=queue_name,
            on_message_callback=self.on_request
        )
        print(f" [x] Awaiting RPC requests on {queue_name}")
        self.connection.channel.start_consuming()

    def on_request(self, ch, method, props, body):
        print(f" [.] Received request: {body.decode()}")

        # Process request
        response = self.process_request(body.decode())

        # Send response
        ch.basic_publish(
            exchange='',
            routing_key=props.reply_to,
            properties=pika.BasicProperties(
                correlation_id=props.correlation_id
            ),
            body=str(response)
        )
        ch.basic_ack(delivery_tag=method.delivery_tag)

    def process_request(self, request):
        # Example: Fibonacci calculation
        n = int(request)
        if n == 0:
            return 0
        elif n == 1:
            return 1
        else:
            a, b = 0, 1
            for _ in range(2, n + 1):
                a, b = b, a + b
            return b
```

## Message Queue Configuration

### Kafka Cluster Configuration
```properties
# server.properties
broker.id=1
listeners=PLAINTEXT://localhost:9092
advertised.listeners=PLAINTEXT://localhost:9092
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
num.partitions=3
num.recovery.threads.per.data.dir=1
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
zookeeper.connect=localhost:2181
zookeeper.connection.timeout.ms=18000
group.initial.rebalance.delay.ms=0
```

### RabbitMQ Configuration
```erlang
% rabbitmq.config
[
  {rabbit, [
    {tcp_listeners, [{"0.0.0.0", 5672}]},
    {ssl_listeners, []},
    {vm_memory_high_watermark, 0.4},
    {disk_free_limit, "2GB"},
    {heartbeat, 60},
    {frame_max, 131072},
    {channel_max, 2047},
    {max_message_size, 134217728}
  ]},
  {rabbitmq_management, [
    {listener, [{port, 15672}]}
  ]}
].
```

## Performance Optimization

### Kafka Performance Tuning
```python
# Producer optimization
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    # Batch messages for better throughput
    batch_size=16384,
    # Wait longer for batching
    linger_ms=10,
    # Compression for network efficiency
    compression_type='gzip',
    # More memory for buffering
    buffer_memory=33554432,
    # Increase retries for reliability
    retries=3,
    # Acknowledge all replicas for durability
    acks='all'
)

# Consumer optimization
consumer = KafkaConsumer(
    'my-topic',
    bootstrap_servers=['localhost:9092'],
    # Fetch larger batches
    fetch_max_bytes=52428800,
    # Fetch more messages per request
    max_poll_records=500,
    # Increase session timeout
    session_timeout_ms=30000,
    # Optimize for throughput
    fetch_min_bytes=1,
    # Auto-commit less frequently
    auto_commit_interval_ms=1000
)
```

### RabbitMQ Performance Tuning
```python
# Connection optimization
connection = pika.BlockingConnection(
    pika.ConnectionParameters(
        host='localhost',
        # Use larger frame size
        frame_max=131072,
        # Increase heartbeat interval
        heartbeat=600,
        # Connection timeout
        connection_attempts=3,
        # Retry delay
        retry_delay=5
    )
)

# Channel optimization
channel = connection.channel()
# Prefetch more messages
channel.basic_qos(prefetch_count=100)
# Use publisher confirms
channel.confirm_delivery()
```

## Monitoring and Observability

### Kafka Monitoring
```python
from kafka import KafkaAdminClient
from kafka.admin import ConfigResourceType

class KafkaMonitor:
    def __init__(self, bootstrap_servers):
        self.admin_client = KafkaAdminClient(
            bootstrap_servers=bootstrap_servers
        )

    def get_topic_metrics(self, topic_name):
        # Get topic configuration
        configs = self.admin_client.describe_configs([
            ConfigResource(ConfigResourceType.TOPIC, topic_name)
        ])

        # Get consumer group offsets
        consumer_groups = self.admin_client.list_consumer_groups()

        return {
            'topic': topic_name,
            'configs': configs,
            'consumer_groups': consumer_groups
        }

    def get_broker_metrics(self):
        # Get broker information
        cluster_metadata = self.admin_client.describe_cluster()

        return {
            'cluster_id': cluster_metadata.cluster_id,
            'controllers': cluster_metadata.controllers,
            'brokers': cluster_metadata.brokers
        }

# Custom metrics collection
class KafkaMetricsCollector:
    def __init__(self, consumer):
        self.consumer = consumer
        self.metrics = {
            'messages_consumed': 0,
            'bytes_consumed': 0,
            'lag': 0
        }

    def collect_metrics(self):
        for message in self.consumer:
            self.metrics['messages_consumed'] += 1
            self.metrics['bytes_consumed'] += len(message.value)

            # Calculate lag
            partitions = self.consumer.committed()
            for partition, offset in partitions.items():
                high_water = self.consumer.highwater(partition)
                if high_water:
                    self.metrics['lag'] += high_water - offset
```

### RabbitMQ Monitoring
```python
import requests
import json

class RabbitMQMonitor:
    def __init__(self, host='localhost', port=15672, username='guest', password='guest'):
        self.base_url = f"http://{host}:{port}/api"
        self.auth = (username, password)

    def get_queue_metrics(self):
        response = requests.get(f"{self.base_url}/queues", auth=self.auth)
        queues = response.json()

        metrics = []
        for queue in queues:
            metrics.append({
                'name': queue['name'],
                'messages': queue['messages'],
                'messages_ready': queue['messages_ready'],
                'messages_unacknowledged': queue['messages_unacknowledged'],
                'message_stats': queue.get('message_stats', {}),
                'memory': queue['memory']
            })

        return metrics

    def get_connection_metrics(self):
        response = requests.get(f"{self.base_url}/connections", auth=self.auth)
        connections = response.json()

        return {
            'total_connections': len(connections),
            'connections': connections
        }

    def get_overview(self):
        response = requests.get(f"{self.base_url}/overview", auth=self.auth)
        return response.json()
```

## Real-World Examples

### Netflix Message Queue Architecture
```
Components:
- Kafka for event streaming
- RabbitMQ for task queues
- SNS/SQS for cloud messaging

Use Cases:
- Video processing pipeline
- Recommendation updates
- User activity tracking
- System monitoring
```

### Uber Message Queue Architecture
```
Components:
- Kafka for real-time events
- Redis Pub/Sub for notifications
- Custom queue system for ride matching

Use Cases:
- Trip lifecycle events
- Driver-passenger matching
- Real-time location updates
- Payment processing
```

### Twitter Message Queue Architecture
```
Components:
- Kafka for event streaming
- Custom queues for timeline processing
- Redis for real-time notifications

Use Cases:
- Tweet processing
- Timeline generation
- Notification delivery
- Analytics collection
```

## Interview Tips

### Common Questions
1. "When would you use a message queue?"
2. "What's the difference between Kafka and RabbitMQ?"
3. "How do you handle message ordering?"
4. "What happens if a consumer fails?"
5. "How do you ensure message delivery?"

### Answer Framework
1. **Understand Requirements**: Throughput, latency, ordering, durability
2. **Choose System**: Kafka vs RabbitMQ vs Redis based on needs
3. **Design Architecture**: Producers, consumers, topics/queues
4. **Handle Failures**: Retries, dead letter queues, monitoring
5. **Consider Trade-offs**: Performance vs complexity vs reliability

### Key Points to Emphasize
- Decoupling benefits
- Scalability advantages
- Failure handling strategies
- Message ordering guarantees
- Performance considerations

## Practice Problems

1. **Design order processing system with message queues**
2. **Implement real-time notification system**
3. **Create data processing pipeline**
4. **Design event-driven microservices**
5. **Build reliable task queue system**

## Further Reading

- **Books**: "Kafka: The Definitive Guide" by Neha Narkhede, "RabbitMQ in Action" by Alvaro Videla
- **Documentation**: Apache Kafka Documentation, RabbitMQ Documentation
- **Concepts**: Event Sourcing, CQRS, Microservices Patterns

# Network Fundamentals

## Definition

Networking in system design refers to the communication protocols, architectures, and patterns that enable different components of a distributed system to communicate with each other efficiently and reliably.

## OSI Model

### 7 Layers of OSI Model
```
┌─────────────────────────────────────┐
│ 7. Application Layer (HTTP, HTTPS)  │
├─────────────────────────────────────┤
│ 6. Presentation Layer (SSL/TLS)     │
├─────────────────────────────────────┤
│ 5. Session Layer (NetBIOS, RPC)     │
├─────────────────────────────────────┤
│ 4. Transport Layer (TCP, UDP)       │
├─────────────────────────────────────┤
│ 3. Network Layer (IP, ICMP)         │
├─────────────────────────────────────┤
│ 2. Data Link Layer (Ethernet, WiFi) │
├─────────────────────────────────────┤
│ 1. Physical Layer (Cables, Signals) │
└─────────────────────────────────────┘
```

### TCP/IP Model (Practical)
```
┌─────────────────────────────────────┐
│ Application Layer (HTTP, FTP, SMTP) │
├─────────────────────────────────────┤
│ Transport Layer (TCP, UDP)          │
├─────────────────────────────────────┤
│ Internet Layer (IP, ICMP)           │
├─────────────────────────────────────┤
│ Link Layer (Ethernet, WiFi)         │
└─────────────────────────────────────┘
```

## Transport Layer Protocols

### TCP (Transmission Control Protocol)
```
Three-Way Handshake:
Client → Server: SYN
Server → Client: SYN-ACK
Client → Server: ACK

Connection Established
```

**Characteristics:**
- **Connection-oriented**: Establishes connection before data transfer
- **Reliable**: Guarantees delivery, order, and error checking
- **Flow Control**: Manages data transmission rate
- **Congestion Control**: Prevents network overload

**Use Cases:**
- Web browsing (HTTP/HTTPS)
- File transfer (FTP)
- Email (SMTP)
- Database connections

### UDP (User Datagram Protocol)
```
Client → Server: Datagram (no handshake)
Server → Client: Datagram (no acknowledgment)
```

**Characteristics:**
- **Connectionless**: No connection establishment
- **Unreliable**: No guarantee of delivery or order
- **Low overhead**: Minimal header size
- **Fast**: No handshaking or acknowledgment delays

**Use Cases:**
- Video streaming
- Online gaming
- DNS queries
- VoIP

## Application Layer Protocols

### HTTP/HTTPS
```
HTTP Request:
GET /api/users HTTP/1.1
Host: example.com
Authorization: Bearer token123
Content-Type: application/json

HTTPS Request:
Same as HTTP, but encrypted with TLS/SSL
```

**HTTP Methods:**
- **GET**: Retrieve resource
- **POST**: Create resource
- **PUT**: Update resource (replace)
- **PATCH**: Update resource (partial)
- **DELETE**: Remove resource

**HTTP Status Codes:**
- **2xx**: Success (200 OK, 201 Created)
- **3xx**: Redirection (301 Moved, 302 Found)
- **4xx**: Client Error (400 Bad Request, 404 Not Found)
- **5xx**: Server Error (500 Internal Error, 503 Unavailable)

### WebSocket
```
Handshake:
GET /socket HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: key123

Communication:
Client → Server: Message
Server → Client: Message
```

**Characteristics:**
- **Full-duplex**: Bidirectional communication
- **Persistent**: Connection stays open
- **Low latency**: No HTTP overhead after handshake
- **Real-time**: Instant message delivery

### gRPC (Remote Procedure Call)
```
Protocol Buffer Definition:
service UserService {
  rpc GetUser (UserRequest) returns (UserResponse);
  rpc CreateUser (CreateUserRequest) returns (UserResponse);
}

Generated Code:
- Client stubs
- Server interfaces
- Message serialization
```

**Features:**
- **Protocol Buffers**: Efficient binary serialization
- **Streaming**: Unary, server, client, and bidirectional streaming
- **HTTP/2**: Multiplexing, header compression, server push
- **Multi-language**: Code generation for multiple languages
- **Built-in**: Authentication, load balancing, monitoring

**Use Cases:**
- **Microservices Communication**: Efficient service-to-service calls
- **Mobile Apps**: Bandwidth-efficient API calls
- **Real-time Systems**: Streaming data pipelines

### GraphQL
```
Query:
{
  user(id: "123") {
    name
    email
    posts {
      title
      content
    }
  }
}

Response:
{
  "data": {
    "user": {
      "name": "John Doe",
      "email": "john@example.com",
      "posts": [...]
    }
  }
}
```

**Characteristics:**
- **Single Endpoint**: One URL for all operations
- **Client-Driven**: Clients specify exactly what data they need
- **Type System**: Strongly typed schema
- **Introspection**: Self-documenting API
- **Real-time**: Subscriptions for live updates

**Benefits:**
- **Over-fetching Prevention**: Get exactly what's needed
- **Under-fetching Prevention**: Single request for complex data
- **Versioning**: Schema evolution without breaking changes
- **Developer Experience**: Rich tooling and documentation

## Network Architecture Patterns

### Client-Server Architecture
```
Client ←→ Server ←→ Database
   ↑         ↑         ↑
Multiple   Single    Multiple
```

**Components:**
- **Client**: Initiates requests
- **Server**: Processes requests, returns responses
- **Database**: Stores and retrieves data

**Pros:**
- Simple to understand and implement
- Centralized management
- Clear separation of concerns

**Cons:**
- Single point of failure
- Scalability limitations
- Network bottleneck

### Peer-to-Peer Architecture
```
Peer 1 ←→ Peer 2 ←→ Peer 3
   ↑        ↑        ↑
   └────────┼────────┘
            ↓
        Distributed Network
```

**Characteristics:**
- **Decentralized**: No central server
- **Scalable**: Add peers to increase capacity
- **Resilient**: No single point of failure
- **Complex**: Coordination and consistency challenges

**Use Cases:**
- File sharing (BitTorrent)
- Blockchain networks
- Distributed computing

### Microservices Architecture
```
API Gateway
    ↓
┌─────────────────────────────────────┐
│ Service A │ Service B │ Service C   │
└─────────────────────────────────────┘
    ↓         ↓         ↓
Database A  Database B  Database C
```

**Communication Patterns:**
- **Synchronous**: HTTP/REST, gRPC
- **Asynchronous**: Message queues, event streaming
- **Service Discovery**: DNS, Consul, etcd

### Service Mesh Architecture
```
┌────────────────────────────────────────────────────────┐
│                    API Gateway                         │
│                                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   Service   │  │   Service   │  │   Service   │     │
│  │     A       │  │     B       │  │     C       │     │
│  │  ┌─────────┐│  │  ┌─────────┐│  │  ┌─────────┐│     │
│  │  │  Side   ││  │  │  Side   ││  │  │  Side   ││     │
│  │  │  Car    ││  │  │  Car    ││  │  │  Car    ││     │
│  │  └─────────┘│  │  └─────────┘│  │  └─────────┘│     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└────────────────────────────────────────────────────────┘
                        ↓
                ┌─────────────┐
                │ Control     │
                │ Plane       │
                │ (Istio)     │
                └─────────────┘
```

**Components:**
- **Data Plane**: Sidecars handle service-to-service communication
- **Control Plane**: Manages configuration, policies, and observability
- **Service Discovery**: Automatic service registration and discovery
- **Load Balancing**: Intelligent routing and load distribution

**Features:**
- **Traffic Management**: Routing rules, load balancing, circuit breaking
- **Security**: mTLS encryption, authentication, authorization
- **Observability**: Distributed tracing, metrics, logging
- **Resilience**: Retries, timeouts, fault injection

**Examples:**
- **Istio**: Comprehensive service mesh for Kubernetes
- **Linkerd**: Lightweight service mesh
- **Consul**: Service networking platform

### API Gateway Pattern
```
Internet
    ↓
┌─────────────────────────────────┐
│         API Gateway             │
│  ┌─────────────────────────┐    │
│  │ Rate Limiting           │    │
│  │ Authentication          │    │
│  │ Request Routing         │    │
│  │ Response Transformation │    │
│  │ Caching                 │    │
│  └─────────────────────────┘    │
└─────────────────────────────────┘
    ↓
┌──────────────────────────────────────┐
│   Microservices Cluster              │
│                                      │
│  Service A    Service B    Service C │
└──────────────────────────────────────┘
```

**Responsibilities:**
- **Request Routing**: Route requests to appropriate services
- **Authentication**: JWT validation, OAuth, API keys
- **Rate Limiting**: Prevent abuse and ensure fair usage
- **Load Balancing**: Distribute traffic across service instances
- **Caching**: Cache responses to reduce backend load
- **Transformation**: Modify requests/responses (format conversion)
- **Monitoring**: Collect metrics and logs

**Examples:**
- **Kong**: Open-source API gateway
- **AWS API Gateway**: Managed service
- **Netflix Zuul**: Custom gateway solution

## Load Balancing

### DNS Load Balancing
```
example.com → [IP1, IP2, IP3]
Client resolves to random IP
```

**Pros:**
- Simple to implement
- Geographic distribution
- No additional infrastructure

**Cons:**
- No health checking
- Uneven distribution
- DNS caching issues

### Application Load Balancer
```
Client → Load Balancer → [Server1, Server2, Server3]
```

**Algorithms:**
- **Round Robin**: Sequential distribution
- **Least Connections**: Route to least busy server
- **IP Hash**: Consistent routing based on client IP
- **Weighted**: Distribute based on server capacity

## Network Security

### SSL/TLS Handshake
```
1. Client Hello: Supported cipher suites
2. Server Hello: Selected cipher suite, certificate
3. Client Key Exchange: Pre-master secret (encrypted)
4. Server Finished: Verify handshake
5. Secure Communication: Encrypted data transfer
```

### Zero-Trust Networking
```
Traditional: Trust internal network
Zero-Trust: Never trust, always verify

Internet → Identity Verification → Access Control → Resource
    ↓             ↓                    ↓           ↓
  AuthN        AuthZ               Policies    Services
```

**Principles:**
- **Verify Identity**: Authenticate all users and devices
- **Least Privilege**: Grant minimum required access
- **Micro-Segmentation**: Divide network into small segments
- **Continuous Monitoring**: Monitor all traffic and behavior
- **Assume Breach**: Design for compromised credentials

**Implementation:**
- **Identity Providers**: OAuth, SAML, OpenID Connect
- **Policy Engines**: Attribute-based access control (ABAC)
- **Service Mesh**: mTLS between services
- **Network Segmentation**: VPCs, security groups, network ACLs

### Firewall Patterns
```
Internet → Firewall → Internal Network
            ↓
         Rules:
         - Allow HTTP (80)
         - Allow HTTPS (443)
         - Block all other ports
```

**Types:**
- **Packet Filtering**: Filter based on IP/port
- **Stateful Inspection**: Track connection state
- **Application Layer**: Filter based on content
- **Next-Generation**: Advanced threat detection

### Cloud Network Security
```
┌────────────────────────────────────────────────────────┐
│                    Internet                            │
│                                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   WAF       │  │   Load      │  │   Security  │     │
│  │             │  │   Balancer  │  │   Groups    │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│                                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   VPC       │  │   Subnet    │  │   NACL      │     │
│  │             │  │             │  │             │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│                                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   Service   │  │   Service   │  │   Service   │     │
│  │     A       │  │     B       │  │     C       │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└────────────────────────────────────────────────────────┘
```

**Components:**
- **WAF (Web Application Firewall)**: Protect against web attacks
- **Security Groups**: Instance-level firewall rules
- **Network ACLs**: Subnet-level traffic control
- **VPC**: Isolated virtual network
- **VPN/Direct Connect**: Secure hybrid connections

## Performance Optimization

### Connection Pooling
```
Application
    ↓
Connection Pool
    ↓
[Conn1][Conn2][Conn3] → Database
```

**Benefits:**
- Reduced connection overhead
- Better resource utilization
- Improved response time

### HTTP/2 Multiplexing
```
Single TCP Connection
├── Stream 1: GET /style.css
├── Stream 2: GET /script.js
└── Stream 3: GET /image.jpg
```

**Features:**
- **Multiplexing**: Multiple requests over single connection
- **Header Compression**: HPACK algorithm
- **Server Push**: Proactively send resources
- **Binary Protocol**: More efficient than text

### CDN (Content Delivery Network)
```
User → CDN Edge → Origin Server (if cache miss)
       ↓
   Cached Content (if cache hit)
```

**Benefits:**
- Reduced latency
- Lower bandwidth costs
- Improved availability
- DDoS mitigation

## Network Reliability Patterns

### Circuit Breaker Pattern
```
Service A → Circuit Breaker → Service B
    ↓             ↓             ↓
 Normal      Open State      Fallback
   ↓            ↓             ↓
 Success     Fast Fail       Cached/Default
```

**States:**
- **Closed**: Normal operation, requests pass through
- **Open**: Failure threshold exceeded, requests fail fast
- **Half-Open**: Testing if service recovered

**Benefits:**
- **Prevent Cascading Failures**: Stop failing service calls
- **Fast Failure**: Immediate response instead of timeout
- **Automatic Recovery**: Self-healing when service recovers

### Retry and Timeout Patterns
```python
def call_with_retry(service_call, max_retries=3, timeout=5):
    for attempt in range(max_retries):
        try:
            return service_call(timeout=timeout)
        except TimeoutError:
            if attempt == max_retries - 1:
                raise
            # Exponential backoff
            time.sleep(2 ** attempt)
```

**Strategies:**
- **Fixed Retry**: Same delay between retries
- **Exponential Backoff**: Increasing delay (1s, 2s, 4s...)
- **Jitter**: Random delay to prevent thundering herd
- **Circuit Breaker Integration**: Don't retry when circuit is open

### Bulkhead Pattern
```
Service Pool
├── Bulkhead 1 (User Service) → DB Connection Pool 1
├── Bulkhead 2 (Order Service) → DB Connection Pool 2
└── Bulkhead 3 (Payment Service) → DB Connection Pool 3
```

**Benefits:**
- **Isolate Failures**: One service failure doesn't affect others
- **Resource Protection**: Prevent resource exhaustion
- **Priority Management**: Allocate resources by importance

## Distributed Tracing

### Trace Context Propagation
```
Request: GET /api/users/123
Headers:
  X-Trace-Id: abc123
  X-Span-Id: span456
  X-Parent-Span-Id: span123

Service A (span456) → Service B (span789) → Service C (span101)
```

**Components:**
- **Trace ID**: Unique identifier for entire request flow
- **Span ID**: Unique identifier for each operation
- **Parent Span ID**: Links spans in the call hierarchy
- **Baggage**: Additional context data

### Tracing Standards
- **OpenTracing**: Vendor-neutral API
- **OpenCensus**: Google's tracing framework
- **Jaeger**: Distributed tracing system
- **Zipkin**: Distributed tracing system

## Network Monitoring

### Key Metrics
- **Bandwidth**: Data transfer rate
- **Latency**: Round-trip time (p50, p95, p99)
- **Packet Loss**: Percentage of lost packets
- **Jitter**: Variation in latency
- **Throughput**: Actual data transfer rate
- **Error Rate**: Percentage of failed requests
- **Connection Pool Usage**: Active vs idle connections

### Observability Stack
```
┌────────────────────────────────────────────────────────┐
│                    Application                         │
│                                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   Metrics   │  │   Logs      │  │   Traces    │     │
│  │ (Prometheus)│  │ (ELK)       │  │ (Jaeger)    │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│                                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   Alerts    │  │ Dashboards  │  │   Analysis  │     │
│  │ Alertmanager│  │ (Grafana)   │  │             │     │
│  │             │  │             │  │             │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└────────────────────────────────────────────────────────┘
```

### Monitoring Tools
```bash
# Network latency
ping google.com

# Bandwidth usage
iftop -i eth0

# Connection monitoring
netstat -an | grep ESTABLISHED

# Packet capture
tcpdump -i eth0 port 80

# Container networking
docker network ls
kubectl get pods -o wide
```

### Service Mesh Observability
```
Service Mesh Control Plane
    ↓
┌────────────────────────────────────────────────────────┐
│                    Data Plane                          │
│                                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   Metrics   │  │   Tracing   │  │   Logging   │     │
│  │ Collection  │  │ Collection  │  │ Collection  │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│                                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ Prometheus  │  │ Jaeger      │  │ Fluentd     │     │
│  │             │  │             │  │             │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└────────────────────────────────────────────────────────┘
```

## Real-World Examples

### Netflix
- **Architecture**: Global CDN + Edge computing
- **Protocols**: HTTP/2, TLS 1.3
- **Optimization**: Adaptive streaming, prefetching

### Google
- **Architecture**: Global anycast network
- **Protocols**: QUIC (HTTP/3), gRPC
- **Optimization**: Network load balancing, edge caching

### Amazon
- **Architecture**: Multi-region infrastructure
- **Protocols**: HTTP/2, WebSockets
- **Optimization**: Route 53, CloudFront CDN

## Interview Tips

### Common Questions
1. "What's the difference between TCP and UDP?"
2. "How does HTTPS work?"
3. "How would you design a load balancer?"
4. "What's the difference between HTTP/1.1 and HTTP/2?"
5. "How does a service mesh work?"
6. "When would you use gRPC vs REST?"
7. "How do you implement zero-trust networking?"
8. "What's the circuit breaker pattern?"

### Answer Framework
1. **Understand Requirements**: Performance, security, reliability, scale
2. **Choose Protocols**: TCP/UDP, HTTP/WebSocket/gRPC/GraphQL
3. **Design Architecture**: Client-server, microservices, service mesh
4. **Implement Reliability**: Circuit breakers, retries, bulkheads
5. **Ensure Security**: Zero-trust, encryption, access control
6. **Add Observability**: Monitoring, tracing, logging
7. **Optimize Performance**: Caching, connection pooling, CDN

### Key Points to Emphasize
- Trade-offs between performance and reliability
- Security considerations at each layer
- Scalability implications of design choices
- Cost optimization strategies
- Monitoring and troubleshooting
- Modern cloud-native patterns (service mesh, API gateway)
- Distributed systems challenges

## Practice Problems

1. **Design a real-time gaming communication system**
2. **Implement a video streaming protocol**
3. **Design a global API gateway with service mesh**
4. **Create a secure file transfer system with zero-trust**
5. **Design a network monitoring solution for microservices**
6. **Implement distributed tracing for a complex system**
7. **Design a resilient service-to-service communication layer**
8. **Create a GraphQL API for a social media platform**

## Further Reading

- **Books**:
  - "Computer Networking" by Kurose and Ross
  - "Designing Data-Intensive Applications" by Martin Kleppmann
  - "Site Reliability Engineering" by Google

- **RFCs**:
  - RFC 2616 (HTTP/1.1), RFC 7540 (HTTP/2)
  - RFC 8446 (TLS 1.3), RFC 9114 (HTTP/3)

- **Documentation**:
  - TCP/IP Illustrated, Network Programming with Sockets
  - Istio Service Mesh Documentation
  - gRPC Documentation

- **Online Resources**:
  - "Service Mesh Patterns" by Buoyant
  - "Envoy Proxy Documentation"
  - "Kubernetes Networking Guide"
  - "AWS Networking Best Practices"

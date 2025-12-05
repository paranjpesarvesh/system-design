# Load Balancers

## Definition

A load balancer is a server that acts as a traffic cop, sitting in front of your servers and routing client requests across all servers capable of fulfilling those requests in a manner that maximizes speed and capacity utilization.

## Load Balancing Architecture

```
Internet
    ↓
[Load Balancer]
    ↓
┌───────────────────────────────────────┐
│  Server 1  │  Server 2  │  Server 3   │
└───────────────────────────────────────┘
```

## Types of Load Balancers

### 1. Layer 4 (Transport Layer)
- **Protocol**: TCP/UDP
- **Decision Making**: Based on IP address and port
- **Speed**: Fast, low latency
- **Use Case**: General purpose load balancing

```
Client IP:Port → LB → Server IP:Port
```

### 2. Layer 7 (Application Layer)
- **Protocol**: HTTP/HTTPS
- **Decision Making**: Based on content, headers, URL
- **Speed**: Slower but more intelligent
- **Use Case**: Content-based routing, SSL termination

```
HTTP Request → LB → Parse URL/Headers → Route to appropriate server
```

### 3. Global Server Load Balancing (GSLB)
- **Scope**: Geographic distribution
- **Decision Making**: Based on location, health, latency
- **Use Case**: Multi-region deployments, CDN

## Load Balancing Algorithms

### 1. Round Robin
```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A
```

- **Pros**: Simple, evenly distributed
- **Cons**: Ignores server capacity
- **Use Case**: Servers with similar capabilities

### 2. Weighted Round Robin
```
Server A (Weight 3): A-A-A-
Server B (Weight 2): -B-B--
Server C (Weight 1): ---C--
Sequence: A-A-A-B-B-C
```

- **Pros**: Considers server capacity
- **Cons**: Static weights
- **Use Case**: Heterogeneous server environments

### 3. Least Connections
```
Server A: 2 connections
Server B: 5 connections
Server C: 1 connection

New Request → Server C (fewest connections)
```

- **Pros**: Dynamic load distribution
- **Cons**: Requires connection tracking
- **Use Case**: Long-lived connections

### 4. IP Hash
```
Client IP: 192.168.1.1 → Hash → Server B
Client IP: 192.168.1.2 → Hash → Server A
Client IP: 192.168.1.1 → Hash → Server B (consistent)
```

- **Pros**: Session persistence
- **Cons**: Uneven distribution
- **Use Case**: Stateful applications, sticky sessions

### 5. Least Response Time
```
Server A: Response time 50ms
Server B: Response time 100ms
Server C: Response time 75ms

New Request → Server A (fastest response)
```

- **Pros**: Performance-based routing
- **Cons**: Requires active monitoring
- **Use Case**: Performance-sensitive applications

## Health Checking

### Health Check Methods
```
Load Balancer → Health Check → Server Response
     ↓              ↓              ↓
  Every 30s     TCP/HTTP       200 OK
```

### Types of Health Checks
1. **TCP Check**: Connection establishment
2. **HTTP Check**: HTTP response with specific status code
3. **SSL Check**: SSL handshake verification
4. **Custom Check**: Application-specific health endpoint

### Health Check Configuration
- **Interval**: How often to check (e.g., 30 seconds)
- **Timeout**: How long to wait for response (e.g., 5 seconds)
- **Unhealthy Threshold**: Consecutive failures before marking unhealthy
- **Healthy Threshold**: Consecutive successes before marking healthy

## Session Persistence (Sticky Sessions)

### Implementation Methods
1. **Source IP Affinity**: Route based on client IP
2. **Cookie-based**: Load balancer sets/reads cookie
3. **SSL Session ID**: Use SSL session identifier

### Cookie-based Example
```
First Request:
Client → LB: No cookie
LB → Server A: Process request
LB → Client: Set-Cookie: SERVER=A

Subsequent Requests:
Client → LB: Cookie: SERVER=A
LB → Server A: Route to same server
```

## SSL/TLS Termination

### Architecture
```
Client ←→ HTTPS ←→ Load Balancer ←→ HTTP ←→ Servers
```

### Benefits
- **Offloading**: Reduces server CPU load
- **Centralized Management**: Single place for certificates
- **Caching**: Can cache decrypted content

### Drawbacks
- **Security**: Load balancer sees unencrypted traffic
- **Compliance**: May violate security requirements

## High Availability for Load Balancers

### Active-Passive Setup
```
          ┌─ Primary LB ─┐
Internet ─┤              ├── Servers
          └─ Backup LB ──┘
```

### Active-Active Setup
```
          ┌─ LB 1 ─┐
Internet ─┤        ├── Servers
          └─ LB 2 ─┘
```

### Failover Process
1. **Health Monitoring**: Continuous health checks
2. **Failure Detection**: Identify primary failure
3. **IP Failover**: Move virtual IP to backup
4. **Session Sync**: Transfer active sessions

## Real-World Implementations

### 1. Hardware Load Balancers
- **Examples**: F5 BIG-IP, Citrix ADC
- **Pros**: High performance, feature-rich
- **Cons**: Expensive, vendor lock-in

### 2. Software Load Balancers
- **Examples**: Nginx, HAProxy, Envoy
- **Pros**: Flexible, cost-effective, cloud-native
- **Cons**: Requires management and scaling

### 3. Cloud Load Balancers
- **AWS**: Application Load Balancer (ALB), Network Load Balancer (NLB)
- **Google Cloud**: Cloud Load Balancing
- **Azure**: Azure Load Balancer, Application Gateway

## Load Balancer Configuration Example

### Nginx Configuration
```nginx
upstream backend {
    least_conn;
    server server1.example.com weight=3;
    server server2.example.com weight=2;
    server server3.example.com weight=1;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /health {
        access_log off;
        return 200 "healthy\n";
    }
}
```

### HAProxy Configuration
```
global
    daemon
    maxconn 4096

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend web_frontend
    bind *:80
    default_backend web_servers

backend web_servers
    balance roundrobin
    option httpchk GET /health
    server web1 server1.example.com:80 check
    server web2 server2.example.com:80 check
    server web3 server3.example.com:80 check
```

## Performance Considerations

### Throughput
- **Connections per Second**: CPS rating
- **Bandwidth**: Gbps capacity
- **Concurrent Connections**: Maximum simultaneous connections

### Latency
- **Processing Time**: Time to make routing decision
- **Network Overhead**: Additional hop in request path
- **SSL Handshake**: Time for encryption setup

### Resource Usage
- **CPU**: SSL termination, compression
- **Memory**: Connection tracking, caching
- **Network**: Bandwidth, packet processing

## Monitoring and Metrics

### Key Metrics
- **Request Rate**: RPS per backend server
- **Response Time**: Latency measurements
- **Error Rate**: 5xx responses, timeouts
- **Connection Count**: Active and total connections
- **Health Status**: Backend server availability

### Monitoring Tools
- **Built-in**: Load balancer statistics
- **APM**: Application performance monitoring
- **Logging**: Access logs, error logs
- **Metrics**: Prometheus, Grafana

## Interview Tips

### Common Questions
1. "What's the difference between L4 and L7 load balancers?"
2. "How would you handle session persistence?"
3. "What algorithms would you use for different scenarios?"
4. "How do you ensure high availability for load balancers?"

### Answer Framework
1. **Understand Requirements**: Traffic patterns, session needs
2. **Choose Type**: L4 vs L7 based on requirements
3. **Select Algorithm**: Based on server capabilities and traffic
4. **Design for HA**: Redundancy, health checking, failover
5. **Monitor and Optimize**: Performance metrics and tuning

### Key Points to Emphasize
- Trade-offs between performance and features
- Importance of health checking
- Session persistence requirements
- SSL termination considerations
- High availability design

## Practice Problems

1. **Design a load balancer for a video streaming service**
2. **Handle flash sale traffic with auto-scaling**
3. **Implement geographic load balancing for global app**
4. **Design load balancer for microservices architecture**
5. **Create load balancing strategy for IoT platform**

## Further Reading

- **RFCs**: RFC 3261 (SIP Load Balancing), RFC 6455 (WebSocket)
- **Documentation**: Nginx Load Balancing, HAProxy Configuration
- **Books**: "Load Balancing Servers" by Tony Bourke

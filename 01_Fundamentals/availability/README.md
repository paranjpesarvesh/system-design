# Availability and Reliability

## Definitions

### Availability
Availability is the probability that a system is operational and accessible when needed. It's typically expressed as a percentage of uptime.

### Reliability
Reliability is the probability that a system will perform its intended function without failure over a specified time period.

## Availability Metrics

### Availability Percentages
| Availability | Downtime per Year | Downtime per Month | Downtime per Week |
|--------------|-------------------|--------------------|-------------------|
| 99%          | 3.65 days         | 7.2 hours          | 1.68 hours        |
| 99.9%        | 8.76 hours        | 43.2 minutes       | 10.1 minutes      |
| 99.99%       | 52.56 minutes     | 4.32 minutes       | 1.01 minutes      |
| 99.999%      | 5.26 minutes      | 25.9 seconds       | 6.05 seconds      |

### Key Metrics
- **MTBF (Mean Time Between Failures)**: Average time between system failures
- **MTTR (Mean Time To Repair)**: Average time to recover from failure
- **MTTF (Mean Time To Failure)**: Average time until a component fails

## High Availability Patterns

### 1. Redundancy
```
Primary Server
    ↓
[Replica 1, Replica 2, Replica 3]
```

- **Active-Active**: All replicas handle traffic
- **Active-Passive**: One active, others on standby
- **Multi-Active**: Multiple active across regions

### 2. Failover Mechanisms
```
Health Check → Failover Controller → Traffic Redirect
```

- **Automatic Failover**: System detects failure and switches automatically
- **Manual Failover**: Human intervention required
- **Graceful Degradation**: Reduced functionality instead of complete failure

### 3. Load Balancing with Health Checks
```
Client → Load Balancer → [Healthy Servers Only]
```

- **Health Checks**: Regular monitoring of server health
- **Traffic Routing**: Only route to healthy instances
- **Circuit Breaker**: Stop sending traffic to failing services

## Reliability Engineering

### 1. Fault Tolerance
- **Definition**: System continues operating despite component failures
- **Techniques**: Redundancy, error correction, graceful degradation

### 2. Fault Isolation
- **Bulkhead Pattern**: Isolate failures to prevent cascade
- **Circuit Breakers**: Stop calls to failing services
- **Timeouts**: Prevent waiting indefinitely

### 3. Recovery Mechanisms
- **Automatic Recovery**: Self-healing systems
- **Checkpoint/Restore**: Save state for recovery
- **Retry Logic**: Handle transient failures

## CAP Theorem

### The Trade-off
```
Consistency (C) + Availability (A) + Partition Tolerance (P)
You can only choose 2 out of 3
```

### CP Systems (Consistency + Partition Tolerance)
- **Characteristics**: Prioritize data consistency over availability
- **Use Cases**: Financial systems, databases
- **Examples**: HBase, MongoDB (with strong consistency)

### AP Systems (Availability + Partition Tolerance)
- **Characteristics**: Prioritize availability over consistency
- **Use Cases**: Social media, caching systems
- **Examples**: Cassandra, DynamoDB, DNS

### CA Systems (Consistency + Availability)
- **Characteristics**: No network partitions assumed
- **Use Cases**: Single datacenter systems
- **Examples**: Traditional RDBMS

## Disaster Recovery

### Recovery Time Objective (RTO)
- **Definition**: Maximum acceptable downtime
- **Examples**: 
  - Critical systems: < 1 hour
  - Important systems: < 4 hours
  - Non-critical: < 24 hours

### Recovery Point Objective (RPO)
- **Definition**: Maximum acceptable data loss
- **Examples**:
  - Real-time systems: < 1 minute
  - Business systems: < 1 hour
  - Analytics: < 24 hours

### Disaster Recovery Strategies

#### 1. Backup and Restore
- **Cold Site**: Empty facility with power/cooling
- **Warm Site**: Equipped facility with some systems
- **Hot Site**: Fully operational duplicate site

#### 2. Geographic Distribution
```
Primary Region (US-East)
    ↓ Replication
Secondary Region (US-West)
    ↓ Replication
Tertiary Region (EU)
```

#### 3. Multi-Region Active-Active
- **Benefits**: Zero downtime, global performance
- **Challenges**: Data consistency, complexity, cost

## Monitoring and Alerting

### Key Metrics
- **Uptime**: Percentage of time system is available
- **Error Rate**: Percentage of failed requests
- **Response Time**: Latency measurements (p50, p95, p99)
- **Throughput**: Requests per second

### Alerting Strategy
```
Metric Threshold → Alert → Escalation → Resolution
```

- **Warning**: Degraded performance
- **Critical**: Service impact
- **Emergency**: System outage

### SLA vs SLO vs SLI

#### SLI (Service Level Indicator)
- **Definition**: Measurable metric of service performance
- **Examples**: Request latency, error rate, availability

#### SLO (Service Level Objective)
- **Definition**: Target value for SLI
- **Examples**: 99.9% availability, p99 latency < 500ms

#### SLA (Service Level Agreement)
- **Definition**: Contractual commitment to customers
- **Examples**: 99.9% uptime guarantee with credits for violations

### Error Budgets
```
Error Budget = 100% - SLO Target
Example: 99.9% SLO = 0.1% Error Budget
```

**Benefits:**
- **Innovation Freedom**: Spend budget on improvements
- **Risk Management**: Balance reliability vs features
- **Data-Driven Decisions**: Quantify reliability investments

**Implementation:**
- **Calculate Budget**: Total allowed downtime/errors
- **Track Usage**: Monitor error budget consumption
- **Alert Thresholds**: Warnings at 50%, 80%, 100% usage
- **Budget Burn Rate**: Rate of error budget consumption

### Site Reliability Engineering (SRE)

#### Core Principles
```
Reliability = (Successful Requests) / (Total Requests)
Availability = Uptime / Total Time
```

**Key Practices:**
- **Service Level Management**: Define and monitor SLIs/SLOs/SLAs
- **Error Budgeting**: Balance innovation with reliability
- **Blameless Postmortems**: Learn from incidents without blame
- **Toil Reduction**: Automate manual tasks
- **Graduated Alerting**: Different response levels for incidents

#### SRE Tools and Practices
```
Monitoring Stack:
├── Metrics Collection (Prometheus)
├── Log Aggregation (ELK Stack)
├── Tracing (Jaeger/Zipkin)
└── Alerting (PagerDuty/OpsGenie)

Incident Response:
├── Alert → Triage → Investigation
├── Mitigation → Resolution → Retrospective
└── Prevention → Documentation → Training
```

### Chaos Engineering
```
Chaos Engineering Process:
1. Define Steady State
2. Form Hypotheses
3. Design Experiments
4. Run in Production
5. Analyze Results
6. Improve System
```

**Principles:**
- **Build Confidence**: Test system resilience in production
- **Minimize Blast Radius**: Start small, expand gradually
- **Run Continuously**: Regular chaos experiments
- **Automate**: Make chaos part of CI/CD pipeline

**Common Experiments:**
- **Instance Termination**: Random server shutdowns
- **Network Latency**: Simulate network delays
- **Service Degradation**: Reduce service capacity
- **Dependency Failures**: Break external service calls

### Incident Management

#### Incident Response Process
```
Detection → Triage → Investigation → Mitigation → Resolution → Retrospective
```

**Roles:**
- **Incident Commander**: Overall coordination
- **Communications Lead**: Internal/external updates
- **Operations Lead**: Technical resolution
- **Subject Matter Experts**: Domain-specific help

#### Blameless Postmortems
```
Postmortem Structure:
├── Timeline of Events
├── Impact Assessment
├── Root Cause Analysis
├── Lessons Learned
├── Action Items
└── Prevention Measures
```

**Best Practices:**
- **Focus on Systems**: Not individual blame
- **Encourage Participation**: Include all involved parties
- **Actionable Outcomes**: Specific, measurable improvements
- **Follow-up Tracking**: Ensure action items are completed

### Reliability Testing

#### Types of Tests
```
Load Testing:     Normal expected load
Stress Testing:   Beyond normal capacity
Spike Testing:    Sudden traffic increases
Volume Testing:   Large data volumes
Endurance Testing: Sustained load over time
Chaos Testing:    Random failures and disruptions
```

**Testing Strategy:**
- **Shift Left**: Test reliability early in development
- **Automated Testing**: Include in CI/CD pipelines
- **Production Testing**: Safe chaos experiments
- **Game Days**: Simulated disaster scenarios

## Real-World Examples

### Google Search
- **Availability**: 99.99%+
- **Strategy**: Multi-region, automatic failover, redundancy
- **Architecture**: Distributed across datacenters globally

### Amazon S3
- **Availability**: 99.99% (Standard), 99.9% (Infrequent Access)
- **Strategy**: Multi-AZ replication, automatic failover
- **Durability**: 99.999999999% (11 9's)

### GitHub
- **Availability**: 99.95%+
- **Strategy**: Active-active across regions, database clustering
- **Challenges**: High write loads, global collaboration

## Interview Tips

### Common Questions
1. "How would you design a highly available system?"
2. "What's the difference between availability and reliability?"
3. "How do you handle database failures?"
4. "Explain CAP theorem with examples."
5. "What's an error budget and how do you use it?"
6. "How do you conduct a blameless postmortem?"
7. "What's the difference between SLA, SLO, and SLI?"

### Answer Framework
1. **Define Requirements**: Availability targets, RTO/RPO, error budgets
2. **Identify Failure Points**: Single points of failure, dependencies
3. **Design Redundancy**: Multi-region, multi-AZ, active-active/passive
4. **Implement Reliability Patterns**: Circuit breakers, retries, bulkheads
5. **Plan Incident Response**: Detection, triage, mitigation, postmortems
6. **Monitor and Alert**: SLIs, SLOs, error budgets, chaos engineering
7. **Test Regularly**: Load testing, chaos experiments, game days

### Key Points to Emphasize
- Start with understanding availability requirements and business impact
- Design for failure, not success (assume components will fail)
- Implement comprehensive monitoring and observability
- Use SRE practices: error budgets, blameless postmortems, toil reduction
- Test failure scenarios regularly with chaos engineering
- Balance reliability with innovation through error budgets
- Consider cost implications of high availability

## Practice Problems

1. **Design a 99.99% available API gateway with multi-region failover**
2. **Create a disaster recovery plan for e-commerce platform with RTO/RPO**
3. **Design a highly available database system with active-active replication**
4. **Implement circuit breaker and bulkhead patterns for microservices**
5. **Design monitoring system for distributed application with SLOs**
6. **Conduct chaos engineering experiments for a critical service**
7. **Design incident response process with blameless postmortems**
8. **Implement error budget tracking and alerting system**

## Further Reading

- **Books**: "Site Reliability Engineering" by Google, "Release It!" by Michael Nygard
- **Concepts**: Chaos Engineering, Observability, Incident Response
- **Tools**: Prometheus, Grafana, PagerDuty, Chaos Monkey
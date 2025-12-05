# Monitoring and Observability

## Definition

Monitoring and observability are practices that enable teams to understand system behavior, detect issues, and troubleshoot problems in complex distributed systems through metrics, logs, and traces.

## Three Pillars of Observability

### 1. Metrics (What)
- **Definition**: Numerical measurements over time
- **Examples**: CPU usage, request rate, error rate
- **Characteristics**: Aggregated, pre-computed, efficient storage

### 2. Logs (Why)
- **Definition**: Timestamped records of discrete events
- **Examples**: Error messages, debug information, audit trails
- **Characteristics**: Detailed, event-driven, high cardinality

### 3. Traces (Where)
- **Definition**: Request flow through distributed systems
- **Examples**: Service-to-service calls, database queries
- **Characteristics**: Causal relationships, performance bottlenecks

## Service Level Management

### Service Level Indicators (SLIs)
```
SLI Examples:
├── Availability: uptime / total_time
├── Latency: request_duration < target_threshold
├── Throughput: requests_per_second
├── Error Rate: failed_requests / total_requests
├── Data Freshness: age_of_data < max_age
└── Correctness: valid_responses / total_responses
```

### Service Level Objectives (SLOs)
```
SLO Definition:
├── Target: 99.9% of requests < 500ms latency
├── Window: Rolling 28-day window
├── Budget: 0.1% error budget remaining
└── Burn Rate: Current consumption rate
```

**SLO Implementation:**
```python
class SLOTracker:
    def __init__(self, slo_target=0.999, window_days=28):
        self.slo_target = slo_target
        self.window_seconds = window_days * 24 * 60 * 60
        self.measurements = []
        self.error_budget_used = 0.0

    def record_measurement(self, success: bool, timestamp: float):
        """Record a measurement (success/failure)"""
        self.measurements.append({
            'success': success,
            'timestamp': timestamp
        })

        # Clean old measurements
        cutoff = timestamp - self.window_seconds
        self.measurements = [
            m for m in self.measurements
            if m['timestamp'] > cutoff
        ]

    def calculate_sli(self) -> float:
        """Calculate current SLI"""
        if not self.measurements:
            return 1.0

        successful = sum(1 for m in self.measurements if m['success'])
        total = len(self.measurements)

        return successful / total

    def calculate_error_budget(self) -> float:
        """Calculate remaining error budget"""
        current_sli = self.calculate_sli()
        return max(0.0, self.slo_target - (1.0 - current_sli))

    def get_burn_rate(self) -> float:
        """Calculate current error budget burn rate"""
        # Simplified: actual implementation would use more sophisticated calculation
        if not self.measurements:
            return 0.0

        recent_failures = sum(1 for m in self.measurements[-100:] if not m['success'])
        return recent_failures / 100.0
```

### Service Level Agreements (SLAs)
```
SLA Components:
├── Service Description: What service provides
├── Performance Metrics: SLI specifications
├── Availability Guarantees: Uptime commitments
├── Support Levels: Response times for incidents
├── Remedies: Credits/penalties for violations
└── Exclusions: Force majeure, planned maintenance
```

## Anomaly Detection and AI Monitoring

### Statistical Anomaly Detection
```python
import numpy as np
from scipy import stats
from collections import deque

class AnomalyDetector:
    def __init__(self, window_size=100, threshold_sigma=3):
        self.window_size = window_size
        self.threshold_sigma = threshold_sigma
        self.measurements = deque(maxlen=window_size)

    def add_measurement(self, value: float):
        """Add a new measurement"""
        self.measurements.append(value)

    def is_anomalous(self, value: float) -> bool:
        """Check if value is anomalous"""
        if len(self.measurements) < 10:  # Need minimum data
            return False

        # Calculate z-score
        mean = np.mean(self.measurements)
        std = np.std(self.measurements)

        if std == 0:
            return False

        z_score = abs(value - mean) / std
        return z_score > self.threshold_sigma

    def detect_seasonal_anomalies(self, values: list, period: int) -> list:
        """Detect anomalies in seasonal data"""
        anomalies = []

        for i in range(len(values)):
            # Use seasonal decomposition
            if i >= period:
                seasonal_avg = np.mean(values[i-period:i])
                seasonal_std = np.std(values[i-period:i])

                if seasonal_std > 0:
                    z_score = abs(values[i] - seasonal_avg) / seasonal_std
                    if z_score > self.threshold_sigma:
                        anomalies.append(i)

        return anomalies
```

### Machine Learning Anomaly Detection
```python
from sklearn.ensemble import IsolationForest
from sklearn.preprocessing import StandardScaler
import pandas as pd

class MLAnomalyDetector:
    def __init__(self, contamination=0.1):
        self.contamination = contamination
        self.model = None
        self.scaler = StandardScaler()
        self.is_trained = False

    def train(self, historical_data: pd.DataFrame):
        """Train the anomaly detection model"""
        # Scale features
        scaled_data = self.scaler.fit_transform(historical_data)

        # Train Isolation Forest
        self.model = IsolationForest(
            contamination=self.contamination,
            random_state=42
        )
        self.model.fit(scaled_data)
        self.is_trained = True

    def predict(self, new_data: pd.DataFrame) -> list:
        """Predict anomalies in new data"""
        if not self.is_trained:
            raise ValueError("Model not trained")

        # Scale new data
        scaled_data = self.scaler.transform(new_data)

        # Predict anomalies (-1 for anomaly, 1 for normal)
        predictions = self.model.predict(scaled_data)

        # Convert to boolean list
        return [pred == -1 for pred in predictions]

    def score_anomalies(self, data: pd.DataFrame) -> list:
        """Get anomaly scores"""
        if not self.is_trained:
            raise ValueError("Model not trained")

        scaled_data = self.scaler.transform(data)
        scores = self.model.decision_function(scaled_data)

        return scores.tolist()
```

## Cloud-Native Monitoring

### AWS CloudWatch Integration
```python
import boto3
from datetime import datetime, timedelta

class CloudWatchMonitor:
    def __init__(self, region='us-east-1'):
        self.client = boto3.client('cloudwatch', region_name=region)

    def create_alarm(self, alarm_name, metric_name, namespace,
                    threshold, comparison_operator='GreaterThanThreshold'):
        """Create a CloudWatch alarm"""
        response = self.client.put_metric_alarm(
            AlarmName=alarm_name,
            AlarmDescription=f'Alarm for {metric_name}',
            MetricName=metric_name,
            Namespace=namespace,
            Statistic='Average',
            Period=300,  # 5 minutes
            Threshold=threshold,
            ComparisonOperator=comparison_operator,
            EvaluationPeriods=2,
            AlarmActions=[
                'arn:aws:sns:us-east-1:123456789012:alert-topic'
            ]
        )
        return response

    def get_metric_statistics(self, metric_name, namespace,
                            start_time, end_time, period=300):
        """Get metric statistics"""
        response = self.client.get_metric_statistics(
            Namespace=namespace,
            MetricName=metric_name,
            StartTime=start_time,
            EndTime=end_time,
            Period=period,
            Statistics=['Average', 'Maximum', 'Minimum']
        )
        return response['Datapoints']

    def create_dashboard(self, dashboard_name, dashboard_body):
        """Create a CloudWatch dashboard"""
        response = self.client.put_dashboard(
            DashboardName=dashboard_name,
            DashboardBody=dashboard_body
        )
        return response
```

### Kubernetes Monitoring
```yaml
# Prometheus ServiceMonitor for Kubernetes
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-service-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
---
# Grafana Dashboard ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard
  namespace: monitoring
data:
  dashboard.json: |
    {
      "dashboard": {
        "title": "Kubernetes Application Dashboard",
        "panels": [
          {
            "title": "CPU Usage",
            "type": "graph",
            "targets": [
              {
                "expr": "rate(container_cpu_usage_seconds_total{pod=~\"my-app-.*\"}[5m])",
                "legendFormat": "{{pod}}"
              }
            ]
          }
        ]
      }
    }
```

## User Experience Monitoring (RUM)

### Real User Monitoring Implementation
```javascript
// Frontend RUM Script
class RUMMonitor {
    constructor(appId, endpoint) {
        this.appId = appId;
        this.endpoint = endpoint;
        this.init();
    }

    init() {
        // Monitor page load
        window.addEventListener('load', () => {
            this.trackPageLoad();
        });

        // Monitor errors
        window.addEventListener('error', (event) => {
            this.trackError(event.error, event.filename, event.lineno);
        });

        // Monitor XHR/fetch requests
        this.interceptNetworkRequests();

        // Monitor user interactions
        this.trackUserInteractions();
    }

    trackPageLoad() {
        const navigation = performance.getEntriesByType('navigation')[0];
        const loadTime = navigation.loadEventEnd - navigation.fetchStart;

        this.sendMetric('page_load', {
            url: window.location.href,
            loadTime: loadTime,
            domContentLoaded: navigation.domContentLoadedEventEnd - navigation.fetchStart,
            firstPaint: this.getFirstPaint(),
            largestContentfulPaint: this.getLargestContentfulPaint()
        });
    }

    trackError(error, filename, lineno) {
        this.sendMetric('javascript_error', {
            message: error.message,
            filename: filename,
            lineno: lineno,
            stack: error.stack,
            userAgent: navigator.userAgent
        });
    }

    interceptNetworkRequests() {
        const originalFetch = window.fetch;
        window.fetch = async (...args) => {
            const startTime = Date.now();
            try {
                const response = await originalFetch(...args);
                const duration = Date.now() - startTime;

                this.sendMetric('network_request', {
                    url: args[0],
                    method: args[1]?.method || 'GET',
                    status: response.status,
                    duration: duration,
                    size: response.headers.get('content-length')
                });

                return response;
            } catch (error) {
                this.sendMetric('network_error', {
                    url: args[0],
                    error: error.message,
                    duration: Date.now() - startTime
                });
                throw error;
            }
        };
    }

    trackUserInteractions() {
        document.addEventListener('click', (event) => {
            this.sendMetric('user_interaction', {
                type: 'click',
                element: event.target.tagName,
                selector: this.getElementSelector(event.target),
                timestamp: Date.now()
            });
        });
    }

    sendMetric(type, data) {
        fetch(this.endpoint, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify({
                appId: this.appId,
                type: type,
                data: data,
                timestamp: Date.now(),
                userId: this.getUserId(),
                sessionId: this.getSessionId()
            })
        }).catch(err => console.error('Failed to send RUM metric:', err));
    }

    getFirstPaint() {
        const paintEntries = performance.getEntriesByType('paint');
        const fp = paintEntries.find(entry => entry.name === 'first-paint');
        return fp ? fp.startTime : null;
    }

    getLargestContentfulPaint() {
        const lcpEntries = performance.getEntriesByType('largest-contentful-paint');
        return lcpEntries.length > 0 ? lcpEntries[0].startTime : null;
    }

    getElementSelector(element) {
        // Generate CSS selector for element
        if (element.id) return `#${element.id}`;
        if (element.className) return `.${element.className.split(' ')[0]}`;

        let path = [];
        while (element.nodeType === Node.ELEMENT_NODE) {
            let selector = element.nodeName.toLowerCase();
            if (element.id) {
                selector += `#${element.id}`;
                path.unshift(selector);
                break;
            } else if (element.className) {
                selector += `.${element.className.split(' ')[0]}`;
            }

            let sibling = element.previousSibling;
            let nth = 1;
            while (sibling) {
                if (sibling.nodeType === Node.ELEMENT_NODE &&
                    sibling.nodeName.toLowerCase() === selector) {
                    nth++;
                }
                sibling = sibling.previousSibling;
            }

            if (nth !== 1) selector += `:nth-child(${nth})`;
            path.unshift(selector);

            element = element.parentNode;
        }

        return path.join(' > ');
    }

    getUserId() {
        return localStorage.getItem('userId') || 'anonymous';
    }

    getSessionId() {
        let sessionId = sessionStorage.getItem('sessionId');
        if (!sessionId) {
            sessionId = Math.random().toString(36).substring(2);
            sessionStorage.setItem('sessionId', sessionId);
        }
        return sessionId;
    }
}

// Initialize RUM monitoring
const rum = new RUMMonitor('my-app', '/api/rum');
```

## Cost Monitoring and Optimization

### Cloud Cost Monitoring
```python
class CostMonitor:
    def __init__(self, cloud_provider='aws'):
        self.cloud_provider = cloud_provider
        self.cost_history = {}
        self.budgets = {}

    def set_budget(self, service, monthly_budget):
        """Set cost budget for a service"""
        self.budgets[service] = monthly_budget

    def track_costs(self, service, cost, timestamp):
        """Track costs over time"""
        if service not in self.cost_history:
            self.cost_history[service] = []

        self.cost_history[service].append({
            'cost': cost,
            'timestamp': timestamp
        })

        # Keep only last 30 days
        cutoff = timestamp - (30 * 24 * 60 * 60)
        self.cost_history[service] = [
            entry for entry in self.cost_history[service]
            if entry['timestamp'] > cutoff
        ]

    def check_budget_alerts(self):
        """Check if any budgets are exceeded"""
        alerts = []

        for service, budget in self.budgets.items():
            if service in self.cost_history:
                monthly_cost = sum(
                    entry['cost'] for entry in self.cost_history[service]
                    if entry['timestamp'] > time.time() - (30 * 24 * 60 * 60)
                )

                if monthly_cost > budget * 0.8:  # 80% threshold
                    alerts.append({
                        'service': service,
                        'current_cost': monthly_cost,
                        'budget': budget,
                        'percentage': (monthly_cost / budget) * 100
                    })

        return alerts

    def get_cost_optimization_recommendations(self):
        """Generate cost optimization recommendations"""
        recommendations = []

        # Analyze usage patterns
        for service, history in self.cost_history.items():
            if len(history) > 7:  # Need at least a week of data
                daily_costs = [entry['cost'] for entry in history[-7:]]

                # Check for unused resources (very low cost)
                avg_daily_cost = sum(daily_costs) / len(daily_costs)
                if avg_daily_cost < 1.0:  # Less than $1/day
                    recommendations.append({
                        'type': 'unused_resource',
                        'service': service,
                        'message': f'Consider terminating unused {service} resource',
                        'potential_savings': avg_daily_cost * 30
                    })

        return recommendations
```

## Chaos Engineering Integration

### Automated Chaos Experiments
```python
class ChaosController:
    def __init__(self, monitoring_system):
        self.monitoring = monitoring_system
        self.experiments = []

    def define_experiment(self, name, hypothesis, method, duration):
        """Define a chaos experiment"""
        experiment = {
            'name': name,
            'hypothesis': hypothesis,
            'method': method,
            'duration': duration,
            'status': 'defined'
        }
        self.experiments.append(experiment)
        return experiment

    def run_experiment(self, experiment):
        """Run a chaos experiment"""
        print(f"Starting chaos experiment: {experiment['name']}")
        print(f"Hypothesis: {experiment['hypothesis']}")

        # Record baseline metrics
        baseline = self.monitoring.get_current_metrics()

        # Execute chaos method
        experiment['status'] = 'running'
        experiment['start_time'] = time.time()

        try:
            experiment['method']()
            time.sleep(experiment['duration'])

            # Measure impact
            after = self.monitoring.get_current_metrics()
            impact = self.calculate_impact(baseline, after)

            experiment['status'] = 'completed'
            experiment['impact'] = impact

            print(f"Experiment completed. Impact: {impact}")

        except Exception as e:
            experiment['status'] = 'failed'
            experiment['error'] = str(e)
            print(f"Experiment failed: {e}")

    def calculate_impact(self, baseline, after):
        """Calculate the impact of chaos experiment"""
        impact = {}

        for metric, baseline_value in baseline.items():
            if metric in after:
                after_value = after[metric]
                if baseline_value != 0:
                    change_percent = ((after_value - baseline_value) / baseline_value) * 100
                    impact[metric] = change_percent
                else:
                    impact[metric] = 0

        return impact

# Example chaos experiments
def kill_random_pod():
    """Kill a random Kubernetes pod"""
    import subprocess
    result = subprocess.run(['kubectl', 'delete', 'pod', '--random'], capture_output=True)
    return result.returncode == 0

def increase_latency():
    """Add network latency using tc"""
    import subprocess
    # Add 100ms latency to all outgoing traffic
    subprocess.run(['tc', 'qdisc', 'add', 'dev', 'eth0', 'root', 'netem', 'delay', '100ms'])

def fill_disk_space():
    """Fill disk space to simulate disk full scenario"""
    import subprocess
    # Create a large file
    subprocess.run(['dd', 'if=/dev/zero', 'of=/tmp/fill_disk', 'bs=1M', 'count=100'])

# Define experiments
chaos = ChaosController(monitoring_system)

chaos.define_experiment(
    name="pod_failure",
    hypothesis="System remains available when random pods are killed",
    method=kill_random_pod,
    duration=300  # 5 minutes
)

chaos.define_experiment(
    name="network_latency",
    hypothesis="System performance degrades gracefully under high latency",
    method=increase_latency,
    duration=180  # 3 minutes
)
```

## Metrics Collection

### System Metrics
```python
import psutil
import time
from prometheus_client import Counter, Histogram, Gauge, start_http_server

# Define metrics
REQUEST_COUNT = Counter('http_requests_total', 'Total HTTP requests', ['method', 'endpoint'])
REQUEST_DURATION = Histogram('http_request_duration_seconds', 'HTTP request duration')
CPU_USAGE = Gauge('cpu_usage_percent', 'CPU usage percentage')
MEMORY_USAGE = Gauge('memory_usage_bytes', 'Memory usage in bytes')

class MetricsCollector:
    def __init__(self):
        self.start_time = time.time()
    
    def collect_system_metrics(self):
        # CPU metrics
        cpu_percent = psutil.cpu_percent(interval=1)
        CPU_USAGE.set(cpu_percent)
        
        # Memory metrics
        memory = psutil.virtual_memory()
        MEMORY_USAGE.set(memory.used)
        
        # Disk metrics
        disk = psutil.disk_usage('/')
        DISK_USAGE.set(disk.used)
    
    def record_request(self, method, endpoint, duration):
        REQUEST_COUNT.labels(method=method, endpoint=endpoint).inc()
        REQUEST_DURATION.observe(duration)
    
    def start_metrics_server(self, port=8000):
        start_http_server(port)
```

### Application Metrics
```python
class ApplicationMetrics:
    def __init__(self):
        self.user_registrations = Counter('user_registrations_total')
        self.active_users = Gauge('active_users_current')
        self.database_connections = Gauge('database_connections_active')
        self.cache_hit_ratio = Gauge('cache_hit_ratio')
    
    def track_user_registration(self):
        self.user_registrations.inc()
    
    def update_active_users(self, count):
        self.active_users.set(count)
    
    def track_database_performance(self, active_connections, hit_ratio):
        self.database_connections.set(active_connections)
        self.cache_hit_ratio.set(hit_ratio)
```

## Logging Architecture

### Structured Logging
```python
import json
import logging
from datetime import datetime

class StructuredLogger:
    def __init__(self, service_name):
        self.service_name = service_name
        self.logger = logging.getLogger(service_name)
        
        # Configure JSON formatter
        formatter = JsonFormatter()
        handler = logging.StreamHandler()
        handler.setFormatter(formatter)
        self.logger.addHandler(handler)
        self.logger.setLevel(logging.INFO)
    
    def log_request(self, request_id, method, path, user_id=None):
        log_data = {
            'event_type': 'http_request',
            'request_id': request_id,
            'method': method,
            'path': path,
            'user_id': user_id,
            'timestamp': datetime.utcnow().isoformat()
        }
        self.logger.info(json.dumps(log_data))
    
    def log_error(self, request_id, error, stack_trace=None):
        log_data = {
            'event_type': 'error',
            'request_id': request_id,
            'error': str(error),
            'stack_trace': stack_trace,
            'timestamp': datetime.utcnow().isoformat()
        }
        self.logger.error(json.dumps(log_data))
    
    def log_business_event(self, event_type, user_id, properties=None):
        log_data = {
            'event_type': event_type,
            'user_id': user_id,
            'properties': properties or {},
            'timestamp': datetime.utcnow().isoformat()
        }
        self.logger.info(json.dumps(log_data))

class JsonFormatter(logging.Formatter):
    def format(self, record):
        log_entry = {
            'level': record.levelname,
            'logger': record.name,
            'message': record.getMessage(),
            'timestamp': datetime.utcnow().isoformat()
        }
        
        if hasattr(record, 'extra_data'):
            log_entry.update(record.extra_data)
        
        return json.dumps(log_entry)
```

### Log Aggregation
```python
import asyncio
import aiohttp
from elasticsearch import Elasticsearch

class LogAggregator:
    def __init__(self, elasticsearch_url):
        self.es = Elasticsearch([elasticsearch_url])
        self.index_pattern = "logs-{date}"
    
    async def aggregate_logs(self, log_entries):
        # Bulk index logs to Elasticsearch
        actions = []
        for entry in log_entries:
            index_name = self.index_pattern.format(
                date=datetime.utcnow().strftime('%Y.%m.%d')
            )
            
            action = {
                "_index": index_name,
                "_source": entry
            }
            actions.append(action)
        
        # Bulk insert
        from elasticsearch.helpers import bulk
        bulk(self.es, actions)
    
    def search_logs(self, query, start_time, end_time, size=100):
        search_body = {
            "query": {
                "bool": {
                    "must": [
                        {"range": {"timestamp": {"gte": start_time, "lte": end_time}}},
                        {"query_string": {"query": query}}
                    ]
                }
            },
            "sort": [{"timestamp": {"order": "desc"}}],
            "size": size
        }
        
        response = self.es.search(
            index="logs-*",
            body=search_body
        )
        
        return [hit['_source'] for hit in response['hits']['hits']]
```

## Distributed Tracing

### OpenTelemetry Implementation
```python
from opentelemetry import trace
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.instrumentation.flask import FlaskInstrumentor

class TracingSetup:
    def __init__(self, service_name):
        self.service_name = service_name
        self.setup_tracing()
    
    def setup_tracing(self):
        # Set up tracer
        trace.set_tracer_provider(TracerProvider())
        tracer = trace.get_tracer(__name__)
        
        # Set up Jaeger exporter
        jaeger_exporter = JaegerExporter(
            agent_host_name="localhost",
            agent_port=6831,
        )
        
        # Add span processor
        span_processor = BatchSpanProcessor(jaeger_exporter)
        trace.get_tracer_provider().add_span_processor(span_processor)
        
        # Instrument libraries
        RequestsInstrumentor().instrument()
        FlaskInstrumentor().instrument()

class TracedService:
    def __init__(self):
        self.tracer = trace.get_tracer(__name__)
    
    def process_request(self, request_data):
        with self.tracer.start_as_current_span("process_request") as span:
            span.set_attribute("request.size", len(str(request_data)))
            
            # Process data
            result = self.process_data(request_data)
            
            span.set_attribute("result.size", len(str(result)))
            return result
    
    def process_data(self, data):
        with self.tracer.start_as_current_span("process_data") as span:
            # Simulate database call
            with self.tracer.start_as_current_span("database_query"):
                result = self.database_query(data)
                span.set_attribute("db.query", "SELECT * FROM users")
            
            return result
    
    def database_query(self, data):
        # Simulate database operation
        time.sleep(0.1)
        return {"status": "success", "data": data}
```

### Custom Span Creation
```python
class CustomTracing:
    def __init__(self):
        self.tracer = trace.get_tracer(__name__)
    
    def trace_business_operation(self, operation_name, user_id, **kwargs):
        def decorator(func):
            def wrapper(*args, **func_kwargs):
                with self.tracer.start_as_current_span(operation_name) as span:
                    span.set_attribute("user.id", user_id)
                    
                    # Add custom attributes
                    for key, value in kwargs.items():
                        span.set_attribute(f"operation.{key}", value)
                    
                    try:
                        result = func(*args, **func_kwargs)
                        span.set_attribute("operation.success", True)
                        return result
                    except Exception as e:
                        span.set_attribute("operation.success", False)
                        span.set_attribute("operation.error", str(e))
                        span.record_exception(e)
                        raise
            
            return wrapper
        return decorator

# Usage
@CustomTracing().trace_business_operation(
    "user_registration", 
    user_id="user123",
    operation_type="create"
)
def register_user(user_data):
    # Registration logic
    pass
```

## Alerting and Notification

### Alert Rule Engine
```python
import asyncio
from datetime import datetime, timedelta

class AlertManager:
    def __init__(self):
        self.alert_rules = []
        self.notification_channels = []
        self.alert_history = {}
    
    def add_rule(self, rule):
        self.alert_rules.append(rule)
    
    def add_notification_channel(self, channel):
        self.notification_channels.append(channel)
    
    async def evaluate_rules(self):
        while True:
            for rule in self.alert_rules:
                try:
                    await self.evaluate_rule(rule)
                except Exception as e:
                    print(f"Error evaluating rule {rule.name}: {e}")
            
            await asyncio.sleep(60)  # Evaluate every minute
    
    async def evaluate_rule(self, rule):
        # Get current metrics
        current_value = await rule.get_metric_value()
        
        # Check if alert condition is met
        if rule.should_alert(current_value):
            alert_key = f"{rule.name}_{rule.get_alert_key()}"
            
            # Check if already alerted
            if not self.is_alert_active(alert_key):
                await self.trigger_alert(rule, current_value)
                self.set_alert_active(alert_key)
        else:
            # Clear alert if condition resolved
            alert_key = f"{rule.name}_{rule.get_alert_key()}"
            if self.is_alert_active(alert_key):
                await self.clear_alert(rule)
                self.set_alert_inactive(alert_key)
    
    async def trigger_alert(self, rule, value):
        alert = {
            'rule_name': rule.name,
            'severity': rule.severity,
            'message': rule.get_alert_message(value),
            'timestamp': datetime.utcnow(),
            'value': value
        }
        
        # Send notifications
        for channel in self.notification_channels:
            await channel.send_notification(alert)
    
    def is_alert_active(self, alert_key):
        return self.alert_history.get(alert_key, {}).get('active', False)
    
    def set_alert_active(self, alert_key):
        if alert_key not in self.alert_history:
            self.alert_history[alert_key] = {}
        self.alert_history[alert_key]['active'] = True
        self.alert_history[alert_key]['triggered_at'] = datetime.utcnow()
    
    def set_alert_inactive(self, alert_key):
        if alert_key in self.alert_history:
            self.alert_history[alert_key]['active'] = False
            self.alert_history[alert_key]['resolved_at'] = datetime.utcnow()

class AlertRule:
    def __init__(self, name, severity, condition, message_template):
        self.name = name
        self.severity = severity
        self.condition = condition
        self.message_template = message_template
    
    async def get_metric_value(self):
        # Override in subclasses
        pass
    
    def should_alert(self, value):
        return self.condition(value)
    
    def get_alert_message(self, value):
        return self.message_template.format(value=value)
    
    def get_alert_key(self):
        # Override for multi-dimensional alerts
        return "default"

class CPUAlertRule(AlertRule):
    def __init__(self):
        super().__init__(
            name="high_cpu_usage",
            severity="warning",
            condition=lambda x: x > 80,
            message_template="CPU usage is {value}%"
        )
    
    async def get_metric_value(self):
        # Get CPU usage from monitoring system
        return psutil.cpu_percent(interval=1)
```

### Notification Channels
```python
import smtplib
import requests
from email.mime.text import MIMEText

class EmailNotificationChannel:
    def __init__(self, smtp_server, smtp_port, username, password):
        self.smtp_server = smtp_server
        self.smtp_port = smtp_port
        self.username = username
        self.password = password
    
    async def send_notification(self, alert):
        subject = f"[{alert['severity'].upper()}] {alert['rule_name']}"
        body = f"""
        Alert: {alert['rule_name']}
        Severity: {alert['severity']}
        Message: {alert['message']}
        Timestamp: {alert['timestamp']}
        Value: {alert['value']}
        """
        
        msg = MIMEText(body)
        msg['Subject'] = subject
        msg['From'] = self.username
        msg['To'] = 'alerts@company.com'
        
        with smtplib.SMTP(self.smtp_server, self.smtp_port) as server:
            server.starttls()
            server.login(self.username, self.password)
            server.send_message(msg)

class SlackNotificationChannel:
    def __init__(self, webhook_url):
        self.webhook_url = webhook_url
    
    async def send_notification(self, alert):
        color = {
            'info': '#36a64f',
            'warning': '#ff9500',
            'critical': '#ff0000'
        }.get(alert['severity'], '#36a64f')
        
        payload = {
            'attachments': [{
                'color': color,
                'title': f"Alert: {alert['rule_name']}",
                'text': alert['message'],
                'fields': [
                    {'title': 'Severity', 'value': alert['severity'], 'short': True},
                    {'title': 'Value', 'value': str(alert['value']), 'short': True},
                    {'title': 'Timestamp', 'value': str(alert['timestamp']), 'short': False}
                ]
            }]
        }
        
        response = requests.post(self.webhook_url, json=payload)
        response.raise_for_status()
```

## Dashboard and Visualization

### Grafana Dashboard Configuration
```json
{
  "dashboard": {
    "title": "Application Performance Dashboard",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{endpoint}}"
          }
        ]
      },
      {
        "title": "Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "95th percentile"
          },
          {
            "expr": "histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "50th percentile"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "singlestat",
        "targets": [
          {
            "expr": "rate(http_requests_total{status=~\"5..\"}[5m]) / rate(http_requests_total[5m])",
            "legendFormat": "Error Rate"
          }
        ]
      }
    ]
  }
}
```

## Performance Monitoring

### Application Performance Monitoring (APM)
```python
class APMCollector:
    def __init__(self):
        self.tracer = trace.get_tracer(__name__)
        self.metrics_collector = MetricsCollector()
    
    def trace_function_call(self, func_name):
        def decorator(func):
            def wrapper(*args, **kwargs):
                start_time = time.time()
                
                with self.tracer.start_as_current_span(func_name) as span:
                    try:
                        result = func(*args, **kwargs)
                        duration = time.time() - start_time
                        
                        # Record metrics
                        self.metrics_collector.record_function_duration(
                            func_name, duration
                        )
                        
                        span.set_attribute("function.success", True)
                        span.set_attribute("function.duration", duration)
                        
                        return result
                    except Exception as e:
                        duration = time.time() - start_time
                        
                        # Record error metrics
                        self.metrics_collector.record_function_error(
                            func_name, str(e)
                        )
                        
                        span.set_attribute("function.success", False)
                        span.set_attribute("function.error", str(e))
                        span.record_exception(e)
                        
                        raise
            
            return wrapper
        return decorator

# Usage
@APMCollector().trace_function_call("database_query")
def execute_query(sql, params):
    # Database query logic
    pass
```

## Real-World Examples

### Netflix Monitoring Stack
- **Metrics**: Atlas (custom time series database)
- **Logging**: ELK Stack (Elasticsearch, Logstash, Kibana)
- **Tracing**: Zipkin, custom tools
- **Alerting**: Alertmanager, custom alerting

### Google Monitoring
- **Metrics**: Prometheus + Stackdriver
- **Logging**: Cloud Logging
- **Tracing**: Cloud Trace (Dapper)
- **Alerting**: Cloud Monitoring

### Uber Monitoring
- **Metrics**: M3 (distributed metrics platform)
- **Logging**: ELK Stack
- **Tracing**: Jaeger
- **Alerting**: PagerDuty integration

## Interview Tips

### Common Questions
1. "What's the difference between monitoring and observability?"
2. "How would you design a monitoring system?"
3. "What metrics would you track for a web application?"
4. "How would you set up alerting?"
5. "What's the difference between SLI, SLO, and SLA?"
6. "How do you implement chaos engineering?"
7. "What are error budgets and how do you use them?"

### Answer Framework
1. **Define Requirements**: What needs to be monitored? SLIs/SLOs?
2. **Choose Tools**: Metrics, logging, tracing, RUM solutions
3. **Design Architecture**: Collection, storage, visualization, alerting
4. **Implement SLOs**: Define SLIs, set targets, track error budgets
5. **Set Up Alerting**: Rules, thresholds, notifications, escalation
6. **Add Chaos Engineering**: Define experiments, measure resilience
7. **Plan for Scale**: High cardinality, retention, cost optimization

### Key Points to Emphasize
- Three pillars of observability (metrics, logs, traces)
- Importance of structured logging and distributed tracing
- SLI/SLO/SLA framework for service level management
- Error budgets for balancing reliability and innovation
- Chaos engineering for building resilient systems
- Cost monitoring and optimization in cloud environments
- User experience monitoring (RUM) for end-to-end visibility

## Practice Problems

1. **Design monitoring system for microservices with SLOs**
2. **Implement distributed tracing with OpenTelemetry**
3. **Create alerting rules and error budgets for e-commerce platform**
4. **Design log aggregation system with anomaly detection**
5. **Monitor database performance with ML-based anomaly detection**
6. **Implement chaos engineering experiments for a critical service**
7. **Design real user monitoring (RUM) for a web application**
8. **Create cost monitoring and optimization system for cloud infrastructure**

## Further Reading

- **Books**: "Observability Engineering" by Charity Majors
- **Documentation**: Prometheus Documentation, OpenTelemetry Guide
- **Concepts**: SRE principles, SLI/SLO/SLA
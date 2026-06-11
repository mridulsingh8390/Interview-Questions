# Monitoring & Observability — Complete Study Guide
### All Topics · Detailed Q&A · Master Cheatsheet
**2025 Edition — Prometheus · Grafana · Loki · Tempo · Alertmanager · Datadog · New Relic · FluentBit · FluentD · OpenTelemetry · Vector**

---

## Table of Contents
- [Prometheus](#prometheus)
- [Prometheus Operator CRDs](#prometheus-operator)
- [Alertmanager Deep Dive](#alertmanager)
- [Grafana](#grafana)
- [Grafana Loki + LogQL](#loki)
- [Grafana Tempo (Traces)](#tempo)
- [OpenTelemetry (OTel)](#opentelemetry)
- [Datadog](#datadog)
- [New Relic](#new-relic)
- [FluentBit](#fluentbit)
- [FluentD](#fluentd)
- [Vector](#vector)
- [Log Pipeline Design](#log-pipeline)
- [Master Cheatsheet](#master-cheatsheet)

---

## Prometheus

### 🟢 Q1. What is Prometheus and how does it work?

```yaml
# Prometheus pull-based architecture:
# Targets expose /metrics endpoint (HTTP, text format)
# Prometheus scrapes targets on schedule (default 15s)
# Stores time-series data locally (TSDB)
# Alerts via Alertmanager

# prometheus.yaml — basic config
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: production
    region: us-east-1

rule_files:
  - /etc/prometheus/rules/*.yaml

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']

  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: (.+)
        replacement: $1
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app

  - job_name: kubernetes-nodes
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
```

---

### 🟢 Q2. What are metric types in Prometheus?

```python
# Counter — monotonically increasing (total requests, errors)
from prometheus_client import Counter
requests_total = Counter('http_requests_total', 'Total HTTP requests',
    labelnames=['method', 'path', 'status'])
requests_total.labels(method='GET', path='/api', status='200').inc()

# Gauge — can go up or down (current connections, memory usage)
from prometheus_client import Gauge
active_connections = Gauge('active_connections', 'Number of active connections')
active_connections.set(42)
active_connections.inc()
active_connections.dec()

# Histogram — samples observations, calculates buckets (request duration)
from prometheus_client import Histogram
request_duration = Histogram('http_request_duration_seconds', 'Request duration',
    labelnames=['method', 'path'],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0])
with request_duration.labels(method='GET', path='/api').time():
    process_request()

# Summary — percentile calculation on client side
from prometheus_client import Summary
request_latency = Summary('request_latency_seconds', 'Request latency',
    quantiles=[0.5, 0.9, 0.99])
with request_latency.time():
    process_request()
```

---

### 🟡 Q3. How do you write PromQL queries?

```promql
# ===== SELECTORS =====
http_requests_total                              # All time series
http_requests_total{job="api"}                   # Filter by label
http_requests_total{status=~"5.."}               # Regex match (5xx errors)
http_requests_total{status!="200"}               # Not equal
http_requests_total{path!~"/health.*"}           # Negative regex

# ===== FUNCTIONS =====
rate(http_requests_total[5m])                    # Per-second rate over 5m window
irate(http_requests_total[5m])                   # Instantaneous rate (last 2 samples)
increase(http_requests_total[1h])                # Total increase over 1 hour
delta(temperature_celsius[1h])                   # Change (for gauges)

# ===== AGGREGATION =====
sum(rate(http_requests_total[5m]))               # Sum across all series
sum by (status) (rate(http_requests_total[5m]))  # Sum grouped by status
sum without (instance) (rate(http_requests_total[5m]))  # Sum dropping instance label
avg(http_request_duration_seconds)
max(http_request_duration_seconds) by (pod)
min by (namespace) (kube_pod_container_resource_requests{resource="cpu"})
count(up == 0)                                   # Count down targets
topk(5, rate(http_requests_total[5m]))           # Top 5 by request rate
bottomk(3, container_cpu_usage_seconds_total)

# ===== ARITHMETIC =====
# Error rate %
sum(rate(http_requests_total{status=~"5.."}[5m]))
/ sum(rate(http_requests_total[5m])) * 100

# CPU utilization %
100 * (1 - avg by (node) (rate(node_cpu_seconds_total{mode="idle"}[5m])))

# Memory usage GB
container_memory_working_set_bytes / 1024 / 1024 / 1024

# ===== HISTOGRAM PERCENTILES =====
histogram_quantile(0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
)

histogram_quantile(0.95,
  sum by (le, service) (rate(http_request_duration_seconds_bucket[5m]))
)

# ===== USEFUL KUBERNETES QUERIES =====
# Pod CPU usage (cores)
sum by (pod, namespace) (
  rate(container_cpu_usage_seconds_total{container!=""}[5m])
)

# Pod memory (MB)
sum by (pod, namespace) (
  container_memory_working_set_bytes{container!=""}
) / 1024 / 1024

# Node memory available %
(node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Disk usage %
(1 - node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100

# PVC usage %
(1 - kubelet_volume_stats_available_bytes / kubelet_volume_stats_capacity_bytes) * 100

# HTTP error rate
sum by (namespace, pod) (
  rate(istio_requests_total{response_code=~"5.."}[5m])
) / sum by (namespace, pod) (
  rate(istio_requests_total[5m])
)

# Pods not running
kube_pod_status_phase{phase!="Running",phase!="Succeeded"} == 1

# Deployment replicas mismatch
kube_deployment_spec_replicas != kube_deployment_status_ready_replicas
```

---

### 🟡 Q4. How do you write alerting rules?

```yaml
# /etc/prometheus/rules/alerts.yaml
groups:
  - name: service-alerts
    interval: 1m                    # Evaluate this group every 1 min
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          / sum(rate(http_requests_total[5m])) > 0.05
        for: 5m                     # Must be true for 5m before firing
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High error rate on {{ $labels.service }}"
          description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.service }}"
          runbook: "https://wiki.example.com/runbooks/high-error-rate"

      - alert: PodCrashLooping
        expr: |
          increase(kube_pod_container_status_restarts_total[1h]) > 5
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.pod }} is crash looping"
          description: "Container {{ $labels.container }} in pod {{ $labels.pod }} has restarted {{ $value }} times in the last hour."

      - alert: HighP99Latency
        expr: |
          histogram_quantile(0.99,
            sum by (le, service) (rate(http_request_duration_seconds_bucket[5m]))
          ) > 2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High P99 latency on {{ $labels.service }}"
          description: "P99 latency is {{ $value }}s"

      - alert: DiskSpaceLow
        expr: |
          (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes) * 100 < 15
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk space low on {{ $labels.instance }}"
          description: "Only {{ $value | humanize }}% free"

      - alert: KubernetesNodeNotReady
        expr: kube_node_status_condition{condition="Ready", status="true"} == 0
        for: 5m
        labels:
          severity: critical

      - alert: DeploymentReplicasMismatch
        expr: kube_deployment_spec_replicas != kube_deployment_status_ready_replicas
        for: 10m
        labels:
          severity: warning
```

---

### 🟡 Q5. How do you deploy Prometheus on Kubernetes with kube-prometheus-stack?

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=100Gi \
  --set grafana.adminPassword=admin123 \
  -f values.yaml

# values.yaml
alertmanager:
  enabled: true
  config:
    global:
      slack_api_url: 'https://hooks.slack.com/services/...'
    route:
      receiver: 'slack-critical'
      group_by: ['alertname', 'namespace']
      routes:
        - match:
            severity: critical
          receiver: slack-critical
        - match:
            severity: warning
          receiver: slack-warning
    receivers:
      - name: slack-critical
        slack_configs:
          - channel: '#alerts-critical'
            title: 'CRITICAL: {{ .CommonAnnotations.summary }}'
            text: '{{ .CommonAnnotations.description }}'
```

---

## Prometheus Operator CRDs

### 🟡 Q6. What are the Prometheus Operator CRDs?

```yaml
# ===== ServiceMonitor — scrape a Service's pods =====
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: monitoring
  labels:
    app: my-app                         # Must match Prometheus.spec.serviceMonitorSelector
spec:
  selector:
    matchLabels:
      app: my-app                       # Select Services with this label
  namespaceSelector:
    matchNames:
      - production
  endpoints:
    - port: metrics                     # Port name in Service
      path: /metrics
      interval: 15s
      scrapeTimeout: 10s
      honorLabels: true
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_name]
          targetLabel: pod
      metricRelabelings:
        - sourceLabels: [__name__]
          regex: "go_.*"               # Drop Go runtime metrics
          action: drop

---
# ===== PodMonitor — scrape pods directly (no Service needed) =====
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: my-app-pods
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: my-app
  namespaceSelector:
    matchNames:
      - production
  podMetricsEndpoints:
    - port: metrics
      path: /metrics
      interval: 30s

---
# ===== PrometheusRule — alerting + recording rules =====
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-app-rules
  namespace: production
  labels:
    prometheus: kube-prometheus        # Must match Prometheus.spec.ruleSelector
    role: alert-rules
spec:
  groups:
    - name: my-app.rules
      interval: 1m
      rules:
        - alert: MyAppHighErrorRate
          expr: |
            sum(rate(http_requests_total{app="my-app",status=~"5.."}[5m]))
            / sum(rate(http_requests_total{app="my-app"}[5m])) > 0.05
          for: 5m
          labels:
            severity: critical
            app: my-app
          annotations:
            summary: "High error rate"
            description: "Error rate: {{ $value | humanizePercentage }}"

        # Recording rule — pre-compute expensive query
        - record: job:http_requests:rate5m
          expr: sum by (job) (rate(http_requests_total[5m]))

---
# ===== Alertmanager CRD =====
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: my-app-alerting
  namespace: production
spec:
  route:
    groupBy: ['alertname']
    receiver: slack
  receivers:
    - name: slack
      slackConfigs:
        - channel: '#my-app-alerts'
          apiURL:
            name: slack-webhook-secret
            key: webhook_url
```

---

## Alertmanager Deep Dive

### 🟡 Q7. How does Alertmanager work in detail?

```yaml
# alertmanager.yaml — full configuration
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/...'
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'alertmanager@example.com'
  smtp_auth_password: 'password'

# ===== ROUTING TREE =====
route:
  receiver: default                    # Fallback receiver
  group_by: ['alertname', 'cluster', 'namespace']
  group_wait: 30s                      # Wait to group initial alerts
  group_interval: 5m                   # Wait before re-alerting grouped alerts
  repeat_interval: 4h                  # Re-alert if still firing after 4h

  routes:
    # Critical → PagerDuty immediately
    - match:
        severity: critical
      receiver: pagerduty
      group_wait: 10s                  # Faster for critical
      repeat_interval: 1h

    # Database alerts → DBA team
    - match_re:
        alertname: "^(MySQL|Postgres|Redis).*"
      receiver: dba-slack
      group_by: ['alertname', 'instance']

    # Ignore info-level during business hours
    - match:
        severity: info
      time_intervals:
        - business-hours               # Reference to time_intervals below
      receiver: null-receiver          # Silently discard

    # Infrastructure → ops channel
    - match:
        team: ops
      receiver: ops-slack
      continue: true                   # Don't stop — also check parent routes

# ===== INHIBITION (silence redundant alerts) =====
inhibit_rules:
  # If a cluster is down (critical), suppress all individual alerts from it
  - source_match:
      severity: critical
      alertname: ClusterDown
    target_match_re:
      severity: "warning|info"
    equal:
      - cluster                        # Only inhibit if same cluster

  # If node is down, suppress pod/service alerts from that node
  - source_match:
      alertname: NodeNotReady
    target_match_re:
      alertname: "(PodCrashLooping|ServiceEndpointDown)"
    equal:
      - node

# ===== TIME INTERVALS =====
time_intervals:
  - name: business-hours
    time_intervals:
      - weekdays: ['monday:friday']
        times:
          - start_time: '09:00'
            end_time: '18:00'
  - name: on-call-hours
    time_intervals:
      - times:
          - start_time: '18:00'
            end_time: '09:00'

# ===== RECEIVERS =====
receivers:
  - name: default
    slack_configs:
      - channel: '#alerts-general'

  - name: pagerduty
    pagerduty_configs:
      - routing_key: 'pd-integration-key'
        description: '{{ template "pagerduty.description" . }}'
        severity: '{{ .CommonLabels.severity }}'

  - name: dba-slack
    slack_configs:
      - channel: '#dba-alerts'
        title: '{{ .CommonAnnotations.summary }}'
        text: '{{ .CommonAnnotations.description }}'
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
        actions:
          - type: button
            text: 'Runbook :books:'
            url: '{{ .CommonAnnotations.runbook }}'
          - type: button
            text: 'Silence :no_bell:'
            url: '{{ template "__alertmanagerURL" . }}/#/silences/new?filter={alertname="{{ .CommonLabels.alertname }}"}'

  - name: ops-slack
    slack_configs:
      - channel: '#ops-alerts'
    email_configs:
      - to: 'ops-team@example.com'

  - name: null-receiver
    # Empty — silently discard
```

```bash
# Alertmanager API
# Create silence (snooze an alert)
curl -X POST http://alertmanager:9093/api/v2/silences \
  -H "Content-Type: application/json" \
  -d '{
    "matchers": [
      {"name": "alertname", "value": "HighErrorRate", "isRegex": false},
      {"name": "namespace", "value": "production", "isRegex": false}
    ],
    "startsAt": "2024-01-01T00:00:00Z",
    "endsAt": "2024-01-01T02:00:00Z",
    "createdBy": "john@example.com",
    "comment": "Planned maintenance"
  }'

# List active alerts
curl http://alertmanager:9093/api/v2/alerts

# Check config
curl http://alertmanager:9093/api/v2/status
```

---

## Grafana

### 🟢 Q7. What is Grafana and how do you configure data sources?

```yaml
# data sources via provisioning (GitOps)
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
data:
  datasources.yaml: |
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        uid: prometheus
        url: http://prometheus-server.monitoring:9090
        isDefault: true
        jsonData:
          timeInterval: "15s"
          httpMethod: POST

      - name: Loki
        type: loki
        uid: loki
        url: http://loki.monitoring:3100
        jsonData:
          maxLines: 1000
          derivedFields:
            - datasourceUid: jaeger
              matcherRegex: "traceID=(\\w+)"
              name: TraceID
              url: "$${__value.raw}"
              urlDisplayLabel: "Jaeger Trace"

      - name: Jaeger
        type: jaeger
        uid: jaeger
        url: http://jaeger-query.monitoring:16686

      - name: Tempo
        type: tempo
        uid: tempo
        url: http://tempo.monitoring:3200
        jsonData:
          tracesToLogs:
            datasourceUid: loki
            tags: [{ key: "service.name", value: "service" }]
```

---

### 🟡 Q8. How do you create and provision Grafana dashboards as code?

```json
// Dashboard JSON — provision via ConfigMap
{
  "__inputs": [{ "name": "DS_PROMETHEUS", "type": "datasource" }],
  "__requires": [{ "type": "datasource", "id": "prometheus" }],
  "title": "Application Dashboard",
  "uid": "my-app-dashboard",
  "schemaVersion": 38,
  "refresh": "1m",
  "time": { "from": "now-1h", "to": "now" },
  "panels": [
    {
      "title": "Request Rate",
      "type": "timeseries",
      "datasource": { "uid": "prometheus" },
      "targets": [
        {
          "expr": "sum by (status) (rate(http_requests_total{app=\"my-app\"}[5m]))",
          "legendFormat": "{{ status }}"
        }
      ],
      "fieldConfig": {
        "defaults": { "unit": "reqps" }
      }
    },
    {
      "title": "P99 Latency",
      "type": "timeseries",
      "targets": [
        {
          "expr": "histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{app=\"my-app\"}[5m])))",
          "legendFormat": "p99"
        }
      ]
    }
  ]
}
```

```yaml
# Provision dashboard via ConfigMap (grafana sidecar)
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"          # Sidecar watches for this label
data:
  my-app.json: |
    { ... dashboard JSON ... }
```

---

## Grafana Loki + LogQL

### 🟡 Q9. What is Grafana Loki and how do you query with LogQL?

```
Loki = "Prometheus, but for logs"
- Indexes only labels (not full text) → cost-efficient
- Stores log content in object storage (S3, GCS)
- LogQL — SQL-like query language for logs
- Integrates natively with Grafana
```

```bash
# Install Loki stack
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set grafana.enabled=false \
  --set prometheus.enabled=false \
  --set promtail.enabled=true
```

```yaml
# promtail config (ships logs to Loki)
config:
  clients:
    - url: http://loki:3100/loki/api/v1/push
  scrape_configs:
    - job_name: kubernetes-pods
      kubernetes_sd_configs:
        - role: pod
      pipeline_stages:
        - cri: {}
        - json:
            expressions:
              level: level
              msg: message
              trace_id: traceId
        - labels:
            level:
            trace_id:
      relabel_configs:
        - source_labels: [__meta_kubernetes_pod_label_app]
          target_label: app
        - source_labels: [__meta_kubernetes_namespace]
          target_label: namespace
        - source_labels: [__meta_kubernetes_pod_name]
          target_label: pod
```

```logql
# ===== LOGQL QUERIES =====

# Basic stream selector
{app="my-app"}
{namespace="production", pod=~"my-app-.*"}

# Filter by content
{app="my-app"} |= "error"          # Contains "error"
{app="my-app"} != "debug"          # Doesn't contain "debug"
{app="my-app"} |~ "ERROR|WARN"     # Regex match
{app="my-app"} !~ "healthcheck"    # Negative regex

# Parse JSON logs
{app="my-app"} | json              # Auto-parse JSON
{app="my-app"} | json level, msg, traceId  # Extract specific fields

# Parse logfmt (key=value format)
{app="my-app"} | logfmt

# Regex parser
{app="my-app"} | regexp `(?P<level>\w+) (?P<msg>.*)`

# Filter on extracted labels
{app="my-app"} | json | level="error"
{app="my-app"} | json | duration > 1s

# ===== METRIC QUERIES (LogQL metrics) =====
# Request rate from logs
rate({app="my-app"} |= "HTTP" [5m])

# Error rate
sum(rate({app="my-app"} | json | level="error" [5m]))

# Count errors by endpoint
sum by (path) (
  rate({app="my-app"} | json | level="error" [5m])
)

# P99 latency from log data
quantile_over_time(0.99,
  {app="my-app"} | json | unwrap duration_ms [5m]
)

# Count unique users
count_over_time({app="my-app"} | json | user != "" [1h])
```

---

## Grafana Tempo (Traces)

### 🟡 Q10. What is Grafana Tempo and how do you set it up?

```
Tempo = distributed tracing backend
- Compatible with Jaeger, Zipkin, OpenTelemetry
- Stores traces in object storage (S3/GCS/Azure Blob)
- Very cost-efficient (no indexing, only trace ID lookup)
- Integrates with Grafana for trace visualization
- Works with Loki via exemplars (link logs to traces)
```

```bash
helm install tempo grafana/tempo-distributed \
  --namespace monitoring \
  -f values.yaml
```

```yaml
# tempo values.yaml
storage:
  trace:
    backend: s3
    s3:
      bucket: my-tempo-traces
      region: us-east-1

distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318
    jaeger:
      protocols:
        thrift_http:
          endpoint: 0.0.0.0:14268
        grpc:
          endpoint: 0.0.0.0:14250
    zipkin:
      endpoint: 0.0.0.0:9411

querier:
  max_concurrent_queries: 20

compactor:
  compaction:
    block_retention: 720h    # 30 days
```

---

## OpenTelemetry (OTel)

### 🟡 Q11. What is OpenTelemetry and how do you implement it?

```
OpenTelemetry (OTel):
  - CNCF project — vendor-neutral observability standard
  - Unified API for traces, metrics, and logs
  - Replaces: OpenTracing, OpenCensus, vendor SDKs
  - OTel Collector: agent/gateway to receive, process, export signals

Architecture:
  App (OTel SDK) → OTel Collector → Backend (Tempo, Jaeger, Prometheus, Datadog, etc.)

Benefits:
  - Change backends without code changes
  - One agent for all observability signals
  - Rich automatic instrumentation
```

```python
# Python OTel SDK example
from opentelemetry import trace, metrics
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.sdk.resources import Resource

# Configure tracer
resource = Resource.create({"service.name": "my-api", "deployment.environment": "production"})
provider = TracerProvider(resource=resource)
provider.add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint="http://otel-collector:4317", insecure=True))
)
trace.set_tracer_provider(provider)

# Auto-instrument Flask
FlaskInstrumentor().instrument()
RequestsInstrumentor().instrument()

# Manual instrumentation
tracer = trace.get_tracer("my-service")

def process_order(order_id):
    with tracer.start_as_current_span("process-order") as span:
        span.set_attribute("order.id", order_id)
        span.set_attribute("order.type", "express")
        
        with tracer.start_as_current_span("validate-inventory"):
            result = check_inventory(order_id)
            span.set_attribute("inventory.available", result)
        
        with tracer.start_as_current_span("charge-payment") as payment_span:
            try:
                charge(order_id)
            except PaymentError as e:
                payment_span.record_exception(e)
                payment_span.set_status(trace.StatusCode.ERROR)
                raise
```

```yaml
# OTel Collector config (otel-collector-config.yaml)
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
  prometheus:
    config:
      scrape_configs:
        - job_name: my-app
          static_configs:
            - targets: ['my-app:8080']

processors:
  batch:
    timeout: 5s
    send_batch_size: 1000
  memory_limiter:
    limit_mib: 512
    check_interval: 1s
  resource:
    attributes:
      - key: cluster
        value: production
        action: upsert
  filter:
    traces:
      span:
        - 'attributes["http.url"] == "/health"'  # Drop health check traces

exporters:
  otlp:
    endpoint: tempo.monitoring:4317
    tls:
      insecure: true
  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write
  loki:
    endpoint: http://loki:3100/loki/api/v1/push
  datadog:
    api:
      key: ${DD_API_KEY}

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, resource, filter]
      exporters: [otlp]           # → Tempo
    metrics:
      receivers: [otlp, prometheus]
      processors: [memory_limiter, batch]
      exporters: [prometheusremotewrite]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [loki]
```

```yaml
# Deploy OTel Collector as DaemonSet (node agent mode)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: otel-collector
spec:
  selector:
    matchLabels:
      app: otel-collector
  template:
    spec:
      containers:
        - name: otel-collector
          image: otel/opentelemetry-collector-contrib:latest
          args: ["--config=/conf/otel-collector-config.yaml"]
          ports:
            - containerPort: 4317    # OTLP gRPC
            - containerPort: 4318    # OTLP HTTP
          volumeMounts:
            - name: config
              mountPath: /conf
          resources:
            limits: { cpu: 200m, memory: 256Mi }
            requests: { cpu: 100m, memory: 128Mi }
      volumes:
        - name: config
          configMap:
            name: otel-collector-config
```

---

## Datadog

### 🟢 Q12. What is Datadog and how do you install the agent?

```bash
# Install Datadog Agent on Kubernetes (Helm)
helm repo add datadog https://helm.datadoghq.com
helm install datadog-agent datadog/datadog \
  --namespace monitoring \
  --set datadog.apiKey=$DD_API_KEY \
  --set datadog.appKey=$DD_APP_KEY \
  --set datadog.site=datadoghq.com \
  --set agents.image.tag=latest \
  --set clusterAgent.enabled=true \
  --set datadog.logs.enabled=true \
  --set datadog.logs.containerCollectAll=true \
  --set datadog.apm.portEnabled=true \
  --set datadog.processAgent.enabled=true \
  --set datadog.orchestratorExplorer.enabled=true
```

```yaml
# datadog-values.yaml — comprehensive config
datadog:
  apiKey: <DD_API_KEY>
  site: datadoghq.com
  tags:
    - env:production
    - team:platform
    - cluster:prod-eks-1

  logs:
    enabled: true
    containerCollectAll: true
    autoMultiLineDetection: true

  apm:
    portEnabled: true           # Enable APM trace collection
    socketEnabled: true

  processAgent:
    enabled: true
    processCollection: true     # Collect process list

  systemProbe:
    enabled: true               # Network performance monitoring

  orchestratorExplorer:
    enabled: true               # K8s resource collection

  networkMonitoring:
    enabled: true

clusterAgent:
  enabled: true
  metricsProvider:
    enabled: true               # Datadog Metrics Provider for HPA

agents:
  tolerations:
    - operator: Exists           # Run on all nodes including tainted
```

---

### 🟡 Q13. How do you set up Datadog APM and distributed tracing?

```python
# Python APM setup
from ddtrace import patch_all, tracer
patch_all()                          # Auto-instrument all libraries

from ddtrace import Pin
import flask

app = flask.Flask(__name__)

@app.route('/api/orders/<order_id>')
def get_order(order_id):
    with tracer.trace('db.query', service='orders-db', resource='SELECT * FROM orders') as span:
        span.set_tag('order.id', order_id)
        result = db.query(f"SELECT * FROM orders WHERE id = {order_id}")
        span.set_tag('db.rows_returned', len(result))
    return jsonify(result)

# Environment variables for APM
# DD_ENV=production
# DD_SERVICE=my-api
# DD_VERSION=1.2.3
# DD_AGENT_HOST=datadog-agent.monitoring
# DD_TRACE_AGENT_PORT=8126
```

---

### 🟡 Q14. How do you create Datadog monitors and alerts?

```python
# Datadog Python API (monitors as code)
from datadog import initialize, api

initialize(api_key=DD_API_KEY, app_key=DD_APP_KEY)

# Create monitor
monitor = api.Monitor.create(
    type="metric alert",
    name="High Error Rate - Production API",
    message="""
    @slack-alerts-critical
    Error rate above 5% for 5 minutes.
    Dashboard: https://app.datadoghq.com/dashboard/xxx
    Runbook: https://wiki.example.com/runbooks/error-rate
    """,
    query="sum(last_5m):sum:trace.http.request.errors{env:production,service:my-api}.as_rate() / sum:trace.http.request.hits{env:production,service:my-api}.as_rate() > 0.05",
    options={
        "thresholds": {"critical": 0.05, "warning": 0.02},
        "notify_no_data": True,
        "no_data_timeframe": 10,
        "evaluation_delay": 60,
        "require_full_window": False,
        "renotify_interval": 60,
    },
    tags=["env:production", "team:backend"],
    priority=2
)
```

---

### 🟡 Q15. How do you configure Datadog synthetics and DogStatsD?

```python
# DogStatsD — emit custom metrics to Datadog
from datadog import statsd

# Counter
statsd.increment('orders.created', tags=['env:production', 'payment:stripe'])

# Gauge
statsd.gauge('cache.hit_rate', 0.87, tags=['cache:redis'])

# Histogram (distribution)
statsd.histogram('api.response_time_ms', 245.3, tags=['endpoint:/api/orders'])

# Timer context manager
with statsd.timed('db.query.duration', tags=['query:select_orders']):
    results = db.query("SELECT * FROM orders")

# Distribution (more accurate percentiles)
statsd.distribution('page.load.time', 1200, tags=['page:checkout'])
```

```python
# Datadog Synthetics API — create browser/API tests
test = api.Synthetics.create_test(
    type="api",
    name="Production API Health Check",
    message="@slack-alerts-critical",
    locations=["aws:us-east-1", "aws:eu-west-1"],
    options={
        "tick_every": 60,          # Every 60 seconds
        "min_failure_duration": 0,
        "min_location_failed": 1,
    },
    config={
        "request": {
            "method": "GET",
            "url": "https://api.example.com/health",
            "headers": {"Accept": "application/json"},
            "timeout": 30
        },
        "assertions": [
            {"type": "statusCode", "operator": "is", "target": 200},
            {"type": "responseTime", "operator": "lessThan", "target": 2000},
            {"type": "body", "operator": "contains", "target": '"status":"ok"'},
        ]
    }
)
```

---

## New Relic

### 🟢 Q16. What is New Relic and how do you install the agent?

```bash
# Install New Relic agent via Helm
helm repo add newrelic https://helm-charts.newrelic.com/charts
helm install newrelic-bundle newrelic/nri-bundle \
  --namespace newrelic \
  --create-namespace \
  --set global.licenseKey=$NEW_RELIC_LICENSE_KEY \
  --set global.cluster=production-eks \
  --set newrelic-infrastructure.enabled=true \
  --set nri-kube-events.enabled=true \
  --set kube-state-metrics.enabled=true \
  --set nri-prometheus.enabled=true \
  --set newrelic-logging.enabled=true \
  --set newrelic-pixie.enabled=false
```

---

### 🟡 Q17. How do you write NRQL queries in New Relic?

```sql
-- NRQL (New Relic Query Language) — SQL-like

-- Request rate
SELECT rate(count(*), 1 minute) 
FROM Transaction 
WHERE appName = 'my-api' 
SINCE 30 minutes AGO 
TIMESERIES

-- Error rate by endpoint
SELECT percentage(count(*), WHERE error IS TRUE) AS 'Error Rate'
FROM Transaction 
WHERE appName = 'my-api'
FACET request.uri 
SINCE 1 hour AGO 
LIMIT 20

-- P95 response time
SELECT percentile(duration, 95) AS 'P95 (ms)'
FROM Transaction 
WHERE appName = 'my-api'
TIMESERIES 5 minutes
SINCE 1 hour AGO

-- Slowest transactions
SELECT average(duration), count(*) 
FROM Transaction 
WHERE appName = 'my-api'
FACET name 
ORDER BY average(duration) DESC 
LIMIT 10

-- Infrastructure CPU
SELECT average(cpuPercent) AS 'CPU %'
FROM SystemSample 
FACET hostname 
TIMESERIES 10 minutes 
SINCE 1 hour AGO

-- Kubernetes pod restarts
SELECT sum(restartCount) 
FROM K8sPodSample 
FACET podName, namespaceName 
WHERE clusterName = 'production-eks'
SINCE 1 day AGO 
LIMIT 20

-- Custom events
SELECT count(*) FROM OrderCreated 
WHERE payment_method = 'stripe' 
FACET country 
SINCE 1 day AGO

-- New Relic distributed tracing
SELECT * FROM Span 
WHERE service.name = 'my-api' 
AND error IS TRUE 
SINCE 1 hour AGO 
LIMIT 100
```

---

### 🟡 Q18. How do you set up New Relic alerts?

```python
# New Relic alerts API
import requests

headers = {
    "X-Api-Key": NEW_RELIC_USER_KEY,
    "Content-Type": "application/json"
}

# Create alert condition (NRQL-based)
condition = requests.post(
    "https://api.newrelic.com/v2/alerts_nrql_conditions.json",
    headers=headers,
    json={
        "nrql_condition": {
            "type": "static",
            "name": "High Error Rate",
            "enabled": True,
            "nrql": {
                "query": "SELECT percentage(count(*), WHERE error IS TRUE) FROM Transaction WHERE appName = 'my-api'"
            },
            "terms": [
                {"threshold": "5", "operator": "above", "priority": "critical",
                 "time_function": "all", "duration": "5"},
                {"threshold": "2", "operator": "above", "priority": "warning",
                 "time_function": "all", "duration": "10"}
            ],
            "value_function": "single_value",
            "violation_time_limit_seconds": 3600,
            "expiration": {"expiration_duration": 300, "open_violation_on_expiration": False}
        }
    }
)
```

---

## FluentBit

### 🟢 Q19. What is FluentBit and how does it work?

```ini
# /etc/fluent-bit/fluent-bit.conf

[SERVICE]
    Flush         5
    Daemon        Off
    Log_Level     info
    HTTP_Server   On
    HTTP_Listen   0.0.0.0
    HTTP_Port     2020

[INPUT]
    Name              tail
    Path              /var/log/containers/*.log
    multiline.parser  docker, cri
    Tag               kube.*
    Refresh_Interval  5
    Mem_Buf_Limit     50MB
    Skip_Long_Lines   On
    DB                /var/log/flb_kube.db    # Checkpoint (track position)

[FILTER]
    Name                kubernetes
    Match               kube.*
    Kube_URL            https://kubernetes.default.svc:443
    Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
    Merge_Log           On
    Merge_Log_Key       log_processed
    K8S-Logging.Parser  On
    K8S-Logging.Exclude On

[FILTER]
    Name    grep
    Match   kube.*
    Exclude log /health|/ready|/metrics   # Drop health check logs

[OUTPUT]
    Name            loki
    Match           *
    Host            loki.monitoring.svc.cluster.local
    Port            3100
    Labels          job=fluentbit, cluster=production
    Label_Keys      $kubernetes['namespace_name'], $kubernetes['pod_name'], $kubernetes['labels']['app']
    Remove_Keys     kubernetes, stream, _p

[OUTPUT]
    Name            s3
    Match           *
    region          us-east-1
    bucket          my-logs-archive
    total_file_size 100M
    upload_timeout  10m
    store_dir       /tmp/fluent-bit-s3
```

---

### 🟡 Q20. How do you write FluentBit parsers and Lua filters?

```ini
# Custom parsers
[PARSER]
    Name        json_app
    Format      json
    Time_Key    timestamp
    Time_Format %Y-%m-%dT%H:%M:%S.%LZ

[PARSER]
    Name        apache_error
    Format      regex
    Regex       ^\[(?<time>[^\]]*)\] \[(?<level>[^\]]*)\] (?<message>.*)$
    Time_Key    time
    Time_Format %a %b %d %H:%M:%S %Y

[PARSER]
    Name        nginx_access
    Format      regex
    Regex       ^(?<remote>[^ ]*) (?<host>[^ ]*) (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")?$
    Time_Key    time
    Time_Format %d/%b/%Y:%H:%M:%S %z
```

```lua
-- Lua filter — complex log processing
function enrich_log(tag, timestamp, record)
    -- Add deployment environment
    record["environment"] = os.getenv("ENVIRONMENT") or "unknown"
    
    -- Parse error level from message
    if record["log"] then
        local log = record["log"]
        if string.match(log, "ERROR") or string.match(log, "CRITICAL") then
            record["severity"] = "error"
        elseif string.match(log, "WARN") then
            record["severity"] = "warning"
        else
            record["severity"] = "info"
        end
        
        -- Extract trace ID if present
        local trace_id = string.match(log, "traceId=([a-f0-9-]+)")
        if trace_id then
            record["trace_id"] = trace_id
        end
        
        -- Redact sensitive data
        record["log"] = string.gsub(log, '"password":"[^"]*"', '"password":"[REDACTED]"')
        record["log"] = string.gsub(record["log"], '"token":"[^"]*"', '"token":"[REDACTED]"')
    end
    
    return 1, timestamp, record  -- 1=modify, 0=keep, -1=drop
end
```

---

## FluentD

### 🟢 Q21. What is FluentD and how does it differ from FluentBit?

```
FluentBit:                          FluentD:
  Lightweight (450KB)                 Heavier (40MB+)
  C language                          Ruby language
  Less plugins (~100)                 1000+ plugins
  Lower memory/CPU                    Higher resource usage
  Designed for edge/DaemonSet         Designed for aggregator
  Kubernetes log shipper              Log processing hub

Typical architecture:
  FluentBit (DaemonSet per node) → FluentD (Deployment, aggregator) → S3/Elasticsearch/Loki
```

```xml
<!-- /etc/fluent/fluent.conf — FluentD config -->
<source>
  @type forward
  port 24224
  bind 0.0.0.0
  tag app.*
</source>

<filter kube.**>
  @type kubernetes_metadata
  @id filter_kube_metadata
  kubernetes_url "https://#{ENV['KUBERNETES_SERVICE_HOST']}:#{ENV['KUBERNETES_SERVICE_PORT_HTTPS']}"
  bearer_token_file /var/run/secrets/kubernetes.io/serviceaccount/token
  ca_file /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
</filter>

<filter app.**>
  @type record_transformer
  enable_ruby true
  <record>
    hostname "#{Socket.gethostname}"
    environment "#{ENV['ENVIRONMENT'] || 'unknown'}"
    @timestamp ${time.strftime('%Y-%m-%dT%H:%M:%S.%LZ')}
  </record>
</filter>

<!-- Route errors to different output -->
<match app.error>
  @type elasticsearch
  host elasticsearch.logging.svc
  port 9200
  index_name app-errors-%Y.%m.%d
  <buffer time>
    @type file
    path /var/log/fluent/buffer/errors
    timekey 1h
    timekey_wait 10m
    chunk_limit_size 256m
    total_limit_size 4g
    flush_interval 5s
    retry_max_interval 30s
  </buffer>
</match>

<match app.**>
  @type s3
  s3_bucket my-logs-archive
  s3_region us-east-1
  path logs/%Y/%m/%d/%H/
  <buffer time>
    @type file
    path /var/log/fluent/buffer/s3
    timekey 1h
    timekey_wait 10m
  </buffer>
</match>
```

---

## Vector

### 🟡 Q22. What is Vector and why is it popular?

```
Vector (by Datadog):
  - Modern observability pipeline tool (Rust-based → very fast)
  - Unifies log, metric, and trace collection
  - Replaces FluentBit + FluentD for many use cases
  - <strong>Much faster</strong> and more memory-efficient than FluentD
  - VRL (Vector Remap Language) for powerful transformations
  - Single binary handles collection, transformation, and routing
```

```yaml
# vector.yaml
api:
  enabled: true
  address: 0.0.0.0:8686

sources:
  kubernetes_logs:
    type: kubernetes_logs
    extra_label_selector: "app!=prometheus"  # Exclude prometheus pods

  host_metrics:
    type: host_metrics
    collectors: [cpu, disk, filesystem, load, memory, network]

  internal_metrics:
    type: internal_metrics

transforms:
  # Parse and enrich Kubernetes logs
  parse_logs:
    type: remap
    inputs: [kubernetes_logs]
    source: |
      # Parse JSON logs
      if is_string(.message) {
        structured, err = parse_json(.message)
        if err == null {
          ., err = merge(., structured)
          del(.message)
        }
      }
      
      # Add metadata
      .environment = get_env_var!("ENVIRONMENT")
      .cluster = "production-eks"
      
      # Redact sensitive fields
      if exists(.password) { .password = "[REDACTED]" }
      if exists(.token) { .token = "[REDACTED]" }
      
      # Parse log level
      if !exists(.level) && exists(.msg) {
        if contains(string!(.msg), "ERROR") {
          .level = "error"
        } else if contains(string!(.msg), "WARN") {
          .level = "warn"
        } else {
          .level = "info"
        }
      }
      
      # Drop health check logs
      if exists(.kubernetes.pod_labels.app) {
        if contains(string!(.path ?? ""), "/health") {
          abort
        }
      }

  # Route errors
  route_by_severity:
    type: route
    inputs: [parse_logs]
    route:
      errors: '.level == "error" || .level == "critical"'
      warnings: '.level == "warn"'
      info: '.level == "info"'

sinks:
  # All logs to Loki
  loki:
    type: loki
    inputs: [parse_logs]
    endpoint: http://loki.monitoring:3100
    labels:
      app: "{{ kubernetes.pod_labels.app }}"
      namespace: "{{ kubernetes.pod_namespace }}"
      level: "{{ level }}"

  # Error logs to S3 for long-term retention
  s3_errors:
    type: aws_s3
    inputs: [route_by_severity.errors]
    bucket: my-error-logs
    region: us-east-1
    key_prefix: "errors/{{ kubernetes.pod_namespace }}/{{ now() | strftime(\"%Y/%m/%d/\") }}"
    encoding:
      codec: json

  # Metrics to Prometheus
  prometheus:
    type: prometheus_remote_write
    inputs: [host_metrics, internal_metrics]
    endpoint: http://prometheus:9090/api/v1/write

  # Console for debugging
  console:
    type: console
    inputs: [parse_logs]
    encoding:
      codec: json
    target: stdout
```

---

## Log Pipeline Design

### 🔴 Q23. How do you design a production-grade log pipeline?

```
Architecture options:

Option 1: FluentBit → Loki (Simple, Kubernetes-native)
  Pods → FluentBit DaemonSet → Loki → Grafana
  Good for: Small/medium clusters, cost-conscious

Option 2: FluentBit → FluentD → Multiple sinks
  Pods → FluentBit (per node) → FluentD (aggregator) → S3, Elasticsearch, Splunk
  Good for: Complex routing, multiple backends, enterprise compliance

Option 3: Vector (All-in-one)
  Pods → Vector DaemonSet → Loki + S3 + Datadog
  Good for: Modern setups, performance-critical

Option 4: OTel Collector → Multiple backends
  Pods (OTel SDK) → OTel Collector → Loki, Tempo, Prometheus, Datadog
  Good for: Unified observability (logs + traces + metrics), vendor portability

Production requirements to address:
  1. Backpressure — buffer when downstream is slow
  2. Retry with exponential backoff — handle outages
  3. Deduplication — don't send duplicate logs on restart
  4. Ordering — preserve log order per pod
  5. Security — TLS between components
  6. Filtering — drop high-volume health checks
  7. Redaction — remove PII/secrets before storage
  8. Routing — different retention per log type (errors 1yr, debug 7d)
  9. Multi-tenancy — separate team logs
  10. Compression — gzip before sending to save bandwidth
```

---

## Master Cheatsheet

### PromQL Quick Reference
```promql
rate(metric[5m])                   # Per-second rate
increase(metric[1h])               # Total increase
histogram_quantile(0.99, sum by(le)(rate(hist_bucket[5m])))
sum by (label) (metric)
avg without (instance) (metric)
topk(5, metric)
```

### LogQL Quick Reference
```logql
{app="x"}                          # Select stream
{app="x"} |= "error"              # Filter
{app="x"} | json | level="error"  # Parse + filter
rate({app="x"}[5m])               # Rate metric
sum(rate({app="x"} |= "error" [5m]))
```

### Question Coverage Index
| Q# | Topic | Difficulty |
|---|---|---|
| Q1 | Prometheus architecture | 🟢 |
| Q2 | Metric types | 🟢 |
| Q3 | PromQL queries | 🟡 |
| Q4 | Alerting rules | 🟡 |
| Q5 | kube-prometheus-stack deploy | 🟡 |
| Q6 | Prometheus Operator CRDs | 🟡 |
| Q7 | Alertmanager routing, inhibition, grouping | 🟡 |
| Q8 | Grafana data sources & dashboards | 🟢 |
| Q9 | Grafana Loki + LogQL | 🟡 |
| Q10 | Grafana Tempo (distributed traces) | 🟡 |
| Q11 | OpenTelemetry (OTel) | 🟡 |
| Q12 | Datadog agent & Kubernetes | 🟢 |
| Q13 | Datadog APM & tracing | 🟡 |
| Q14 | Datadog monitors | 🟡 |
| Q15 | Datadog Synthetics & DogStatsD | 🟡 |
| Q16 | New Relic agent | 🟢 |
| Q17 | NRQL queries | 🟡 |
| Q18 | New Relic alerts | 🟡 |
| Q19 | FluentBit architecture & config | 🟢 |
| Q20 | FluentBit parsers & Lua filters | 🟡 |
| Q21 | FluentD vs FluentBit | 🟢 |
| Q22 | Vector (modern log pipeline) | 🟡 |
| Q23 | Production log pipeline design | 🔴 |

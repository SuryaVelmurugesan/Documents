# Phase 13: Observability — Metrics, Dashboards, and Logs

> **Goal**: You can't fix what you can't see. Build a full observability stack — Prometheus for metrics, Grafana for dashboards, Loki for logs — so you know exactly what's happening inside your cluster at all times.

---

## 13.1 — The Three Pillars of Observability

| Pillar | Question It Answers | Tool |
|---|---|---|
| **Metrics** | "How is my system performing right now? What trends do I see?" | **Prometheus** |
| **Logs** | "What happened? What errors occurred? What was the sequence of events?" | **Loki** (or EFK stack) |
| **Traces** | "How did a single request flow through my distributed system? Where was the bottleneck?" | **Jaeger** / **Tempo** |

```
         ┌──────────────────────────────────────────┐
         │            Observability Stack            │
         │                                          │
         │   Metrics ──► Prometheus ──► Grafana      │
         │                                          │
         │   Logs ──► Promtail ──► Loki ──► Grafana  │
         │                                          │
         │   Traces ──► App SDK ──► Tempo ──► Grafana │
         │                                          │
         │          Single pane of glass: GRAFANA    │
         └──────────────────────────────────────────┘
```

Grafana is the unified UI for all three pillars.

---

## 13.2 — Prometheus: The Metrics Engine

Prometheus is the de facto standard for Kubernetes metrics. It's a CNCF graduated project (same maturity level as Kubernetes itself).

### How Prometheus Works: The Pull Model

Unlike traditional monitoring (agents push metrics to a server), Prometheus **pulls** (scrapes) metrics from targets:

```
Prometheus Server
      │
      │──── scrape every 15s ──► Pod A   /metrics  (HTTP endpoint)
      │──── scrape every 15s ──► Pod B   /metrics
      │──── scrape every 15s ──► Pod C   /metrics
      │──── scrape every 15s ──► kubelet /metrics
      │──── scrape every 15s ──► Node Exporter /metrics
      │
      ▼
 Time-Series Database (TSDB)
      │
      ▼
 PromQL queries ──► Grafana dashboards
                ──► Alertmanager ──► Slack/PagerDuty/Email
```

### What a /metrics Endpoint Looks Like

Every application that Prometheus monitors exposes an HTTP endpoint (typically `/metrics`) in a specific text format:

```
# HELP http_requests_total Total number of HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET", status="200", path="/api/users"} 15234
http_requests_total{method="POST", status="201", path="/api/orders"} 892
http_requests_total{method="GET", status="500", path="/api/users"} 12

# HELP http_request_duration_seconds HTTP request latency
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.01"} 9800
http_request_duration_seconds_bucket{le="0.05"} 14500
http_request_duration_seconds_bucket{le="0.1"} 15000
http_request_duration_seconds_bucket{le="+Inf"} 15234
http_request_duration_seconds_sum 245.67
http_request_duration_seconds_count 15234

# HELP process_resident_memory_bytes Resident memory size in bytes
# TYPE process_resident_memory_bytes gauge
process_resident_memory_bytes 52428800
```

### Metric Types

| Type | What It Represents | Example |
|---|---|---|
| **Counter** | A value that only goes up (resets on restart) | Total HTTP requests, total errors |
| **Gauge** | A value that can go up or down | Current memory usage, active connections, temperature |
| **Histogram** | Observations bucketed by value (for latency distributions) | Request duration (how many requests took <10ms, <50ms, <100ms) |
| **Summary** | Similar to histogram, calculates percentiles client-side | Request duration p50, p90, p99 |

### Prometheus Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    Prometheus Server                      │
│                                                          │
│  ┌──────────┐   ┌──────────┐   ┌──────────────────────┐ │
│  │ Retrieval │   │   TSDB   │   │     HTTP Server      │ │
│  │ (Scraper) │──►│ (Storage)│──►│  (PromQL API + UI)   │ │
│  └──────────┘   └──────────┘   └──────────────────────┘ │
│       ▲                                    │              │
│       │                                    ▼              │
│  Service Discovery              Alertmanager              │
│  (K8s API, DNS, file)           (Routes, Silences)        │
└──────────────────────────────────────────────────────────┘
```

| Component | Purpose |
|---|---|
| **Retrieval** | Discovers targets and scrapes them at configured intervals |
| **TSDB** | Stores time-series data locally on disk (default 15-day retention) |
| **HTTP Server** | Serves the PromQL API (queried by Grafana) and a basic web UI |
| **Service Discovery** | Automatically finds scrape targets via Kubernetes API, DNS, or static config |
| **Alertmanager** | Receives alerts from Prometheus, deduplicates, groups, and routes to notification channels |

---

## 13.3 — PromQL: Querying Metrics

PromQL (Prometheus Query Language) is how you query the time-series database.

### Basic Queries

```promql
# Instant vector — current value of a metric
http_requests_total

# Filter by label
http_requests_total{method="GET", status="200"}

# Regex match
http_requests_total{path=~"/api/.*"}

# Negative match
http_requests_total{status!="200"}
```

### Range Vectors and Functions

```promql
# Rate: per-second average over the last 5 minutes (for counters)
rate(http_requests_total[5m])

# Increase: total increase over the last 1 hour
increase(http_requests_total[1h])

# Average across all pods
avg(rate(http_requests_total[5m]))

# Average grouped by status code
avg by (status) (rate(http_requests_total[5m]))

# Sum of all requests per second
sum(rate(http_requests_total[5m]))

# 95th percentile latency (from histogram)
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

### Kubernetes-Specific Queries

```promql
# CPU usage per pod (percentage of requested CPU)
sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="production"}[5m]))
/
sum by (pod) (kube_pod_container_resource_requests{namespace="production", resource="cpu"})
* 100

# Memory usage per pod (bytes)
container_memory_working_set_bytes{namespace="production", container!=""}

# Pod restart count
kube_pod_container_status_restarts_total{namespace="production"}

# Number of pods by phase (Running, Pending, Failed)
count by (phase) (kube_pod_status_phase{namespace="production"} == 1)

# Node CPU utilization
1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))

# Node memory utilization
1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

# Persistent volume usage percentage
kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes * 100
```

> [!TIP]
> **The most useful PromQL pattern for counters**: Always use `rate()` or `increase()` on counters, never the raw value. A counter's raw value is just a number that goes up — `rate()` converts it to "events per second," which is actually meaningful.

---

## 13.4 — Prometheus Operator & ServiceMonitors

In Phase 11, we learned about operators. The **Prometheus Operator** manages the Prometheus deployment and provides CRDs for configuring monitoring declaratively.

### Key CRDs

| CRD | Purpose |
|---|---|
| `Prometheus` | Defines a Prometheus server instance (replicas, retention, storage, version) |
| `ServiceMonitor` | Tells Prometheus which Services to scrape (replaces manual scrape config) |
| `PodMonitor` | Scrapes pods directly (when there's no Service) |
| `PrometheusRule` | Alert rules and recording rules |
| `AlertmanagerConfig` | Alertmanager routing and notification config |

### ServiceMonitor Example

Instead of editing `prometheus.yml` to add scrape targets, create a ServiceMonitor:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: webapp-monitor
  namespace: production
  labels:
    release: kube-prometheus       # Must match Prometheus selector
spec:
  selector:
    matchLabels:
      app: webapp                  # Match Services with this label
  namespaceSelector:
    matchNames:
      - production
  endpoints:
    - port: metrics                # Port name from the Service spec
      interval: 15s               # Scrape every 15 seconds
      path: /metrics              # Metrics endpoint path
```

```
ServiceMonitor ──selects──► Service ──selects──► Pods (scraped by Prometheus)
```

### PodMonitor (When There's No Service)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: batch-jobs-monitor
spec:
  selector:
    matchLabels:
      app: batch-processor
  podMetricsEndpoints:
    - port: metrics
      interval: 30s
```

---

## 13.5 — Alerting with Alertmanager

### Defining Alert Rules (PrometheusRule CRD)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: webapp-alerts
  labels:
    release: kube-prometheus
spec:
  groups:
    - name: webapp.rules
      rules:
        # Alert: High error rate
        - alert: HighErrorRate
          expr: |
            sum(rate(http_requests_total{status=~"5.."}[5m]))
            /
            sum(rate(http_requests_total[5m]))
            > 0.05
          for: 5m                             # Must be true for 5 minutes
          labels:
            severity: critical
          annotations:
            summary: "High error rate detected"
            description: "Error rate is {{ $value | humanizePercentage }} (>5%) for the last 5 minutes"

        # Alert: Pod CrashLooping
        - alert: PodCrashLooping
          expr: rate(kube_pod_container_status_restarts_total[15m]) * 60 * 15 > 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} is crash looping"
            description: "Pod {{ $labels.pod }} in namespace {{ $labels.namespace }} is restarting frequently"

        # Alert: High memory usage
        - alert: HighMemoryUsage
          expr: |
            container_memory_working_set_bytes{container!=""}
            /
            kube_pod_container_resource_limits{resource="memory"}
            > 0.90
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} memory usage >90%"

        # Alert: PVC almost full
        - alert: PVCAlmostFull
          expr: |
            kubelet_volume_stats_used_bytes
            / kubelet_volume_stats_capacity_bytes
            > 0.85
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "PVC {{ $labels.persistentvolumeclaim }} is >85% full"
```

### Alertmanager Routing

```yaml
# Alertmanager configuration (in the Helm values or AlertmanagerConfig CRD)
route:
  receiver: default-slack
  group_by: [alertname, namespace]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match:
        severity: critical
      receiver: pagerduty-critical
    - match:
        severity: warning
      receiver: slack-warnings

receivers:
  - name: default-slack
    slack_configs:
      - api_url: https://hooks.slack.com/services/XXX/YYY/ZZZ
        channel: '#k8s-alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: pagerduty-critical
    pagerduty_configs:
      - service_key: <your-pagerduty-key>

  - name: slack-warnings
    slack_configs:
      - api_url: https://hooks.slack.com/services/XXX/YYY/ZZZ
        channel: '#k8s-warnings'
```

---

## 13.6 — Grafana: Dashboards and Visualization

Grafana connects to Prometheus (and Loki, Tempo) as data sources and provides rich, interactive dashboards.

### Key Grafana Concepts

| Concept | Purpose |
|---|---|
| **Data Source** | Backend to query (Prometheus, Loki, Tempo, InfluxDB, etc.) |
| **Dashboard** | A collection of panels displaying metrics |
| **Panel** | A single graph, gauge, table, or stat widget |
| **Variables** | Dropdown filters (select namespace, pod, node) that apply to all panels |
| **Alerts** | Grafana can also fire alerts (separate from Alertmanager) |

### Essential Kubernetes Dashboards

The `kube-prometheus-stack` ships with pre-built dashboards. The most important ones:

| Dashboard | What It Shows |
|---|---|
| **Kubernetes / Compute Resources / Namespace (Pods)** | CPU/memory usage per pod in a namespace |
| **Kubernetes / Compute Resources / Node** | Node-level CPU, memory, disk, network |
| **Kubernetes / Compute Resources / Cluster** | Cluster-wide resource utilization |
| **Kubernetes / Networking / Namespace (Pods)** | Network I/O per pod |
| **CoreDNS** | DNS query rates, latency, errors |
| **etcd** | etcd health, leader changes, disk I/O, proposal rates |
| **API Server** | Request rates, latency, error rates by verb/resource |

### Importing Community Dashboards

Grafana has thousands of community dashboards at [grafana.com/grafana/dashboards](https://grafana.com/grafana/dashboards/). Some essential ones:

| Dashboard ID | Name | Purpose |
|---|---|---|
| **3119** | Kubernetes Cluster Monitoring | Cluster-wide overview |
| **6417** | Kubernetes Pods Monitoring | Per-pod deep dive |
| **1860** | Node Exporter Full | Detailed node hardware metrics |
| **15661** | K8s Container Logs (via Loki) | Log panel integrated with metrics |

```
In Grafana UI: Dashboards → Import → Enter dashboard ID → Select Prometheus data source → Import
```

---

## 13.7 — Loki: Log Aggregation

Loki is a horizontally-scalable, cost-effective log aggregation system designed by Grafana Labs. It's like "Prometheus, but for logs."

### How Loki Works

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  Pod A   │     │  Pod B   │     │  Pod C   │     │  Node    │
│ (stdout) │     │ (stdout) │     │ (stdout) │     │ (syslog) │
└────┬─────┘     └────┬─────┘     └────┬─────┘     └────┬─────┘
     │                │                │                 │
     └────────────────┴────────────────┴─────────────────┘
                              │
                         ┌────┴────┐
                         │Promtail │   ← Agent on each node (DaemonSet)
                         │(Collect)│     Tails container logs, adds labels
                         └────┬────┘
                              │
                         ┌────┴────┐
                         │  Loki   │   ← Stores logs, indexes by labels only
                         │ (Store) │     (NOT full-text indexing like Elasticsearch)
                         └────┬────┘
                              │
                         ┌────┴────┐
                         │ Grafana │   ← Query and visualize logs
                         │ (Query) │
                         └─────────┘
```

### Loki vs Elasticsearch (EFK)

| Feature | Loki | Elasticsearch (EFK) |
|---|---|---|
| **Indexing** | Labels only (lightweight) | Full-text indexing (heavy) |
| **Resource usage** | Low (10x less than ES) | High (CPU, memory, disk hungry) |
| **Query language** | LogQL | KQL / Lucene |
| **Cost** | Low | High |
| **Best for** | Kubernetes-native, label-based log searching | Full-text search, complex log analysis |
| **Integration** | Native Grafana | Kibana |

### LogQL: Querying Logs

```logql
# All logs from a specific namespace
{namespace="production"}

# Logs from a specific pod
{pod="webapp-abc123"}

# Logs from a specific container
{namespace="production", container="nginx"}

# Filter log lines containing "error" (case insensitive)
{namespace="production"} |~ "(?i)error"

# Filter log lines NOT containing "healthcheck"
{namespace="production"} !~ "healthcheck"

# Parse JSON logs and filter
{namespace="production"} | json | level="error"

# Parse and extract fields from structured logs
{namespace="production"} | json | status_code >= 500

# Count errors per pod over time
count_over_time({namespace="production"} |~ "error" [5m])

# Top 5 pods by log volume
topk(5, sum by (pod) (rate({namespace="production"}[5m])))
```

> [!TIP]
> **Loki + Grafana killer feature**: In Grafana, you can click on a metric spike in a Prometheus graph and instantly jump to the corresponding logs in Loki for that exact time range and pod. This metrics-to-logs correlation is incredibly powerful for debugging.

---

## 13.8 — The kube-prometheus-stack: One Helm Chart to Rule Them All

The easiest way to deploy the full observability stack is the **kube-prometheus-stack** Helm chart. It installs:

- Prometheus (server + operator)
- Alertmanager
- Grafana (with pre-configured dashboards and data sources)
- Node Exporter (hardware metrics from each node)
- kube-state-metrics (Kubernetes object metrics)
- Pre-built alerting rules
- Pre-built dashboards

### Installation

```powershell
# Add the Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install the full stack
helm install monitoring prometheus-community/kube-prometheus-stack `
  --namespace monitoring `
  --create-namespace `
  --set grafana.adminPassword=admin123
```

### Adding Loki

```powershell
# Add Grafana Helm repo (Loki is maintained by Grafana Labs)
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Loki + Promtail
helm install loki grafana/loki-stack `
  --namespace monitoring `
  --set promtail.enabled=true `
  --set loki.persistence.enabled=true `
  --set loki.persistence.size=10Gi
```

### Accessing the Dashboards

```powershell
# Grafana
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80
# http://localhost:3000 — admin / admin123

# Prometheus UI
kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090
# http://localhost:9090

# Alertmanager UI
kubectl port-forward svc/monitoring-kube-prometheus-alertmanager -n monitoring 9093:9093
# http://localhost:9093
```

### Verifying Components

```powershell
# List all pods in the monitoring namespace
kubectl get pods -n monitoring
# Expected:
# monitoring-grafana-xxx
# monitoring-kube-prometheus-prometheus-0
# monitoring-kube-prometheus-alertmanager-0
# monitoring-kube-prometheus-operator-xxx
# monitoring-kube-state-metrics-xxx
# monitoring-prometheus-node-exporter-xxx (one per node)
# loki-0
# loki-promtail-xxx (one per node)

# Check ServiceMonitors
kubectl get servicemonitors -n monitoring

# Check PrometheusRules (pre-installed alerts)
kubectl get prometheusrules -n monitoring
```

---

## 13.9 — Key Metrics to Monitor

### The Four Golden Signals (Google SRE)

| Signal | What to Measure | PromQL Example |
|---|---|---|
| **Latency** | How long requests take | `histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))` |
| **Traffic** | How much demand is hitting your system | `sum(rate(http_requests_total[5m]))` |
| **Errors** | Rate of failed requests | `sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))` |
| **Saturation** | How "full" your resources are | `container_memory_working_set_bytes / kube_pod_container_resource_limits{resource="memory"}` |

### The RED Method (For Services)

| Signal | Metric |
|---|---|
| **R**ate | Requests per second |
| **E**rrors | Errors per second |
| **D**uration | Request latency distribution |

### The USE Method (For Infrastructure)

| Signal | Metric |
|---|---|
| **U**tilization | % of resource in use (CPU, memory, disk) |
| **S**aturation | Work queued (waiting for resources) |
| **E**rrors | Error counts (disk I/O errors, network drops) |

---

## 13.10 — Distributed Tracing (Overview)

For microservices architectures, tracing follows a single request across multiple services:

```
User Request → API Gateway → Auth Service → Order Service → Payment Service → Database
                    │              │              │               │
                    ▼              ▼              ▼               ▼
              Trace Span 1   Span 2          Span 3          Span 4
                    └──────────────┴──────────────┴───────────────┘
                                     One Trace
```

### Tools

| Tool | Type | Integration |
|---|---|---|
| **Jaeger** | CNCF, standalone | OpenTelemetry SDK in your app |
| **Tempo** | Grafana Labs, Grafana-native | OpenTelemetry SDK, native Grafana datasource |
| **Zipkin** | Older, still used | Zipkin instrumentation |

### OpenTelemetry (OTel)

The standard for instrumenting applications. Provides SDKs for every language:

```
Your App + OTel SDK → OTel Collector → Tempo/Jaeger
                                      → Prometheus (metrics)
                                      → Loki (logs)
```

> [!NOTE]
> Setting up distributed tracing requires instrumenting your application code (adding the OpenTelemetry SDK). It's application-specific, not infrastructure-only like Prometheus and Loki. For this curriculum, understanding the concept and architecture is sufficient.

---

## 13.11 — Monitoring Your Monitoring

```promql
# Is Prometheus itself healthy?
up{job="prometheus"}

# Prometheus scrape duration (is scraping too slow?)
prometheus_target_interval_length_seconds

# How many targets is Prometheus scraping?
count(up)

# How many targets are down?
count(up == 0)

# TSDB storage size
prometheus_tsdb_storage_size_bytes

# Alertmanager notification failures
alertmanager_notifications_failed_total
```

---

## Phase 13 — Cheat Sheet

| Command | Purpose |
|---|---|
| `helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace` | Install full monitoring stack |
| `helm install loki grafana/loki-stack -n monitoring` | Install Loki + Promtail |
| `kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80` | Access Grafana |
| `kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090` | Access Prometheus UI |
| `kubectl get servicemonitors -n monitoring` | List scrape targets |
| `kubectl get prometheusrules -n monitoring` | List alert rules |
| `kubectl get pods -n monitoring` | Verify monitoring stack health |
| `kubectl top pods` | Quick resource usage (metrics-server) |
| `kubectl top nodes` | Node resource usage |
| `kubectl logs <pod> -n monitoring` | Debug monitoring components |

---

## 🔬 Lab Exercise 13: Deploy a Full Observability Stack

### Objective

Install Prometheus, Grafana, and Loki on Minikube. Deploy a sample app, create a ServiceMonitor, build a dashboard, query logs, and set up an alert.

### Step 1: Install the Monitoring Stack

```powershell
# Add repos
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack
helm install monitoring prometheus-community/kube-prometheus-stack `
  --namespace monitoring --create-namespace `
  --set grafana.adminPassword=lab13pass `
  --set prometheus.prometheusSpec.retention=6h

# Install Loki
helm install loki grafana/loki-stack `
  --namespace monitoring `
  --set promtail.enabled=true

# Wait for all pods
kubectl get pods -n monitoring -w
```

### Step 2: Access Grafana

```powershell
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80
# http://localhost:3000 — admin / lab13pass
```

### Step 3: Deploy a Sample App With Metrics

Create `lab13-app.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lab13-app
  namespace: default
  labels:
    app: lab13-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: lab13-app
  template:
    metadata:
      labels:
        app: lab13-app
    spec:
      containers:
        - name: web
          image: nginx:1.27
          ports:
            - name: http
              containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: lab13-app-svc
  labels:
    app: lab13-app
spec:
  selector:
    app: lab13-app
  ports:
    - name: http
      port: 80
      targetPort: 80
```

```powershell
kubectl apply -f lab13-app.yaml
```

### Step 4: Explore Pre-Built Dashboards

In Grafana:
1. Go to **Dashboards → Browse**
2. Open **"Kubernetes / Compute Resources / Namespace (Pods)"**
3. Select namespace `default`
4. Observe CPU and memory graphs for your `lab13-app` pods

### Step 5: Query Prometheus

```powershell
kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090
```

Open `http://localhost:9090` and run:

```promql
# CPU usage of your app
sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="default", pod=~"lab13-app.*"}[5m]))

# Memory usage
container_memory_working_set_bytes{namespace="default", pod=~"lab13-app.*", container="web"}

# Pod restart count
kube_pod_container_status_restarts_total{namespace="default"}
```

### Step 6: Query Logs in Grafana (via Loki)

In Grafana:
1. Go to **Explore** (compass icon)
2. Select **Loki** as the data source
3. Run:
   ```logql
   {namespace="default", pod=~"lab13-app.*"}
   ```
4. Generate some logs:
   ```powershell
   kubectl exec lab13-app-xxx -- curl -s localhost
   kubectl exec lab13-app-xxx -- curl -s localhost/nonexistent
   ```
5. Filter for errors:
   ```logql
   {namespace="default", pod=~"lab13-app.*"} |~ "404"
   ```

### Step 7: Create a Custom Alert

Create `lab13-alert.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: lab13-alerts
  namespace: monitoring
  labels:
    release: monitoring            # Must match the Prometheus operator selector
spec:
  groups:
    - name: lab13.rules
      rules:
        - alert: Lab13HighMemory
          expr: |
            container_memory_working_set_bytes{namespace="default", pod=~"lab13-app.*", container="web"}
            > 50 * 1024 * 1024
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: "Lab13 pod {{ $labels.pod }} memory >50Mi"
```

```powershell
kubectl apply -f lab13-alert.yaml

# Verify the rule was picked up
kubectl get prometheusrules -n monitoring
```

Check in Prometheus UI → **Alerts** tab to see your custom alert.

### What to Submit

1. Output of `kubectl get pods -n monitoring` (showing all monitoring components running)
2. A PromQL query result screenshot or output showing your app's CPU usage
3. A LogQL query result showing logs from your app
4. Output of `kubectl get prometheusrules -n monitoring` showing your custom alert

---

> **⏸️ CHECKPOINT**: Submit your lab results or request to skip. Final phase: **Phase 14: CKA Certification Prep** — exam strategy, key topics, and practice exercises.

# Phase 9: Horizontal Pod Autoscaler (HPA)

> **Goal**: Stop guessing replica counts. Let Kubernetes automatically scale your workloads up and down based on real-time metrics — CPU utilization, memory consumption, or custom application metrics.

---

## 9.1 — Why Manual Scaling Fails

In Phase 2, you scaled Deployments with `kubectl scale --replicas=N`. The problem:

```
Night:     2 replicas   → Wasting money (low traffic, over-provisioned)
Morning:   2 replicas   → Users see slow responses (traffic ramping up)
You scale: 10 replicas  → OK for now
Lunch:     10 replicas  → Wasting money again (traffic dipped)
Afternoon: 10 replicas  → Spike hits, still not enough
You scale: 20 replicas  → Playing catch-up
```

You're always **reactive** — scaling after users feel the pain. The HPA makes you **proactive** — scaling based on metrics before users notice.

```
HPA watches CPU → CPU > 70% → add pods → CPU drops → remove pods
          ↕
     Fully automatic, continuous, no human intervention
```

---

## 9.2 — Prerequisites: Metrics Server

The HPA needs metrics to make decisions. It reads metrics from the **Metrics Server** — a cluster add-on that collects CPU and memory usage from kubelets.

```powershell
# Enable on Minikube
minikube addons enable metrics-server

# Verify it's running
kubectl get pods -n kube-system -l k8s-app=metrics-server

# Wait ~60 seconds, then test
kubectl top nodes
kubectl top pods
```

> [!IMPORTANT]
> Without metrics-server, the HPA will show `<unknown>` for current metrics and will never scale. Always verify `kubectl top pods` works before setting up HPAs.

### The Metrics Pipeline

```
kubelet (cAdvisor)  →  Metrics Server  →  metrics.k8s.io API  →  HPA Controller
                                                                      │
                                                              Reads every 15s (default)
                                                                      │
                                                              Computes desired replicas
                                                                      │
                                                              Scales the Deployment
```

---

## 9.3 — HPA Architecture: The Control Loop

The HPA controller runs a continuous loop:

```
1. Query current metric value (e.g., avg CPU across all pods = 85%)
2. Compare against target value (e.g., target = 50%)
3. Compute desired replicas:
     desiredReplicas = ceil( currentReplicas × (currentMetricValue / desiredMetricValue) )
     desiredReplicas = ceil( 3 × (85 / 50) ) = ceil(5.1) = 6
4. Scale the Deployment to 6 replicas
5. Wait for stabilization
6. Repeat
```

### The Scaling Formula

```
desiredReplicas = ⌈ currentReplicas × (currentMetricValue / targetValue) ⌉
```

Examples:
| Current Replicas | Current CPU | Target CPU | Desired Replicas | Action |
|---|---|---|---|---|
| 3 | 85% | 50% | ⌈3 × 1.70⌉ = 6 | Scale up to 6 |
| 6 | 30% | 50% | ⌈6 × 0.60⌉ = 4 | Scale down to 4 |
| 4 | 48% | 50% | ⌈4 × 0.96⌉ = 4 | No change (within tolerance) |

> [!NOTE]
> The HPA has a **tolerance band** (default ±10%). If the ratio is between 0.9 and 1.1, no scaling happens. This prevents flapping (constant up/down cycling).

---

## 9.4 — HPA v2 Manifest (The Current Standard)

```yaml
apiVersion: autoscaling/v2            # v2 is the current stable API
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment                   # What to scale
    name: webapp                       # Deployment name

  minReplicas: 2                       # Never go below 2
  maxReplicas: 20                      # Never go above 20

  metrics:
    - type: Resource                   # Built-in resource metric
      resource:
        name: cpu
        target:
          type: Utilization            # Percentage of requested CPU
          averageUtilization: 50       # Target: 50% average across all pods
```

### Key Fields

| Field | Purpose |
|---|---|
| `scaleTargetRef` | The resource to scale (Deployment, StatefulSet, ReplicaSet) |
| `minReplicas` | Floor — HPA will never scale below this |
| `maxReplicas` | Ceiling — HPA will never scale above this |
| `metrics` | List of metrics to evaluate. If multiple are specified, the HPA computes desired replicas for each and takes the **highest** value |

> [!IMPORTANT]
> **Your Pods MUST have resource requests defined** for CPU/memory-based HPA to work. The HPA calculates utilization as a percentage of the *request*, not the *limit*. No request = no percentage = HPA shows `<unknown>`.
>
> ```yaml
> resources:
>   requests:
>     cpu: 200m      # ← HPA needs this to calculate "50% of 200m = 100m"
> ```

---

## 9.5 — CPU-Based Autoscaling (Most Common)

### Quick Setup (Imperative)

```powershell
# Create an HPA that targets 50% CPU for a Deployment
kubectl autoscale deployment webapp --cpu-percent=50 --min=2 --max=10
```

### Declarative (Preferred)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

### Monitoring the HPA

```powershell
# View HPA status
kubectl get hpa
# NAME         REFERENCE           TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
# webapp-hpa   Deployment/webapp   23%/50%   2         10        2          5m

# Detailed view with conditions
kubectl describe hpa webapp-hpa

# Watch in real-time
kubectl get hpa -w
```

The `TARGETS` column shows `currentValue/targetValue`. When current exceeds target, scaling begins.

---

## 9.6 — Memory-Based Autoscaling

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70        # Scale when avg memory > 70% of request

    # You can also use absolute values:
    # - type: Resource
    #   resource:
    #     name: memory
    #     target:
    #       type: AverageValue
    #       averageValue: 500Mi         # Scale when avg memory > 500Mi per pod
```

> [!WARNING]
> **Memory-based HPA is tricky.** Unlike CPU (which naturally fluctuates with load), memory usage in many applications only grows — it doesn't drop when load decreases (JVM heap, in-memory caches, connection pools). This means the HPA scales up but **never scales back down**. Use memory HPA only for workloads where memory usage truly correlates with load.

---

## 9.7 — Multi-Metric HPA

Combine multiple metrics. The HPA evaluates each and uses the **maximum** desired replica count:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
    # Scale on CPU
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50

    # AND scale on memory
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
```

If CPU says "need 5 pods" and memory says "need 8 pods," the HPA scales to 8.

---

## 9.8 — Custom and External Metrics

Beyond CPU/memory, you can scale on application-specific metrics (requests per second, queue depth, active connections).

### Metric Types

| Type | Source | Example |
|---|---|---|
| `Resource` | Metrics Server (CPU/memory) | CPU at 50% |
| `Pods` | Custom metrics adapter (per-pod metric) | Requests per second per pod |
| `Object` | Custom metrics adapter (Kubernetes object metric) | Ingress requests per second |
| `External` | External metrics adapter (outside K8s) | SQS queue depth, Pub/Sub message count |

### Custom Metric Example (Requests Per Second)

Requires a custom metrics adapter (e.g., **Prometheus Adapter**):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 2
  maxReplicas: 50
  metrics:
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second     # Custom metric from Prometheus
        target:
          type: AverageValue
          averageValue: "100"                # Scale when avg RPS > 100 per pod
```

### External Metric Example (AWS SQS Queue Depth)

Requires an external metrics provider (e.g., **KEDA**, **CloudWatch adapter**):

```yaml
metrics:
  - type: External
    external:
      metric:
        name: sqs_queue_length
        selector:
          matchLabels:
            queue: "orders-queue"
      target:
        type: AverageValue
        averageValue: "30"                   # Scale when avg queue depth > 30
```

> [!NOTE]
> Custom/external metrics require additional components (Prometheus + Prometheus Adapter, or KEDA). We'll cover KEDA briefly in section 9.11. For the lab, we'll stick with CPU-based scaling.

---

## 9.9 — Scaling Behavior: Fine-Tuning the Speed

By default, the HPA scales up quickly and scales down slowly (to prevent flapping). You can customize this:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30        # Wait 30s of sustained high load before scaling up
      policies:
        - type: Percent
          value: 100                        # Can double the replica count per interval
          periodSeconds: 60                 # Per 60-second window
        - type: Pods
          value: 5                          # Or add max 5 pods per interval
          periodSeconds: 60
      selectPolicy: Max                     # Use whichever policy allows MORE scaling

    scaleDown:
      stabilizationWindowSeconds: 300       # Wait 5 minutes of sustained low load before scaling down
      policies:
        - type: Percent
          value: 10                         # Remove max 10% of pods per interval
          periodSeconds: 60
      selectPolicy: Min                     # Use whichever policy allows LESS scaling (conservative)
```

### Behavior Parameters Explained

| Parameter | Purpose | Default |
|---|---|---|
| `stabilizationWindowSeconds` | How long to wait after the last scaling event before scaling again | Scale up: 0s, Scale down: 300s (5 min) |
| `policies[].type` | `Pods` (absolute count) or `Percent` (percentage of current replicas) | — |
| `policies[].value` | The amount (pods or percentage) | — |
| `policies[].periodSeconds` | Time window for the policy | — |
| `selectPolicy` | `Max` (most aggressive), `Min` (most conservative), `Disabled` (no scaling in this direction) | `Max` for up, `Max` for down |

### Disable Scale-Down Entirely

```yaml
behavior:
  scaleDown:
    selectPolicy: Disabled          # HPA only scales up, never down
```

This is useful for workloads where you want human approval before scaling down.

---

## 9.10 — Vertical Pod Autoscaler (VPA) Overview

While HPA changes the **number** of pods, VPA changes the **resource requests/limits** of existing pods.

```
HPA: "You need more pods"     → 3 pods × 200m CPU  →  6 pods × 200m CPU
VPA: "Your pods need more CPU" → 3 pods × 200m CPU  →  3 pods × 500m CPU
```

### VPA Modes

| Mode | Behavior |
|---|---|
| `Off` | Only provides recommendations (view with `kubectl describe vpa`) |
| `Initial` | Sets requests only at Pod creation time (doesn't restart running pods) |
| `Auto` | Evicts and recreates Pods with updated requests (causes brief downtime) |

### VPA Manifest

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: webapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  updatePolicy:
    updateMode: "Off"              # Start with Off — just get recommendations
  resourcePolicy:
    containerPolicies:
      - containerName: webapp
        minAllowed:
          cpu: 50m
          memory: 64Mi
        maxAllowed:
          cpu: "2"
          memory: 2Gi
```

```powershell
# VPA is NOT built-in — install it first
# https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler

# View recommendations
kubectl describe vpa webapp-vpa
# Recommendation:
#   Container:  webapp
#     Lower Bound:   cpu: 100m, memory: 128Mi
#     Target:        cpu: 250m, memory: 256Mi    ← Use these as your requests
#     Upper Bound:   cpu: 500m, memory: 512Mi
```

> [!CAUTION]
> **Never use HPA and VPA on the same metric (CPU or memory) for the same Deployment.** They'll fight each other — HPA wants more pods, VPA wants bigger pods, and the result is chaos. You can use HPA on CPU + VPA on memory (or vice versa), or use VPA in `Off` mode just for recommendations alongside HPA.

---

## 9.11 — KEDA: Event-Driven Autoscaling

**KEDA** (Kubernetes Event-Driven Autoscaling) extends HPA to scale on virtually anything: message queue depth, cron schedules, database rows, HTTP request rate, etc.

```
Standard HPA:  CPU / Memory / Custom metrics (needs adapter per source)
KEDA:          60+ built-in scalers (Kafka, SQS, RabbitMQ, Cron, PostgreSQL, Redis, HTTP, ...)
```

### How KEDA Works

```
KEDA ScaledObject → creates/manages → HPA → scales → Deployment
                                        ↑
                              KEDA provides metrics
```

### KEDA ScaledObject Example

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
spec:
  scaleTargetRef:
    name: order-processor
  minReplicaCount: 0               # Can scale to zero! (HPA can't do this)
  maxReplicaCount: 50
  triggers:
    - type: rabbitmq
      metadata:
        queueName: orders
        host: amqp://rabbitmq.default.svc.cluster.local
        queueLength: "10"          # Scale when > 10 messages per pod
```

### Key KEDA Features

| Feature | Standard HPA | KEDA |
|---|---|---|
| Scale to zero | ❌ No (min 1) | ✅ Yes |
| Event-driven sources | Needs custom adapter per source | 60+ built-in scalers |
| Cron-based scaling | ❌ No | ✅ Yes |
| Scale on external metrics | Complex setup | Simple ScaledObject |

> [!TIP]
> KEDA is particularly powerful for **scale-to-zero** workloads (event processors, batch jobs, dev environments). Standard HPA's minimum is 1 pod. KEDA can scale to 0 and wake up pods when events arrive.

---

## 9.12 — Troubleshooting HPA

### HPA Shows `<unknown>` for Metrics

```powershell
kubectl get hpa
# NAME         TARGETS         MINPODS   MAXPODS   REPLICAS
# webapp-hpa   <unknown>/50%   2         10        2
```

**Causes**:
1. Metrics Server is not installed → `minikube addons enable metrics-server`
2. Pods don't have resource requests → Add `resources.requests.cpu` to the Pod spec
3. Metrics Server just started → Wait 60-90 seconds for metrics to populate
4. Pod name mismatch → Verify `scaleTargetRef.name` matches the Deployment name

### HPA Not Scaling Up

```powershell
kubectl describe hpa webapp-hpa
# Check Conditions:
# - AbleToScale: True/False
# - ScalingActive: True/False
# - ScalingLimited: True (might be at maxReplicas)
#
# Check Events:
# - "New size: X; reason: cpu resource utilization above target"
# - "readiness didn't change" → pods aren't becoming Ready
```

### HPA Flapping (Scaling Up and Down Rapidly)

**Cause**: Stabilization window too short, or metric is noisy.

**Fix**: Increase `stabilizationWindowSeconds` in `behavior`:
```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 600    # Wait 10 minutes before scaling down
```

---

## Phase 9 — kubectl Cheat Sheet

| Command | Purpose |
|---|---|
| `minikube addons enable metrics-server` | Enable metrics collection |
| `kubectl top nodes` | Node resource usage |
| `kubectl top pods` | Pod resource usage |
| `kubectl top pods --sort-by=cpu` | Sort by CPU |
| `kubectl autoscale deployment <name> --cpu-percent=50 --min=2 --max=10` | Create HPA imperatively |
| `kubectl get hpa` | List HPAs with current metrics |
| `kubectl get hpa -w` | Watch HPA in real-time |
| `kubectl describe hpa <name>` | Detailed HPA status + conditions + events |
| `kubectl delete hpa <name>` | Remove an HPA |
| `kubectl get apiservices \| findstr metrics` | Verify metrics API is registered |

---

## 🔬 Lab Exercise 9: Autoscale Under Load

### Objective

Deploy a CPU-intensive application, configure an HPA, generate load, and watch Kubernetes automatically scale up and back down.

### Step 1: Deploy the Application

Create `lab9-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lab9-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: lab9
  template:
    metadata:
      labels:
        app: lab9
    spec:
      containers:
        - name: php-apache
          image: registry.k8s.io/hpa-example    # Official K8s HPA test image
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 200m                          # CRITICAL: HPA needs this
            limits:
              cpu: 500m
---
apiVersion: v1
kind: Service
metadata:
  name: lab9-svc
spec:
  selector:
    app: lab9
  ports:
    - port: 80
      targetPort: 80
```

```powershell
kubectl apply -f lab9-deployment.yaml
```

### Step 2: Create the HPA

Create `lab9-hpa.yaml`:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: lab9-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: lab9-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60        # Faster scale-down for lab purposes
```

```powershell
kubectl apply -f lab9-hpa.yaml

# Verify HPA is working (wait for metrics to appear)
kubectl get hpa lab9-hpa
# TARGETS should show a number, not <unknown>
```

### Step 3: Generate Load

Open a **second terminal** and run a load generator:

```powershell
kubectl run load-generator --image=busybox:1.36 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://lab9-svc; done"
```

### Step 4: Watch the Scaling

In your **first terminal**:

```powershell
# Watch HPA metrics update
kubectl get hpa lab9-hpa -w

# In another window, watch pods scale up
kubectl get pods -l app=lab9 -w
```

You should see:
1. CPU target rises above 50%
2. HPA increases replicas (2 → 3 → 5 → ...)
3. CPU per pod drops as load is distributed
4. Scaling stabilizes

### Step 5: Remove Load and Watch Scale-Down

```powershell
# Kill the load generator
kubectl delete pod load-generator

# Watch pods scale back down (after stabilization window)
kubectl get hpa lab9-hpa -w
kubectl get pods -l app=lab9 -w
```

### What to Submit

1. Your `lab9-deployment.yaml` and `lab9-hpa.yaml`
2. Output of `kubectl get hpa lab9-hpa` **during load** (showing elevated CPU and increased replicas)
3. Output of `kubectl get hpa lab9-hpa` **after load stops** (showing scale-down)
4. Output of `kubectl describe hpa lab9-hpa` (showing scaling events in the Events section)

---

> **⏸️ CHECKPOINT**: Submit your lab results or request to skip. Next: **Phase 10: Custom Resource Definitions (CRDs)** — extending the Kubernetes API with your own resource types.

# Phase 6: Application Health, Security, & Troubleshooting

> **Goal**: Make your applications production-ready. Configure health probes so Kubernetes knows when your app is truly healthy, set resource boundaries so one rogue Pod can't starve the cluster, lock down access with RBAC, and master the debugging toolkit that separates "it's broken" from "I know exactly why."

---

## 6.1 — Health Probes: Telling Kubernetes What "Healthy" Means

By default, Kubernetes only knows if your container *process* is running. It has no idea if your application is actually serving traffic, stuck in a deadlock, or waiting for a database connection. **Probes** give Kubernetes application-level health awareness.

### Three Types of Probes

| Probe | Question It Answers | What Happens on Failure |
|---|---|---|
| **Liveness** | "Is the app alive or deadlocked?" | Container is **killed and restarted** |
| **Readiness** | "Is the app ready to receive traffic?" | Pod is **removed from Service endpoints** (no traffic routed to it) |
| **Startup** | "Has the app finished starting up?" | Liveness and readiness probes are **disabled** until startup succeeds |

### How They Work Together

```
Pod starts
    │
    ▼
┌─────────────────┐
│  Startup Probe   │  ← Runs first. Protects slow-starting apps from being killed
│  (if configured) │     by liveness probe during boot.
└────────┬────────┘
         │ succeeds
         ▼
┌─────────────────┐     ┌──────────────────┐
│  Liveness Probe  │     │  Readiness Probe  │  ← Both run concurrently after startup
│  (is it alive?)  │     │  (can it serve?)  │     succeeds.
└─────────────────┘     └──────────────────┘
```

### Probe Mechanisms

Three ways to check health:

#### 1. HTTP GET (Most Common for Web Apps)

```yaml
livenessProbe:
  httpGet:
    path: /healthz              # Your app's health endpoint
    port: 8080
    httpHeaders:                # Optional custom headers
      - name: X-Health-Check
        value: "kubernetes"
  initialDelaySeconds: 15       # Wait 15s after container start before first probe
  periodSeconds: 10             # Probe every 10 seconds
  timeoutSeconds: 3             # Probe must respond within 3s
  failureThreshold: 3           # 3 consecutive failures = unhealthy
  successThreshold: 1           # 1 success = healthy again (only for readiness)
```

HTTP status 200-399 = success. Anything else = failure.

#### 2. TCP Socket (For Non-HTTP Services — Databases, Message Queues)

```yaml
livenessProbe:
  tcpSocket:
    port: 3306                  # Just checks: can I open a TCP connection?
  initialDelaySeconds: 30
  periodSeconds: 10
```

Connection accepted = success. Connection refused/timeout = failure.

#### 3. Exec Command (For Custom Checks)

```yaml
livenessProbe:
  exec:
    command:
      - /bin/sh
      - -c
      - "pg_isready -U postgres"    # Run this command inside the container
  initialDelaySeconds: 30
  periodSeconds: 10
```

Exit code 0 = success. Non-zero = failure.

### A Production-Grade Pod With All Three Probes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: production-app
spec:
  containers:
    - name: app
      image: myapp:2.0
      ports:
        - containerPort: 8080

      # Startup probe — gives the app up to 5 min (30 × 10s) to start
      startupProbe:
        httpGet:
          path: /healthz
          port: 8080
        failureThreshold: 30
        periodSeconds: 10

      # Liveness probe — restarts container if app becomes unresponsive
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        periodSeconds: 10
        failureThreshold: 3
        timeoutSeconds: 3

      # Readiness probe — removes from service if temporarily unable to serve
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        periodSeconds: 5
        failureThreshold: 2
        timeoutSeconds: 2
```

> [!IMPORTANT]
> **Liveness vs. Readiness — the critical distinction**:
> - **Liveness failure** = "this process is broken, kill it and start fresh" → container restart
> - **Readiness failure** = "this process is temporarily busy (warming cache, high load), stop sending traffic but don't kill it" → removed from Service endpoints
>
> **Common mistake**: Making the liveness probe too aggressive or checking dependencies (like a database) in the liveness probe. If the database is down, restarting the app container won't fix it — it'll just create a crash loop. Check dependencies in the **readiness** probe instead.

> [!TIP]
> **Startup probe saves you from a classic problem**: Slow-starting apps (JVM, .NET, large ML model loading) that take 2+ minutes to boot. Without a startup probe, the liveness probe would kill the container before it finishes starting. The startup probe holds off all other probes until the app signals it's initialized.

---

## 6.2 — Resource Management: Requests, Limits, and QoS

Without resource constraints, a single misbehaving Pod can consume all CPU and memory on a node, starving every other workload. **Requests** and **Limits** prevent this.

### Requests vs. Limits

| Concept | Purpose | What Happens |
|---|---|---|
| **Requests** | The *guaranteed minimum* resources for your container. Used by the scheduler to decide which node has enough capacity. | A Pod won't be scheduled on a node that can't satisfy its requests. |
| **Limits** | The *hard ceiling*. The container cannot exceed this. | CPU: throttled (slowed down). Memory: OOM-killed (container is killed). |

```yaml
spec:
  containers:
    - name: app
      image: myapp:2.0
      resources:
        requests:
          cpu: 250m          # 250 millicores = 0.25 CPU cores (guaranteed)
          memory: 256Mi      # 256 MiB (guaranteed)
        limits:
          cpu: 500m          # 500 millicores = 0.5 CPU cores (ceiling)
          memory: 512Mi      # 512 MiB (ceiling — OOM kill if exceeded)
```

### CPU Units

| Value | Meaning |
|---|---|
| `1` | 1 full CPU core |
| `500m` | 500 millicores = 0.5 CPU core |
| `100m` | 100 millicores = 0.1 CPU core |
| `250m` | 250 millicores = 0.25 CPU core |

### Memory Units

| Value | Meaning |
|---|---|
| `128Mi` | 128 mebibytes (134,217,728 bytes) |
| `1Gi` | 1 gibibyte (1,073,741,824 bytes) |
| `256M` | 256 megabytes (256,000,000 bytes — slightly less than 256Mi) |

> [!WARNING]
> **CPU over-limit = throttled** (slower but alive). **Memory over-limit = OOM-killed** (container is terminated). This is the single most common reason for pods in `OOMKilled` status. If your app uses more memory than its limit, Kubernetes kills it. Period.

### Quality of Service (QoS) Classes

Kubernetes assigns every Pod a QoS class based on its resource configuration. This determines which Pods get killed first under node memory pressure:

| QoS Class | Condition | Eviction Priority |
|---|---|---|
| **Guaranteed** | Every container has requests = limits for both CPU and memory | Last to be evicted (highest priority) |
| **Burstable** | At least one container has requests ≠ limits, or only requests are set | Medium priority |
| **BestEffort** | No requests or limits set at all | First to be evicted (lowest priority) |

```powershell
# Check a Pod's QoS class
kubectl get pod <name> -o jsonpath='{.status.qosClass}'
```

> [!TIP]
> **Production recommendation**: Always set both requests AND limits. Set them equal for critical workloads (databases, payment processing) to get `Guaranteed` QoS. Set limits higher than requests for burstable workloads (web servers, batch jobs).

### LimitRanges — Default Constraints per Namespace

A `LimitRange` sets default requests/limits for containers that don't specify them, and enforces min/max boundaries:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: development
spec:
  limits:
    - type: Container
      default:              # Default LIMITS (applied if container doesn't set limits)
        cpu: 500m
        memory: 256Mi
      defaultRequest:       # Default REQUESTS (applied if container doesn't set requests)
        cpu: 100m
        memory: 128Mi
      max:                  # No container can exceed these
        cpu: "2"
        memory: 2Gi
      min:                  # No container can go below these
        cpu: 50m
        memory: 64Mi
```

### ResourceQuotas — Namespace-Level Budgets

A `ResourceQuota` caps the total resources all Pods in a namespace can consume:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: development
spec:
  hard:
    requests.cpu: "10"          # Total CPU requests across all pods: 10 cores max
    requests.memory: 20Gi      # Total memory requests: 20 GiB max
    limits.cpu: "20"            # Total CPU limits: 20 cores max
    limits.memory: 40Gi
    pods: "50"                  # Max 50 pods in this namespace
    persistentvolumeclaims: "10"
    services: "20"
```

```powershell
kubectl get resourcequota -n development
kubectl describe resourcequota team-quota -n development
# Shows used vs. hard limits for each resource
```

---

## 6.3 — RBAC: Role-Based Access Control

RBAC controls **who** (identity) can do **what** (verbs) on **which resources** (nouns) in **which scope** (namespace or cluster-wide).

### The Four RBAC Objects

```
                    Namespace-Scoped          Cluster-Scoped
                    ─────────────────         ────────────────
 Defines rules:     Role                      ClusterRole
 Binds to user:     RoleBinding               ClusterRoleBinding
```

### ServiceAccounts — Pod Identity

Every Pod runs as a **ServiceAccount**. If you don't specify one, it uses `default` in its namespace.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: default
```

Assign it to a Pod:
```yaml
spec:
  serviceAccountName: app-service-account
  containers:
    - name: app
      image: myapp:2.0
```

```powershell
# Create a service account
kubectl create serviceaccount app-sa

# List service accounts
kubectl get serviceaccounts
kubectl get sa

# See what a pod is running as
kubectl get pod <name> -o jsonpath='{.spec.serviceAccountName}'
```

### Role — Define Permissions (Namespace-Scoped)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
  - apiGroups: [""]              # "" = core API group (pods, services, configmaps)
    resources: ["pods"]
    verbs: ["get", "list", "watch"]

  - apiGroups: [""]
    resources: ["pods/log"]      # Sub-resource: pod logs
    verbs: ["get"]

  - apiGroups: ["apps"]          # "apps" API group (deployments, replicasets)
    resources: ["deployments"]
    verbs: ["get", "list"]
```

### Available Verbs

| Verb | Meaning |
|---|---|
| `get` | Read a specific resource by name |
| `list` | List all resources of this type |
| `watch` | Stream changes in real-time |
| `create` | Create new resources |
| `update` | Modify existing resources |
| `patch` | Partially modify resources |
| `delete` | Delete resources |
| `deletecollection` | Delete multiple resources at once |

### ClusterRole — Cluster-Wide Permissions

Same as Role but not namespaced — applies across all namespaces or to cluster-scoped resources (nodes, PVs, namespaces themselves):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-viewer
rules:
  - apiGroups: [""]
    resources: ["nodes"]           # Nodes are cluster-scoped
    verbs: ["get", "list", "watch"]

  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list"]
```

### RoleBinding — Attach Permissions to an Identity

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: app-service-account
    namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

This says: "The ServiceAccount `app-service-account` in namespace `default` gets the permissions defined in Role `pod-reader`."

### ClusterRoleBinding — Cluster-Wide Binding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-viewer-binding
subjects:
  - kind: ServiceAccount
    name: monitoring-sa
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: node-viewer
  apiGroup: rbac.authorization.k8s.io
```

### Testing Permissions

```powershell
# Can the current user get pods?
kubectl auth can-i get pods
# yes

# Can a specific service account create deployments?
kubectl auth can-i create deployments --as=system:serviceaccount:default:app-service-account
# no

# What can a service account do? (list all permissions)
kubectl auth can-i --list --as=system:serviceaccount:default:app-service-account
```

> [!IMPORTANT]
> **Principle of Least Privilege**: Always grant the minimum permissions needed. Start with nothing, add only what's required. Never use `ClusterRoleBinding` with the built-in `cluster-admin` ClusterRole for application workloads — that's the equivalent of running everything as root.

### Built-in ClusterRoles (Pre-Installed)

| ClusterRole | Permissions |
|---|---|
| `cluster-admin` | Full access to everything. God mode. |
| `admin` | Full access within a namespace (no ResourceQuota/Namespace modification) |
| `edit` | Read/write most resources in a namespace (no Roles/RoleBindings) |
| `view` | Read-only access to most resources in a namespace |

```powershell
# Quick way to give a SA read-only access to a namespace
kubectl create rolebinding view-binding --clusterrole=view --serviceaccount=default:app-sa --namespace=default
```

---

## 6.4 — The Debugging Toolkit: A Systematic Approach

When something breaks, follow this flowchart. Don't guess — diagnose.

### The Debugging Ladder

```
Step 1: GET  →  What's the status?
Step 2: DESCRIBE  →  Why is it in this status?
Step 3: LOGS  →  What did the app say before it died?
Step 4: EXEC  →  What's happening inside the container right now?
Step 5: EVENTS  →  What did the cluster do?
Step 6: DEBUG  →  Attach a debug container for deep inspection
```

### Step 1: GET — Assess the Situation

```powershell
kubectl get pods                          # Quick status overview
kubectl get pods -o wide                  # Include node assignment and IP
kubectl get pods -A                       # All namespaces
kubectl get all                           # Pods, Services, Deployments, ReplicaSets
kubectl get events --sort-by='.lastTimestamp'   # Recent cluster activity
```

**Common Pod statuses and their meaning:**

| Status | Meaning | Next Step |
|---|---|---|
| `Pending` | Not scheduled yet — no node has enough resources, or PVC isn't bound | `describe` → check Events |
| `ContainerCreating` | Image is being pulled or volume is being mounted | Wait, or `describe` if stuck |
| `Running` | Container process is alive (doesn't mean app is healthy) | Check readiness, `logs` |
| `CrashLoopBackOff` | Container keeps crashing and K8s is backing off restart attempts | `logs` → `logs --previous` |
| `ImagePullBackOff` | Can't pull the container image (wrong name, tag, or registry auth) | `describe` → check image name |
| `OOMKilled` | Container exceeded its memory limit | Increase limits or fix memory leak |
| `Evicted` | Node ran out of resources, pod was evicted | Check node resources, set requests |
| `Terminating` | Pod is being deleted (stuck = finalizer issue) | Check finalizers |
| `Error` | Container exited with a non-zero exit code | `logs` → `logs --previous` |

### Step 2: DESCRIBE — The Deep Dive

```powershell
kubectl describe pod <name>
```

Key sections to read:
- **Status / Conditions** — Is it `Ready`? `PodScheduled`?
- **Containers** — State (`Waiting`, `Running`, `Terminated`), restart count, last termination reason
- **Events** (bottom) — Chronological log of what Kubernetes did: scheduling, pulling images, starting containers, probe failures

```powershell
# Also works for other resources
kubectl describe deployment <name>
kubectl describe svc <name>
kubectl describe pvc <name>
kubectl describe node <name>         # Check node capacity and allocated resources
```

### Step 3: LOGS — What Did the App Say?

```powershell
# Current container logs
kubectl logs <pod-name>

# Logs from the PREVIOUS container (after a crash — crucial for CrashLoopBackOff)
kubectl logs <pod-name> --previous

# Specific container in a multi-container pod
kubectl logs <pod-name> -c <container-name>

# Follow logs in real-time
kubectl logs <pod-name> -f

# Last 100 lines
kubectl logs <pod-name> --tail=100

# Logs from the last 5 minutes
kubectl logs <pod-name> --since=5m

# Logs from all pods matching a label
kubectl logs -l app=webapp --all-containers
```

> [!TIP]
> **`--previous` is the most overlooked flag in Kubernetes.** When a container is in `CrashLoopBackOff`, `kubectl logs <pod>` shows you the logs from the *current* (probably just-started, empty) container. `--previous` shows you the logs from the *crashed* container — which is where the actual error message is.

### Step 4: EXEC — Get Inside the Container

```powershell
# Open an interactive shell
kubectl exec -it <pod-name> -- /bin/sh
kubectl exec -it <pod-name> -- /bin/bash     # If bash is available

# Run a specific command
kubectl exec <pod-name> -- cat /etc/config/app.conf
kubectl exec <pod-name> -- env
kubectl exec <pod-name> -- curl -s localhost:8080/healthz
kubectl exec <pod-name> -- netstat -tlnp     # What ports are listening?
kubectl exec <pod-name> -- df -h             # Disk usage
kubectl exec <pod-name> -- ps aux            # Running processes

# Multi-container pod
kubectl exec -it <pod-name> -c <container-name> -- /bin/sh
```

### Step 5: EVENTS — The Cluster's Activity Log

```powershell
# All events, sorted by time
kubectl get events --sort-by='.lastTimestamp'

# Events for a specific namespace
kubectl get events -n production --sort-by='.lastTimestamp'

# Events for a specific resource type
kubectl get events --field-selector involvedObject.kind=Pod
kubectl get events --field-selector involvedObject.kind=PersistentVolumeClaim

# Events for a specific object
kubectl get events --field-selector involvedObject.name=my-pod

# Warning events only
kubectl get events --field-selector type=Warning
```

### Step 6: Ephemeral Debug Containers (K8s 1.25+)

When a container doesn't have a shell (distroless images, scratch-based images), you can attach a **debug container** to a running Pod:

```powershell
# Attach a debug container with full tools to a running pod
kubectl debug -it <pod-name> --image=busybox:1.36 --target=<container-name>

# This creates a temporary container inside the same Pod that shares:
# - The process namespace (can see the target container's processes)
# - The network namespace (same IP, can test localhost connections)

# Debug a node directly (creates a privileged pod on the node)
kubectl debug node/<node-name> -it --image=ubuntu
```

### Bonus: Node-Level Debugging

```powershell
# Check node resource usage
kubectl top nodes                           # Requires metrics-server
kubectl top pods                            # Per-pod resource consumption
kubectl top pods --sort-by=memory           # Find memory hogs

# Detailed node info — allocated resources, conditions, taints
kubectl describe node <node-name>

# Install metrics-server on Minikube
minikube addons enable metrics-server
```

---

## 6.5 — Common Failure Scenarios and Diagnosis

### Scenario 1: Pod Stuck in `Pending`

```powershell
kubectl describe pod <name>
# Look for Events like:
# "0/3 nodes are available: 1 Insufficient cpu, 2 Insufficient memory"
# "no persistent volumes available for this claim"
# "0/3 nodes are available: 3 node(s) had volume node affinity conflict"
```

**Fix**: Reduce resource requests, add nodes, or fix PVC binding issues.

### Scenario 2: `CrashLoopBackOff`

```powershell
kubectl logs <pod-name> --previous     # ← THE critical command
# Look for:
# - Application error messages
# - Missing environment variables
# - Failed database connections
# - Permission denied errors
```

**Fix**: Fix the application error, ensure ConfigMaps/Secrets exist, check connectivity.

### Scenario 3: `ImagePullBackOff`

```powershell
kubectl describe pod <name>
# Events will say:
# "Failed to pull image 'myrepo/myapp:v1': rpc error: code = NotFound"
# "unauthorized: authentication required"
```

**Fix**: Correct the image name/tag, create an `imagePullSecret`, or check registry access.

### Scenario 4: Service Not Routing Traffic

```powershell
# Check 1: Does the Service have endpoints?
kubectl get endpoints <service-name>
# If ENDPOINTS is <none>, the selector doesn't match any running Pods

# Check 2: Do the labels match?
kubectl get pods --show-labels
kubectl describe svc <service-name>    # Check the Selector field

# Check 3: Is the targetPort correct?
kubectl exec <pod-name> -- netstat -tlnp    # What port is the app ACTUALLY listening on?
```

### Scenario 5: OOMKilled

```powershell
kubectl describe pod <name>
# Last State: Terminated
# Reason: OOMKilled
# Exit Code: 137

kubectl top pod <name>    # See current memory usage
```

**Fix**: Increase memory limits, or investigate and fix the memory leak.

---

## Phase 6 — kubectl Cheat Sheet

| Command | Purpose |
|---|---|
| `kubectl get pods` | List pods with status |
| `kubectl get pods -o wide` | Include node and IP |
| `kubectl get all` | List common resources |
| `kubectl describe pod <name>` | Full pod details + events |
| `kubectl logs <name>` | Container logs |
| `kubectl logs <name> --previous` | Logs from crashed container |
| `kubectl logs <name> -f` | Follow logs in real-time |
| `kubectl logs <name> --since=5m` | Logs from last 5 minutes |
| `kubectl logs -l app=X --all-containers` | Logs from all pods with label |
| `kubectl exec -it <name> -- /bin/sh` | Shell into container |
| `kubectl debug -it <name> --image=busybox:1.36 --target=<ctr>` | Attach debug container |
| `kubectl get events --sort-by='.lastTimestamp'` | Chronological events |
| `kubectl get events --field-selector type=Warning` | Warning events only |
| `kubectl top nodes` | Node resource usage |
| `kubectl top pods --sort-by=memory` | Pod resource usage by memory |
| `kubectl auth can-i <verb> <resource>` | Check current user permissions |
| `kubectl auth can-i --list --as=system:serviceaccount:<ns>:<sa>` | List SA permissions |
| `kubectl create sa <name>` | Create a ServiceAccount |
| `kubectl create role <name> --verb=get,list --resource=pods` | Create Role imperatively |
| `kubectl create rolebinding <name> --role=<role> --serviceaccount=<ns>:<sa>` | Bind Role to SA |
| `kubectl create clusterrole <name> --verb=get,list --resource=nodes` | Create ClusterRole |
| `kubectl create clusterrolebinding <name> --clusterrole=<cr> --serviceaccount=<ns>:<sa>` | Bind ClusterRole |
| `kubectl describe node <name>` | Node capacity, allocated resources, conditions |
| `minikube addons enable metrics-server` | Enable resource metrics on Minikube |

---

## 🔬 Lab Exercise 6 (Capstone): Production-Ready Application with Probes, Limits, RBAC, and Debugging

### Objective

Deploy a "production-ready" Nginx application with all the Phase 6 concepts applied, then intentionally break it and diagnose the failures.

### Step 1: Create the ServiceAccount and RBAC

Create `lab6-rbac.yaml` with three resources:

1. **ServiceAccount** named `lab6-app-sa` in the `default` namespace

2. **Role** named `lab6-pod-reader`:
   - API group: `""` (core)
   - Resources: `pods`, `pods/log`
   - Verbs: `get`, `list`, `watch`

3. **RoleBinding** named `lab6-pod-reader-binding`:
   - Bind `lab6-pod-reader` Role to `lab6-app-sa` ServiceAccount

Apply:
```powershell
kubectl apply -f lab6-rbac.yaml
```

Verify:
```powershell
kubectl auth can-i get pods --as=system:serviceaccount:default:lab6-app-sa
# yes
kubectl auth can-i create deployments --as=system:serviceaccount:default:lab6-app-sa
# no
```

### Step 2: Create the Production Deployment

Create `lab6-deployment.yaml`:

- **Deployment** named `lab6-app`, 3 replicas
- **ServiceAccountName**: `lab6-app-sa`
- **Container**: `nginx:1.27`, port 80
- **Resources**:
  - Requests: cpu `100m`, memory `64Mi`
  - Limits: cpu `200m`, memory `128Mi`
- **Liveness probe**: HTTP GET to `/` on port 80, periodSeconds 10, failureThreshold 3
- **Readiness probe**: HTTP GET to `/` on port 80, periodSeconds 5, failureThreshold 2
- **Startup probe**: HTTP GET to `/` on port 80, failureThreshold 10, periodSeconds 5

Apply:
```powershell
kubectl apply -f lab6-deployment.yaml
```

Verify:
```powershell
# All 3 pods should be Running and Ready (1/1)
kubectl get pods -l app=lab6-app -o wide

# Check QoS class (should be Burstable since requests ≠ limits)
kubectl get pod <any-pod-name> -o jsonpath='{.status.qosClass}'

# Check the service account
kubectl get pod <any-pod-name> -o jsonpath='{.spec.serviceAccountName}'
```

### Step 3: Break It and Debug It

#### Break 1: Simulate a Liveness Failure

```powershell
# Delete the default nginx index page inside a pod (so / returns 403)
kubectl exec <pod-name> -- rm /usr/share/nginx/html/index.html

# Watch the pod — liveness probe will fail (403 ≠ 200), container will restart
kubectl get pods -w
# After ~30s (3 failures × 10s period), the container restarts
# RESTARTS column increments
```

Capture:
```powershell
kubectl describe pod <pod-name>
# Look for "Liveness probe failed: HTTP probe failed with statuscode: 403" in Events
```

#### Break 2: Simulate an OOMKill

Create a Pod that deliberately exceeds its memory limit:

```yaml
# lab6-oomkill.yaml
apiVersion: v1
kind: Pod
metadata:
  name: lab6-oom-test
spec:
  containers:
    - name: stress
      image: polinux/stress
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "256M", "--vm-hang", "1"]
      resources:
        requests:
          memory: 64Mi
        limits:
          memory: 128Mi       # Limit is 128Mi but we're allocating 256M → OOMKilled
```

```powershell
kubectl apply -f lab6-oomkill.yaml
kubectl get pod lab6-oom-test -w
# Status will go to OOMKilled, then CrashLoopBackOff

kubectl describe pod lab6-oom-test
# Last State: Terminated, Reason: OOMKilled, Exit Code: 137
```

#### Break 3: Simulate ImagePullBackOff

```powershell
kubectl run lab6-bad-image --image=nginx:this-tag-does-not-exist
kubectl get pods -w
# Status: ImagePullBackOff

kubectl describe pod lab6-bad-image
# Events: "Failed to pull image... manifest unknown"

# Clean up
kubectl delete pod lab6-bad-image
```

### Step 4: Resource Monitoring (Optional — Requires metrics-server)

```powershell
minikube addons enable metrics-server

# Wait ~60s for metrics to populate, then:
kubectl top pods -l app=lab6-app
kubectl top nodes
```

### What to Submit

1. Your `lab6-rbac.yaml` (ServiceAccount + Role + RoleBinding)
2. Your `lab6-deployment.yaml` (with all three probes and resource limits)
3. Output of `kubectl auth can-i get pods --as=system:serviceaccount:default:lab6-app-sa`
4. Output of `kubectl describe pod <pod-name>` showing the liveness probe failure event
5. Output of `kubectl describe pod lab6-oom-test` showing the OOMKilled reason
6. Any observations or questions from your debugging

---

> ## 🎓 CURRICULUM COMPLETE
>
> You've covered all six phases:
>
> | Phase | Topic |
> |---|---|
> | 1 | Architecture & Pod Fundamentals |
> | 2 | Declarative Deployments & Scaling |
> | 3 | Service Discovery & Cluster Networking |
> | 4 | Configuration & Secrets Externalization |
> | 5 | Storage & Persistence Lifecycle |
> | 6 | Application Health, Security, & Troubleshooting |
>
> **From here, the next steps toward advanced K8s administration would be:**
> - **Helm** — Package management for Kubernetes (templated manifests)
> - **Network Policies** — Firewall rules between Pods
> - **Horizontal Pod Autoscaler (HPA)** — Automatic scaling based on metrics
> - **Custom Resource Definitions (CRDs)** — Extending the Kubernetes API
> - **Operators** — Encoding domain-specific operational knowledge in controllers
> - **GitOps with ArgoCD / Flux** — Declarative, Git-driven cluster management
> - **Observability stack** — Prometheus + Grafana + Loki for metrics, dashboards, and logs
> - **CKA Certification prep** — The Certified Kubernetes Administrator exam
>
> Submit your lab or let me know what you'd like to dive deeper into.

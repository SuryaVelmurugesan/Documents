# Phase 1: Local Architecture & Pod Fundamentals

> **Goal**: Understand what makes Kubernetes tick under the hood, stand up a local cluster, and get your first Pods running.

---

## 1.1 — Kubernetes Architecture: The Two Halves

A Kubernetes cluster is split into exactly two layers: the **Control Plane** and the **Worker Nodes**. Every single thing that happens in K8s is a conversation between these two.

### Control Plane (The Brain)

Think of it as the "management layer." It never runs your application containers directly.

| Component | What It Does |
|---|---|
| **kube-apiserver** | The front door. Every `kubectl` command, every internal component, talks to this REST API. It's the *only* component that reads/writes to etcd. |
| **etcd** | A distributed key-value store. This *is* your cluster's state — every Pod, every Service, every Secret lives here as a serialized object. If etcd dies and you have no backup, your cluster state is gone. |
| **kube-scheduler** | Watches for newly created Pods that have no assigned node. It evaluates resource requests, taints, tolerations, affinity rules, and picks the best node. It does *not* run the Pod — it just makes the decision. |
| **kube-controller-manager** | Runs a collection of control loops (controllers). Each controller watches a specific resource type and ensures the *actual state* matches the *desired state*. Example: the ReplicaSet controller ensures the correct number of Pod replicas exist. |

### Worker Nodes (The Muscle)

These are the machines that actually run your containers.

| Component | What It Does |
|---|---|
| **kubelet** | An agent running on every node. It receives Pod specs from the API server and ensures the containers described in those specs are running and healthy. It talks directly to the container runtime (containerd, CRI-O). |
| **kube-proxy** | Manages network rules on each node. It implements the `Service` abstraction by programming iptables/IPVS rules so traffic to a Service VIP gets forwarded to the correct Pod IPs. |
| **Container Runtime** | The low-level engine that pulls images and runs containers. Kubernetes uses the **CRI (Container Runtime Interface)** standard. The default in modern K8s is **containerd**. Docker (dockershim) was removed in K8s v1.24. |

### The Reconciliation Loop (Critical Concept)

This is the heartbeat of Kubernetes:

```
You declare: "I want 3 replicas of nginx"
         ↓
API Server stores this desired state in etcd
         ↓
Controller Manager detects: actual state (0 pods) ≠ desired state (3 pods)
         ↓
Controller creates 3 Pod objects
         ↓
Scheduler assigns each Pod to a node
         ↓
Kubelet on each node pulls the image and starts the container
         ↓
Controller continuously watches: if a pod dies, it creates a new one
```

This is **declarative infrastructure**. You never say "start a container." You say "this is the state I want," and Kubernetes converges to it.

---

## 1.2 — Local Cluster Setup: Minikube

For learning, we'll use **Minikube**. It creates a single-node cluster (both control plane and worker on the same VM/container) on your local machine.

> [!NOTE]
> You can also use **Kind** (Kubernetes IN Docker) or **K3s** (lightweight K8s). Minikube is the most beginner-friendly and has the best add-on ecosystem for learning.

### Installation (Windows)

**Prerequisites**: You need a hypervisor. Minikube supports Hyper-V or Docker Desktop.

```powershell
# Option 1: Using winget (recommended)
winget install Kubernetes.minikube

# Option 2: Using Chocolatey
choco install minikube

# Verify installation
minikube version
```

### Starting Your Cluster

```powershell
# Start a cluster with the docker driver (recommended if Docker Desktop is installed)
minikube start --driver=docker

# Or with Hyper-V
minikube start --driver=hyperv

# Verify the cluster is running
minikube status
```

Expected output:
```
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

### Install kubectl

`kubectl` is the CLI tool to interact with any Kubernetes cluster.

```powershell
# Using winget
winget install Kubernetes.kubectl

# Verify
kubectl version --client
```

### Verify Connectivity

```powershell
# This confirms kubectl can talk to your minikube cluster
kubectl cluster-info

# See your single node
kubectl get nodes
```

Expected output:
```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   2m    v1.30.x
```

---

## 1.3 — Manifest Anatomy: The Four Required Fields

Every Kubernetes resource is defined by a YAML manifest. Every manifest has exactly **four top-level fields**:

```yaml
apiVersion: v1              # 1. Which API group and version
kind: Pod                   # 2. What type of resource
metadata:                   # 3. Identity — name, namespace, labels
  name: my-first-pod
  labels:
    app: demo
spec:                       # 4. The desired state — what you actually want
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
```

Let's break this down:

| Field | Purpose | Example |
|---|---|---|
| `apiVersion` | Tells the API server which schema to use to validate this object. Core resources use `v1`. Deployments use `apps/v1`. | `v1`, `apps/v1`, `networking.k8s.io/v1` |
| `kind` | The resource type. | `Pod`, `Deployment`, `Service`, `ConfigMap` |
| `metadata` | Identity data. `name` is required. `namespace` defaults to `default`. `labels` are key-value pairs used for selection and organization. | `name: nginx-pod`, `labels: {app: web}` |
| `spec` | The actual specification. This is where you define containers, volumes, ports, environment variables — everything about the desired state. | Container definitions, resource limits, etc. |

> [!IMPORTANT]
> **How do you know which `apiVersion` to use?** Run:
> ```powershell
> kubectl api-resources
> ```
> This lists every resource type, its short name, API group, and whether it's namespaced. It's the single most useful reference command.

---

## 1.4 — Running, Exploring, and Debugging Pods

### Creating a Pod (Imperative vs. Declarative)

```powershell
# IMPERATIVE: Quick and dirty. Good for testing. Never use in production.
kubectl run nginx-quick --image=nginx:1.27 --port=80

# DECLARATIVE: Apply a manifest file. This is the correct way.
kubectl apply -f pod.yaml
```

### Inspecting Pods

```powershell
# List pods in the default namespace
kubectl get pods

# Wide output — shows node assignment and IP
kubectl get pods -o wide

# Watch pods in real-time (updates every 2s)
kubectl get pods -w

# Full YAML of a running pod (see what K8s actually stored)
kubectl get pod nginx-quick -o yaml
```

### Debugging Pods

When a Pod is stuck in `Pending`, `CrashLoopBackOff`, `ImagePullBackOff`, or `Error`, this is your debugging sequence:

```powershell
# Step 1: DESCRIBE — Shows events, conditions, and scheduling decisions
kubectl describe pod <pod-name>

# Step 2: LOGS — See container stdout/stderr
kubectl logs <pod-name>

# For multi-container pods, specify the container:
kubectl logs <pod-name> -c <container-name>

# Follow logs in real-time:
kubectl logs <pod-name> -f

# Step 3: EXEC — Open a shell inside the running container
kubectl exec -it <pod-name> -- /bin/sh

# For multi-container pods:
kubectl exec -it <pod-name> -c <container-name> -- /bin/sh

# Step 4: EVENTS — Cluster-wide event log
kubectl get events --sort-by='.lastTimestamp'
```

> [!TIP]
> **The debugging order matters**: `describe` → `logs` → `exec` → `events`. This catches 95% of issues. `describe` tells you *why* a Pod can't start. `logs` tells you *why* the app inside is crashing. `exec` lets you poke around inside the container filesystem and network.

### Deleting Pods

```powershell
# Delete a specific pod
kubectl delete pod <pod-name>

# Delete from a manifest file
kubectl delete -f pod.yaml

# Nuclear option: delete all pods in the namespace
kubectl delete pods --all
```

---

## 1.5 — Multi-Container Pods: Sidecar Pattern

A Pod can hold multiple containers that share the same network namespace (localhost) and can share volumes. The most common pattern is the **sidecar**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
  labels:
    app: sidecar-demo
spec:
  containers:
    # Main application container
    - name: app
      image: nginx:1.27
      ports:
        - containerPort: 80
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx

    # Sidecar container — tails the logs
    - name: log-reader
      image: busybox:1.36
      command: ["sh", "-c", "tail -f /var/log/app/access.log"]
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app

  volumes:
    - name: shared-logs
      emptyDir: {}    # Ephemeral volume — lives and dies with the Pod
```

Key points:
- Both containers see `localhost` as each other. If `app` listens on port 80, `log-reader` can `curl localhost:80`.
- The `emptyDir` volume is created when the Pod starts and deleted when the Pod is removed. It's shared between containers.
- Containers in the same Pod are **always co-scheduled** on the same node.

---

## Phase 1 — kubectl Cheat Sheet

| Command | Purpose |
|---|---|
| `minikube start` | Start local cluster |
| `minikube stop` | Stop cluster (preserves state) |
| `minikube delete` | Destroy cluster completely |
| `kubectl cluster-info` | Verify API server connectivity |
| `kubectl get nodes` | List cluster nodes |
| `kubectl api-resources` | List all available resource types + API versions |
| `kubectl run <name> --image=<img>` | Create a pod imperatively |
| `kubectl apply -f <file.yaml>` | Apply a declarative manifest |
| `kubectl get pods` | List pods |
| `kubectl get pods -o wide` | List pods with IP and node info |
| `kubectl get pods -w` | Watch pod status in real-time |
| `kubectl get pod <name> -o yaml` | Dump full pod spec as YAML |
| `kubectl describe pod <name>` | Detailed info + events for debugging |
| `kubectl logs <name>` | View container logs |
| `kubectl logs <name> -c <container>` | View specific container's logs |
| `kubectl exec -it <name> -- /bin/sh` | Shell into a running container |
| `kubectl get events --sort-by='.lastTimestamp'` | View cluster events chronologically |
| `kubectl delete pod <name>` | Delete a pod |
| `kubectl delete -f <file.yaml>` | Delete resources defined in a file |
| `kubectl explain pod.spec.containers` | Built-in docs for any field in a manifest |

> [!TIP]
> `kubectl explain` is your best friend. Want to know what fields `spec.containers` supports?
> ```powershell
> kubectl explain pod.spec.containers --recursive
> ```
> This prints the full schema. No need to Google it.

---

## 🔬 Lab Exercise 1: Deploy and Debug a Multi-Container Pod

### Objective

Create a multi-container Pod where:
1. **Container 1 (`web`)**: Runs `nginx:1.27`, serves on port 80, and writes access logs to a shared volume.
2. **Container 2 (`log-agent`)**: Runs `busybox:1.36`, tails the Nginx access log from the shared volume, acting as a log sidecar.
3. Both containers share an `emptyDir` volume.

### Your Tasks

1. **Write the YAML manifest** (`lab1-pod.yaml`) from scratch. Don't copy the sidecar example above verbatim — adapt it so the Nginx access log path and the busybox tail path match correctly.
   - Nginx writes access logs to: `/var/log/nginx/access.log`
   - Mount the shared volume at `/var/log/nginx` in the web container.
   - Mount the same volume at a path of your choice in the log-agent container.

2. **Apply it**: `kubectl apply -f lab1-pod.yaml`

3. **Verify the pod is running**: `kubectl get pods -o wide`

4. **Generate some traffic**: Use `kubectl exec` to curl localhost from inside the web container:
   ```powershell
   kubectl exec lab1-pod -c web -- curl -s localhost
   ```

5. **Check the sidecar logs**: View the log-agent container's output:
   ```powershell
   kubectl logs lab1-pod -c log-agent
   ```
   You should see the Nginx access log entries generated by your curl commands.

6. **Describe the pod**: Run `kubectl describe pod lab1-pod` and note the Events section.

### What to Submit

Paste the following for my review:
- Your complete `lab1-pod.yaml` manifest
- Output of `kubectl get pods -o wide`
- Output of `kubectl logs lab1-pod -c log-agent` (after generating traffic)
- Output of `kubectl describe pod lab1-pod` (the Events section at the bottom is what I'll focus on)

---

> **⏸️ CHECKPOINT**: I will review your submission, give feedback on your manifest structure and debugging output, and then we proceed to **Phase 2: Declarative Deployments & Scaling**.

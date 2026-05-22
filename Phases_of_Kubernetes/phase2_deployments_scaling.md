# Phase 2: Declarative Deployments & Scaling

> **Goal**: Stop managing individual Pods. Learn to declare the desired state of your application and let Kubernetes handle the rest — including scaling, rolling updates, and automatic rollbacks.

---

## 2.1 — Imperative vs. Declarative: Why It Matters

In Phase 1, you used `kubectl run` to create a Pod. That's **imperative** — you're telling Kubernetes *what to do*, step by step.

The problem? Imperative commands are:
- Not repeatable (no record of what you ran)
- Not version-controllable (can't `git diff` a command)
- Not self-healing (if the Pod dies, it's gone forever — nobody recreates it)

**Declarative** management fixes all three. You write a YAML file that says "I want 3 replicas of nginx:1.27" and apply it. Kubernetes continuously ensures that state exists. You check that YAML into Git. That's your source of truth.

```powershell
# Imperative — DO NOT use in production
kubectl run nginx --image=nginx:1.27
kubectl scale deployment nginx --replicas=3   # won't even work — 'run' creates a Pod, not a Deployment

# Declarative — THE correct approach
kubectl apply -f deployment.yaml
```

> [!IMPORTANT]
> **The Golden Rule**: If you can't reproduce your cluster state by running `kubectl apply -f` on a directory of YAML files, your cluster is unmanageable. Everything goes in a manifest. Everything goes in Git.

---

## 2.2 — The Deployment → ReplicaSet → Pod Hierarchy

This is the single most important resource relationship in Kubernetes:

```
Deployment
  └── ReplicaSet (created automatically)
        └── Pod (created automatically)
        └── Pod
        └── Pod
```

You **never** create ReplicaSets or Pods directly in production. You create a **Deployment**, and it manages everything below it.

### Why Three Layers?

| Resource | Responsibility |
|---|---|
| **Deployment** | Manages the *rollout strategy* — how updates happen (rolling update, recreate). It creates and manages ReplicaSets. |
| **ReplicaSet** | Manages the *replica count* — ensures exactly N pods exist at all times. If a pod crashes, the ReplicaSet creates a new one. |
| **Pod** | The actual running container(s). Ephemeral. Cattle, not pets. |

When you update a Deployment (e.g., change the image tag), the Deployment creates a **new ReplicaSet** with the new spec, scales it up, and scales the old ReplicaSet down. This is how rolling updates work. The old ReplicaSet sticks around (with 0 replicas) so you can **rollback**.

---

## 2.3 — Deployment Manifest: Full Anatomy

```yaml
apiVersion: apps/v1          # Deployments live in the "apps" API group
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp               # Labels on the Deployment itself
    environment: staging
  annotations:
    description: "Frontend web application"   # Human-readable metadata, not used for selection
spec:
  replicas: 3                 # Desired number of Pod replicas
  
  selector:                   # HOW the Deployment finds its Pods
    matchLabels:
      app: webapp             # Must match the Pod template labels below
  
  strategy:                   # How updates are rolled out
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1             # Max pods ABOVE desired count during update
      maxUnavailable: 0       # Max pods BELOW desired count during update (zero-downtime)
  
  template:                   # The Pod template — this IS the Pod spec
    metadata:
      labels:
        app: webapp           # MUST match selector.matchLabels above
        version: v1
    spec:
      containers:
        - name: webapp
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m       # 100 millicores = 0.1 CPU
              memory: 128Mi
            limits:
              cpu: 250m
              memory: 256Mi
```

### Critical Detail: The Label-Selector Contract

```
spec.selector.matchLabels  ←——MUST MATCH——→  spec.template.metadata.labels
```

The Deployment uses `selector.matchLabels` to find which Pods belong to it. If these don't match the Pod template labels, the Deployment will create Pods it can't find, leading to infinite Pod creation. Kubernetes will actually reject the manifest if they don't match.

---

## 2.4 — Labels, Selectors, and Annotations

### Labels (Key-Value Pairs for Selection)

Labels are how Kubernetes objects find and group each other. They're the plumbing behind Deployments, Services, and everything else.

```yaml
metadata:
  labels:
    app: webapp          # What application is this?
    tier: frontend       # What layer? (frontend, backend, database)
    environment: prod    # What environment?
    version: v2.1.0      # What version?
```

```powershell
# Filter pods by label
kubectl get pods -l app=webapp

# Multiple label selectors (AND logic)
kubectl get pods -l app=webapp,tier=frontend

# Inequality selectors
kubectl get pods -l 'environment!=prod'

# Set-based selectors
kubectl get pods -l 'tier in (frontend,backend)'

# Show labels as columns
kubectl get pods --show-labels

# Add a label to an existing resource
kubectl label pod my-pod team=platform

# Remove a label (note the minus sign)
kubectl label pod my-pod team-

# Overwrite an existing label
kubectl label pod my-pod version=v2 --overwrite
```

### Annotations (Metadata for Humans and Tools)

Annotations are NOT used for selection. They store non-identifying metadata — build info, git commit SHAs, monitoring config, etc.

```yaml
metadata:
  annotations:
    git.commit: "a1b2c3d4"
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    kubectl.kubernetes.io/last-applied-configuration: "{...}"  # Added automatically by kubectl apply
```

> [!NOTE]
> **Labels vs. Annotations rule of thumb**: If Kubernetes needs to use it for selection or grouping → **Label**. If it's informational metadata for humans or external tools → **Annotation**.

---

## 2.5 — The Reconciliation Loop in Action

Let's see the Deployment controller in action. Create the deployment:

```powershell
kubectl apply -f deployment.yaml
```

Now watch what happens:

```powershell
# See the Deployment
kubectl get deployment webapp
# NAME     READY   UP-TO-DATE   AVAILABLE   AGE
# webapp   3/3     3            3           30s

# See the ReplicaSet it created (note the random hash suffix)
kubectl get replicaset
# NAME                DESIRED   CURRENT   READY   AGE
# webapp-7d9f8b6c5f   3         3         3       30s

# See the Pods it created (Deployment name + ReplicaSet hash + Pod hash)
kubectl get pods
# NAME                      READY   STATUS    RESTARTS   AGE
# webapp-7d9f8b6c5f-abc12   1/1     Running   0          30s
# webapp-7d9f8b6c5f-def34   1/1     Running   0          30s
# webapp-7d9f8b6c5f-ghi56   1/1     Running   0          30s
```

Now **kill a Pod** and watch the reconciliation:

```powershell
# Delete one pod
kubectl delete pod webapp-7d9f8b6c5f-abc12

# Immediately check pods again
kubectl get pods
# A NEW pod has been created! The ReplicaSet detected actual (2) ≠ desired (3) and fixed it.
# webapp-7d9f8b6c5f-jkl78   1/1     Running   0          5s   ← brand new
# webapp-7d9f8b6c5f-def34   1/1     Running   0          2m
# webapp-7d9f8b6c5f-ghi56   1/1     Running   0          2m
```

This is the power of declarative state. You didn't tell Kubernetes to restart anything. The controller saw the drift and corrected it automatically.

---

## 2.6 — Scaling Workloads

### Manual Scaling

```powershell
# Scale to 5 replicas (imperative, but acceptable for operational tasks)
kubectl scale deployment webapp --replicas=5

# Verify
kubectl get deployment webapp
# NAME     READY   UP-TO-DATE   AVAILABLE   AGE
# webapp   5/5     5            5           5m

# Scale down to 2
kubectl scale deployment webapp --replicas=2
```

### Declarative Scaling

The proper way — change `replicas` in your YAML and re-apply:

```yaml
spec:
  replicas: 5    # Changed from 3 to 5
```

```powershell
kubectl apply -f deployment.yaml
```

> [!TIP]
> In production, you'd use a **Horizontal Pod Autoscaler (HPA)** to scale based on CPU/memory metrics. We'll touch on resource management in Phase 6. For now, understand manual scaling.

---

## 2.7 — Rolling Updates

This is where Deployments earn their keep. When you update the Pod template (e.g., change the image), the Deployment performs a **rolling update** — gradually replacing old Pods with new ones.

### Triggering an Update

```powershell
# Option 1: Edit the YAML and re-apply (PREFERRED)
# Change image from nginx:1.27 to nginx:1.28 in your manifest
kubectl apply -f deployment.yaml

# Option 2: Imperative image update (quick for testing)
kubectl set image deployment/webapp webapp=nginx:1.28
```

### Watching the Rollout

```powershell
# Watch the rollout happen in real-time
kubectl rollout status deployment/webapp
# Waiting for deployment "webapp" rollout to finish: 1 out of 3 new replicas have been updated...
# Waiting for deployment "webapp" rollout to finish: 2 out of 3 new replicas have been updated...
# deployment "webapp" successfully rolled out

# See the ReplicaSets — old one has 0 replicas, new one has 3
kubectl get replicasets
# NAME                DESIRED   CURRENT   READY   AGE
# webapp-7d9f8b6c5f   0         0         0       10m    ← old (nginx:1.27)
# webapp-5c8a4e2d1b   3         3         3       30s    ← new (nginx:1.28)
```

### How Rolling Updates Work (With Your Strategy Config)

Given this strategy:
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # Can temporarily have 4 pods (3 + 1)
    maxUnavailable: 0    # Must always have at least 3 ready pods
```

The sequence is:
1. Create 1 new Pod (now 4 total: 3 old + 1 new)
2. Wait for new Pod to be Ready
3. Terminate 1 old Pod (back to 3 total: 2 old + 1 new)
4. Create 1 new Pod (4 total: 2 old + 2 new)
5. Wait, terminate, repeat until all Pods are new

This gives you **zero-downtime deployments**.

### The Alternative: Recreate Strategy

```yaml
strategy:
  type: Recreate    # Kill all old pods first, then create all new pods
```

This causes downtime but is necessary for apps that can't run two versions simultaneously (e.g., database schema migrations).

---

## 2.8 — Rollbacks

Every update creates a new ReplicaSet. The old ones are kept (by default, the last 10 — controlled by `spec.revisionHistoryLimit`). This gives you instant rollback capability.

```powershell
# View rollout history
kubectl rollout history deployment/webapp
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>

# See details of a specific revision
kubectl rollout history deployment/webapp --revision=1

# ROLLBACK to the previous revision
kubectl rollout undo deployment/webapp

# Rollback to a specific revision
kubectl rollout undo deployment/webapp --to-revision=1

# Verify the rollback
kubectl rollout status deployment/webapp
kubectl get replicasets
```

> [!TIP]
> **Pro tip**: Add `--record` to your `kubectl apply` commands (deprecated but still works) or use annotations to track what changed in each revision:
> ```powershell
> kubectl annotate deployment/webapp kubernetes.io/change-cause="Updated to nginx:1.28 for CVE fix"
> ```
> This makes `rollout history` actually useful because you can see *why* each revision was created.

### Pausing and Resuming Rollouts

If you need to make multiple changes without triggering a rollout for each one:

```powershell
# Pause the deployment — changes will be batched
kubectl rollout pause deployment/webapp

# Make multiple changes
kubectl set image deployment/webapp webapp=nginx:1.28
kubectl set resources deployment/webapp -c webapp --limits=cpu=500m,memory=512Mi

# Resume — triggers a single rollout with all changes
kubectl rollout resume deployment/webapp
```

---

## Phase 2 — kubectl Cheat Sheet

| Command | Purpose |
|---|---|
| `kubectl apply -f <file>` | Apply/update a resource declaratively |
| `kubectl get deployment <name>` | View deployment status |
| `kubectl get replicasets` | List ReplicaSets (see rollout history visually) |
| `kubectl get pods --show-labels` | List pods with their labels |
| `kubectl get pods -l <key>=<value>` | Filter pods by label |
| `kubectl label <resource> <name> <key>=<value>` | Add/update a label |
| `kubectl label <resource> <name> <key>-` | Remove a label |
| `kubectl scale deployment <name> --replicas=N` | Scale replicas imperatively |
| `kubectl set image deployment/<name> <container>=<image>` | Update container image imperatively |
| `kubectl rollout status deployment/<name>` | Watch rollout progress |
| `kubectl rollout history deployment/<name>` | View revision history |
| `kubectl rollout history deployment/<name> --revision=N` | Inspect a specific revision |
| `kubectl rollout undo deployment/<name>` | Rollback to previous revision |
| `kubectl rollout undo deployment/<name> --to-revision=N` | Rollback to specific revision |
| `kubectl rollout pause deployment/<name>` | Pause rollout (batch changes) |
| `kubectl rollout resume deployment/<name>` | Resume a paused rollout |
| `kubectl annotate deployment/<name> key="value"` | Add annotation to a resource |
| `kubectl describe deployment <name>` | Full details + events |
| `kubectl diff -f <file>` | Preview changes before applying |

> [!TIP]
> `kubectl diff -f deployment.yaml` is extremely powerful. It shows you exactly what will change *before* you apply. Use it religiously in production. Think of it as `terraform plan` for Kubernetes.

---

## 🔬 Lab Exercise 2: Deploy, Scale, Update, and Rollback a Web Application

### Objective

Go through the full lifecycle of a Deployment: create → scale → update → rollback.

### Step 1: Create the Initial Deployment

Write a file called `lab2-deployment.yaml` with the following requirements:

- **Kind**: Deployment
- **Name**: `lab2-webapp`
- **Replicas**: 2
- **Labels on the Deployment**: `app: lab2-webapp`, `environment: dev`
- **Pod template labels**: `app: lab2-webapp`, `version: v1`
- **Container name**: `web`
- **Image**: `nginx:1.26` (we'll update this later)
- **Port**: 80
- **Strategy**: RollingUpdate with `maxSurge: 1` and `maxUnavailable: 0`
- **Annotation on the Deployment**: `team: "platform-engineering"`

Apply it:
```powershell
kubectl apply -f lab2-deployment.yaml
```

### Step 2: Verify the Hierarchy

Run these three commands and capture the output:
```powershell
kubectl get deployment lab2-webapp
kubectl get replicasets -l app=lab2-webapp
kubectl get pods -l app=lab2-webapp --show-labels
```

Observe how the naming flows: `Deployment → ReplicaSet (+ hash) → Pod (+ hash)`.

### Step 3: Scale Up

Scale to **5 replicas** using the imperative command:
```powershell
kubectl scale deployment lab2-webapp --replicas=5
kubectl get pods -l app=lab2-webapp
```

### Step 4: Perform a Rolling Update

Update the image from `nginx:1.26` to `nginx:1.27`. Do this **declaratively** — edit your YAML file, change the image tag, and re-apply:

```powershell
kubectl apply -f lab2-deployment.yaml
kubectl rollout status deployment/lab2-webapp
```

After the rollout completes, check:
```powershell
kubectl get replicasets -l app=lab2-webapp
```

You should see **two** ReplicaSets: the old one (0 replicas) and the new one (5 replicas).

### Step 5: Rollback

Roll back to the previous version:
```powershell
kubectl rollout undo deployment/lab2-webapp
kubectl rollout status deployment/lab2-webapp
kubectl get replicasets -l app=lab2-webapp
```

Verify the old ReplicaSet is back to 5 replicas and the pods are running `nginx:1.26` again:
```powershell
kubectl describe pod <any-pod-name> | findstr "Image:"
```

### What to Submit

Paste the following for my review:
1. Your complete `lab2-deployment.yaml` manifest
2. Output of `kubectl get deployment lab2-webapp` (after initial apply)
3. Output of `kubectl get replicasets -l app=lab2-webapp` (after the rolling update — showing 2 ReplicaSets)
4. Output of `kubectl rollout history deployment/lab2-webapp`
5. Output confirming the rollback succeeded (the image is back to `nginx:1.26`)

---

> **⏸️ CHECKPOINT**: Submit your lab results or request to skip. Once confirmed, we move to **Phase 3: Service Discovery & Cluster Networking** — where your Pods finally learn to talk to each other and the outside world.

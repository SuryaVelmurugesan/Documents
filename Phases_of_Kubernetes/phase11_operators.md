# Phase 11: Operators — Self-Managing Applications

> **Goal**: Understand how CRDs come alive. An Operator is a controller that watches your custom resources and automatically manages the underlying infrastructure — creating Deployments, Services, databases, backups, failovers — encoding your operational knowledge into code.

---

## 11.1 — The Operator Pattern: CRD + Controller

In Phase 10, you created CRDs and Custom Resources. But nothing *happened* when you applied them — no Pods were created, no Services appeared. The CRD just stored data in etcd. To make CRDs actionable, you need a **controller**.

```
CRD (Schema)  +  Controller (Logic)  =  Operator
```

### What Makes an Operator?

An Operator is a Kubernetes-native application that:
1. **Watches** Custom Resources via the API server
2. **Compares** desired state (`.spec`) with actual state (running resources)
3. **Reconciles** — creates, updates, or deletes Kubernetes resources to match the desired state
4. **Reports** status back to `.status`
5. **Encodes domain expertise** — handles complex operational tasks (backups, failovers, upgrades, scaling) that would normally require a human DBA/SRE

### The Human Operator Analogy

```
Without Operator (Human DBA):
  Developer: "I need a PostgreSQL cluster with 3 replicas and daily backups"
  DBA: Creates StatefulSet, configures replication, sets up CronJob for backups,
       monitors health, performs failovers, manages upgrades manually
  Time: Hours/days, error-prone

With Operator (Automated):
  Developer: kubectl apply -f postgres-cluster.yaml
  Operator: Creates StatefulSet, configures replication, schedules backups,
            monitors health, auto-failovers, manages rolling upgrades
  Time: Seconds, repeatable, self-healing
```

---

## 11.2 — The Reconciliation Loop (Deep Dive)

Every operator runs a continuous **reconciliation loop**. This is the heart of the Operator pattern:

```
                    ┌─────────────────────────────────┐
                    │                                 │
                    ▼                                 │
        ┌──────────────────┐                          │
        │  Watch for Events │  ← API Server notifies  │
        │  (Create/Update/  │    on CR changes         │
        │   Delete of CR)   │                          │
        └────────┬─────────┘                          │
                 │                                     │
                 ▼                                     │
        ┌──────────────────┐                          │
        │  Read Desired     │  ← CR's .spec            │
        │  State            │                          │
        └────────┬─────────┘                          │
                 │                                     │
                 ▼                                     │
        ┌──────────────────┐                          │
        │  Read Actual      │  ← Query real K8s        │
        │  State            │    resources              │
        └────────┬─────────┘                          │
                 │                                     │
                 ▼                                     │
        ┌──────────────────┐                          │
        │  Desired == Actual?│                         │
        │                   │                          │
        │  YES: Do nothing  │──────────────────────────┘
        │  NO:  Reconcile   │                          
        └────────┬─────────┘                          
                 │ NO                                   
                 ▼                                     
        ┌──────────────────┐                          
        │  Take Action      │  ← Create/Update/Delete  
        │  (Create Deploy,  │    real K8s resources     
        │   Update Service, │                          
        │   Scale Pods)     │                          
        └────────┬─────────┘                          
                 │                                     
                 ▼                                     
        ┌──────────────────┐                          
        │  Update CR .status│  ← Report back           
        └────────┬─────────┘                          
                 │                                     
                 └─────────────────────────────────────┘
```

### Key Properties of a Good Reconciliation Loop

| Property | Meaning |
|---|---|
| **Idempotent** | Running reconciliation multiple times with the same input produces the same result. No duplicate resources. |
| **Level-triggered** | Reacts to the current state, not events. If it missed an event, it still converges to the right state by comparing desired vs. actual. |
| **Edge-agnostic** | Doesn't care *how* the state changed — whether a user edited the CR, a Pod crashed, or a node went down. It just fixes the drift. |
| **Gradual** | Doesn't try to do everything in one pass. If it can't complete, it returns an error and retries with exponential backoff. |

---

## 11.3 — Controller Architecture Internals

### Components of a Controller

```
┌──────────────────────────────────────────────┐
│                 Controller                    │
│                                              │
│  ┌──────────┐    ┌──────────┐    ┌────────┐ │
│  │ Informer │───►│ Work     │───►│Reconcile│ │
│  │ (Watch)  │    │ Queue    │    │Function │ │
│  └──────────┘    └──────────┘    └────────┘ │
│       ▲                              │       │
│       │           API Server         │       │
│       └──────────────────────────────┘       │
└──────────────────────────────────────────────┘
```

| Component | Purpose |
|---|---|
| **Informer** | Watches the API server for changes to specific resource types. Uses a local cache + watch mechanism to be efficient (not constant polling). |
| **Work Queue** | Buffers events. Deduplicates (if 5 events arrive for the same resource, only one reconciliation runs). Rate-limits retries. |
| **Reconcile Function** | Your business logic. Receives a resource key (namespace/name), reads the CR, compares desired vs. actual, takes action. |
| **Local Cache** | Informers maintain a local cache of watched resources. Reads go to the cache (fast). Writes go to the API server. |

### Leader Election

In production, you run multiple replicas of your operator for high availability. But only ONE should be actively reconciling:

```yaml
# Operator Deployment with 2 replicas
spec:
  replicas: 2        # For HA
  # The operator framework handles leader election automatically
  # Only the leader processes events; the standby takes over if the leader dies
```

### Owner References

When an operator creates resources (Deployments, Services) for a CR, it sets **owner references** on them:

```yaml
# Deployment created by the operator
metadata:
  ownerReferences:
    - apiVersion: platform.example.com/v1
      kind: WebApp
      name: my-frontend
      uid: abc-123-def
      controller: true
      blockOwnerDeletion: true
```

This means:
- When the CR is deleted, all owned resources are **garbage collected** automatically
- `kubectl get deployment my-frontend-deploy -o yaml` shows who owns it
- Prevents orphaned resources

---

## 11.4 — Operator Frameworks

You don't build operators from scratch. Use a framework:

### Kubebuilder (Go — Most Popular)

```
scaffolds Go project  →  generates CRD + controller boilerplate  →  you write Reconcile()
```

```powershell
# Initialize a project
kubebuilder init --domain example.com --repo github.com/myorg/webapp-operator

# Create a new API (CRD + Controller)
kubebuilder create api --group platform --version v1 --kind WebApp

# Generated structure:
# api/v1/webapp_types.go       ← Define your CR struct (spec, status)
# controllers/webapp_controller.go ← Write your Reconcile() logic
# config/crd/                   ← Auto-generated CRD YAML
# config/rbac/                  ← Auto-generated RBAC
```

The Reconcile function in Go:

```go
func (r *WebAppReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := r.Log.WithValues("webapp", req.NamespacedName)

    // 1. Fetch the CR
    webapp := &platformv1.WebApp{}
    if err := r.Get(ctx, req.NamespacedName, webapp); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 2. Check if Deployment exists
    deployment := &appsv1.Deployment{}
    err := r.Get(ctx, types.NamespacedName{
        Name:      webapp.Name + "-deployment",
        Namespace: webapp.Namespace,
    }, deployment)

    if errors.IsNotFound(err) {
        // 3. Create the Deployment
        dep := r.buildDeployment(webapp)
        if err := controllerutil.SetControllerReference(webapp, dep, r.Scheme); err != nil {
            return ctrl.Result{}, err
        }
        if err := r.Create(ctx, dep); err != nil {
            return ctrl.Result{}, err
        }
        log.Info("Created Deployment", "name", dep.Name)
    }

    // 4. Update status
    webapp.Status.ReadyReplicas = deployment.Status.ReadyReplicas
    webapp.Status.Phase = "Running"
    if err := r.Status().Update(ctx, webapp); err != nil {
        return ctrl.Result{}, err
    }

    return ctrl.Result{}, nil
}
```

### Operator SDK (Go, Ansible, or Helm — Red Hat)

Builds on Kubebuilder but adds:
- **Helm-based operators**: Wrap an existing Helm chart as an operator (no Go code)
- **Ansible-based operators**: Write reconciliation as Ansible playbooks
- OLM (Operator Lifecycle Manager) integration

```powershell
# Helm-based operator (easiest path)
operator-sdk init --plugins helm --domain example.com
operator-sdk create api --group platform --version v1 --kind WebApp --helm-chart ./mychart
```

### Kopf (Python)

For Python developers. Simpler API, less boilerplate:

```python
import kopf
import kubernetes

@kopf.on.create('platform.example.com', 'v1', 'webapps')
def on_create(spec, name, namespace, **kwargs):
    """Called when a WebApp CR is created."""
    
    api = kubernetes.client.AppsV1Api()
    
    # Build the Deployment
    deployment = kubernetes.client.V1Deployment(
        metadata=kubernetes.client.V1ObjectMeta(name=f"{name}-deployment"),
        spec=kubernetes.client.V1DeploymentSpec(
            replicas=spec.get('replicas', 1),
            selector=kubernetes.client.V1LabelSelector(
                match_labels={"app": name}
            ),
            template=kubernetes.client.V1PodTemplateSpec(
                metadata=kubernetes.client.V1ObjectMeta(labels={"app": name}),
                spec=kubernetes.client.V1PodSpec(
                    containers=[kubernetes.client.V1Container(
                        name="app",
                        image=spec['image'],
                        ports=[kubernetes.client.V1ContainerPort(
                            container_port=spec.get('port', 80)
                        )]
                    )]
                )
            )
        )
    )
    
    # Create it
    api.create_namespaced_deployment(namespace, deployment)
    
    return {'message': f'Deployment {name}-deployment created'}

@kopf.on.update('platform.example.com', 'v1', 'webapps')
def on_update(spec, name, namespace, **kwargs):
    """Called when a WebApp CR is updated."""
    api = kubernetes.client.AppsV1Api()
    
    # Patch the deployment with new spec
    patch = {"spec": {"replicas": spec.get('replicas', 1)}}
    api.patch_namespaced_deployment(f"{name}-deployment", namespace, patch)

@kopf.on.delete('platform.example.com', 'v1', 'webapps')
def on_delete(name, namespace, **kwargs):
    """Called when a WebApp CR is deleted."""
    api = kubernetes.client.AppsV1Api()
    api.delete_namespaced_deployment(f"{name}-deployment", namespace)
```

### Shell-Operator (Bash/Shell)

For the simplest use cases — write operators as shell scripts:

```bash
#!/bin/bash
# hook.sh — triggered when a WebApp CR changes

if [ "$1" == "--config" ]; then
    cat <<EOF
{
  "configVersion": "v1",
  "kubernetes": [{
    "apiVersion": "platform.example.com/v1",
    "kind": "WebApp",
    "executeHookOnEvent": ["Added", "Modified", "Deleted"]
  }]
}
EOF
    exit 0
fi

# Process the event
EVENT=$(cat)
NAME=$(echo "$EVENT" | jq -r '.[0].object.metadata.name')
IMAGE=$(echo "$EVENT" | jq -r '.[0].object.spec.image')
REPLICAS=$(echo "$EVENT" | jq -r '.[0].object.spec.replicas')
TYPE=$(echo "$EVENT" | jq -r '.[0].type')

if [ "$TYPE" == "Event" ]; then
    kubectl create deployment "${NAME}-deploy" --image="$IMAGE" --replicas="$REPLICAS" || \
    kubectl set image deployment/"${NAME}-deploy" app="$IMAGE" && \
    kubectl scale deployment/"${NAME}-deploy" --replicas="$REPLICAS"
fi
```

### Framework Comparison

| Framework | Language | Complexity | Best For |
|---|---|---|---|
| **Kubebuilder** | Go | High | Production-grade operators with complex logic |
| **Operator SDK** | Go/Ansible/Helm | Medium-High | Red Hat ecosystem, OLM integration |
| **Kopf** | Python | Medium | Python teams, rapid prototyping |
| **Shell-Operator** | Bash | Low | Simple automation, glue code |
| **Metacontroller** | Any (webhooks) | Low-Medium | Language-agnostic, webhook-based |

---

## 11.5 — Operator Maturity Model

The Operator Framework defines 5 levels of maturity:

```
Level 5: Auto Pilot
    ▲   Automatic scaling, tuning, anomaly detection
    │
Level 4: Deep Insights
    │   Metrics, alerts, log processing, workload analysis
    │
Level 3: Full Lifecycle
    │   App upgrades, backup/restore, failure recovery
    │
Level 2: Seamless Upgrades
    │   Patch and minor version upgrades
    │
Level 1: Basic Install
    │   Automated deployment and configuration
    ▼
```

| Level | Capability | Example |
|---|---|---|
| **1 — Basic Install** | Deploy the application with custom config | Helm-based operator that creates Deployments |
| **2 — Seamless Upgrades** | Handle version upgrades without downtime | Operator performs rolling update of DB, migrates schema |
| **3 — Full Lifecycle** | Backup, restore, failover, disaster recovery | PostgreSQL operator takes daily backups, auto-failover on primary failure |
| **4 — Deep Insights** | Export metrics, configure alerts, analyze workload | Operator exposes Prometheus metrics, creates ServiceMonitors |
| **5 — Auto Pilot** | Automatic horizontal/vertical scaling, self-tuning, anomaly detection | Operator adjusts connection pool sizes, auto-scales read replicas based on query load |

---

## 11.6 — Real-World Operator Deep Dives

### cert-manager (Level 3)

```yaml
# User creates this:
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-tls
spec:
  secretName: my-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - myapp.example.com
```

**What the operator does:**
1. Sees the Certificate CR
2. Contacts Let's Encrypt via ACME protocol
3. Solves the HTTP-01 or DNS-01 challenge
4. Obtains the TLS certificate
5. Stores cert + key in a Kubernetes Secret (`my-tls-secret`)
6. Auto-renews 30 days before expiry
7. Updates `.status` with certificate details and expiry date

### Prometheus Operator (Level 4)

```yaml
# User creates this:
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: metrics
      interval: 15s
```

**What the operator does:**
1. Watches ServiceMonitor CRs
2. Auto-configures Prometheus scrape targets
3. No manual prometheus.yml editing
4. Discovers new targets as Services/Pods are created
5. Manages Prometheus StatefulSet lifecycle (upgrades, scaling)

### CloudNativePG (Level 5)

```yaml
# User creates this:
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: my-database
spec:
  instances: 3
  storage:
    size: 50Gi
  backup:
    barmanObjectStore:
      destinationPath: s3://my-backups/
  postgresql:
    parameters:
      max_connections: "200"
```

**What the operator does:**
1. Creates a 3-node PostgreSQL cluster (1 primary, 2 replicas)
2. Configures streaming replication
3. Manages automated failover (primary dies → replica promoted in seconds)
4. Schedules continuous WAL archiving to S3
5. Handles rolling upgrades (minor + major PostgreSQL versions)
6. Auto-tunes PostgreSQL parameters based on available resources
7. Exports metrics to Prometheus
8. Manages connection pooling via PgBouncer sidecar

---

## 11.7 — Operator Anti-Patterns

| Anti-Pattern | Why It's Bad | Better Approach |
|---|---|---|
| **God Operator** | One operator managing 20+ different CRDs | Split into focused operators, one per domain |
| **Non-idempotent reconciliation** | Creates duplicates on re-reconciliation | Always check if resource exists before creating |
| **Missing owner references** | Deleting CR leaves orphaned Deployments/Services | Set `controllerutil.SetControllerReference()` on all created resources |
| **Reconcile does too much** | 500-line reconcile function that handles everything | Break into sub-reconcilers, one per managed resource |
| **No status updates** | Users can't see what the operator is doing | Always update `.status` with phase, conditions, messages |
| **No finalizers** | External resources (cloud DBs, DNS records) are leaked on CR deletion | Add a finalizer, clean up external resources before removing it |
| **Watching the world** | Informers on every resource type | Only watch resources your operator manages |

### Finalizers Pattern

```yaml
# CR with a finalizer
metadata:
  finalizers:
    - cleanup.platform.example.com/resources
```

```
User deletes CR
       ↓
API Server adds deletionTimestamp but doesn't remove the CR yet
       ↓
Operator sees deletionTimestamp, runs cleanup logic
(delete external DNS record, remove cloud database, etc.)
       ↓
Operator removes the finalizer from the CR
       ↓
API Server now deletes the CR from etcd
```

Without finalizers, `kubectl delete` would immediately remove the CR, and the operator would never get a chance to clean up external resources.

---

## 11.8 — Operator Lifecycle Manager (OLM)

OLM manages the lifecycle of operators themselves:

```
Install operator → Upgrade operator → Manage dependencies → Handle permissions
```

```powershell
# Install OLM (if not already present)
kubectl apply -f https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.28.0/crds.yaml
kubectl apply -f https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.28.0/olm.yaml

# Browse available operators
kubectl get packagemanifests

# Install an operator via OLM
kubectl apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: my-operator
  namespace: operators
spec:
  channel: stable
  name: my-operator
  source: operatorhubio-catalog
  sourceNamespace: olm
EOF
```

### OperatorHub.io

[OperatorHub.io](https://operatorhub.io) is the public registry of operators. Browse, search, and install production-ready operators for databases, monitoring, networking, security, and more.

---

## Phase 11 — Cheat Sheet

| Command | Purpose |
|---|---|
| `kubectl get crd` | List CRDs (see what operators installed) |
| `kubectl get <custom-resource>` | List instances of operator-managed resources |
| `kubectl describe <cr> <name>` | See CR status + operator-written conditions |
| `kubectl logs -l control-plane=controller-manager -n <operator-ns>` | Operator controller logs |
| `kubectl get events --field-selector reason=ReconcileError` | Operator reconciliation errors |
| `kubectl api-resources --api-group=<group>` | See all resources in an API group |
| `kubectl get pods -n <operator-namespace>` | Verify operator pod is running |

---

## 🔬 Lab Exercise 11: Build a Simple Operator With a Shell Script

### Objective

Create a minimal operator that watches `WebApp` CRs (from Phase 10) and automatically creates a Deployment and Service for each one.

### Approach

We'll use a simple **watch + reconcile loop** in a shell script, running as a Pod inside the cluster. This demonstrates the operator pattern without requiring Go or Python tooling.

### Step 1: Ensure the CRD Exists

Use the CRD from Lab 10, or create a simplified version:

```yaml
# lab11-crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: webapps.lab.example.com
spec:
  group: lab.example.com
  names:
    plural: webapps
    singular: webapp
    kind: WebApp
    shortNames: [wa]
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: [image, replicas]
              properties:
                image:
                  type: string
                replicas:
                  type: integer
                  minimum: 1
                  default: 1
                port:
                  type: integer
                  default: 80
      subresources:
        status: {}
      additionalPrinterColumns:
        - name: Image
          type: string
          jsonPath: .spec.image
        - name: Replicas
          type: integer
          jsonPath: .spec.replicas
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
```

### Step 2: Create the RBAC

Create `lab11-rbac.yaml`:
- **ServiceAccount**: `webapp-operator`
- **ClusterRole** with permissions to:
  - `webapps` (get, list, watch, update/status)
  - `deployments` (get, list, create, update, delete)
  - `services` (get, list, create, update, delete)
- **ClusterRoleBinding** binding the SA to the ClusterRole

### Step 3: Create the Operator Script as a ConfigMap

Create `lab11-operator-script.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: operator-script
data:
  reconcile.sh: |
    #!/bin/sh
    echo "WebApp Operator starting..."
    
    while true; do
      # Get all WebApp CRs
      WEBAPPS=$(kubectl get webapps -o json 2>/dev/null)
      
      echo "$WEBAPPS" | jq -c '.items[]' 2>/dev/null | while read -r item; do
        NAME=$(echo "$item" | jq -r '.metadata.name')
        NAMESPACE=$(echo "$item" | jq -r '.metadata.namespace')
        IMAGE=$(echo "$item" | jq -r '.spec.image')
        REPLICAS=$(echo "$item" | jq -r '.spec.replicas')
        PORT=$(echo "$item" | jq -r '.spec.port // 80')
        
        echo "Reconciling WebApp: $NAME (image=$IMAGE, replicas=$REPLICAS, port=$PORT)"
        
        # Check if Deployment exists
        if ! kubectl get deployment "${NAME}-deploy" -n "$NAMESPACE" > /dev/null 2>&1; then
          echo "Creating Deployment for $NAME"
          kubectl create deployment "${NAME}-deploy" \
            --image="$IMAGE" \
            --replicas="$REPLICAS" \
            -n "$NAMESPACE"
          
          # Expose it as a Service
          kubectl expose deployment "${NAME}-deploy" \
            --port="$PORT" \
            --target-port="$PORT" \
            --name="${NAME}-svc" \
            -n "$NAMESPACE" 2>/dev/null || true
        else
          # Update replicas if changed
          CURRENT=$(kubectl get deployment "${NAME}-deploy" -n "$NAMESPACE" -o jsonpath='{.spec.replicas}')
          if [ "$CURRENT" != "$REPLICAS" ]; then
            echo "Scaling $NAME from $CURRENT to $REPLICAS"
            kubectl scale deployment "${NAME}-deploy" --replicas="$REPLICAS" -n "$NAMESPACE"
          fi
        fi
      done
      
      sleep 10    # Poll every 10 seconds
    done
```

### Step 4: Deploy the Operator

Create `lab11-operator-deployment.yaml`:
- A Deployment named `webapp-operator` with 1 replica
- Image: `bitnami/kubectl:latest` (has kubectl + jq)
- Command: `["/bin/sh", "/scripts/reconcile.sh"]`
- Mount the ConfigMap at `/scripts`
- Use the `webapp-operator` ServiceAccount

### Step 5: Test the Operator

```powershell
# Apply everything
kubectl apply -f lab11-crd.yaml
kubectl apply -f lab11-rbac.yaml
kubectl apply -f lab11-operator-script.yaml
kubectl apply -f lab11-operator-deployment.yaml

# Watch operator logs
kubectl logs -f -l app=webapp-operator

# Create a WebApp CR
kubectl apply -f - <<EOF
apiVersion: lab.example.com/v1
kind: WebApp
metadata:
  name: test-app
spec:
  image: nginx:1.27
  replicas: 2
  port: 80
EOF

# Watch the operator create the Deployment and Service
kubectl get deployments
kubectl get services

# Scale by updating the CR
kubectl patch wa test-app --type=merge -p '{"spec":{"replicas":5}}'

# Watch the operator scale the Deployment
kubectl get pods -w

# Delete the CR
kubectl delete wa test-app

# Note: without owner references, the Deployment and Service will remain
# A real operator would set ownerReferences and use finalizers
```

### What to Submit

1. Your `lab11-rbac.yaml` and `lab11-operator-deployment.yaml`
2. Output of `kubectl logs` from the operator showing reconciliation messages
3. Output of `kubectl get deployments` showing operator-created deployments
4. Output showing the operator scaling the deployment after you patched the CR

---

> **⏸️ CHECKPOINT**: Submit your lab results or request to skip. Next: **Phase 12: GitOps with ArgoCD** — declarative, Git-driven cluster management.

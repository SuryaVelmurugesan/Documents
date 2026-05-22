# Phase 12: GitOps with ArgoCD

> **Goal**: Make Git the single source of truth for your entire cluster. GitOps means your cluster state is version-controlled, auditable, and self-healing — if someone makes a manual change, ArgoCD detects the drift and corrects it automatically.

---

## 12.1 — What Is GitOps?

GitOps is an operational model that applies DevOps best practices (version control, code review, CI/CD) to **infrastructure and deployment**. The core idea:

**Git is the source of truth. The cluster converges to match Git. Always.**

### The Four GitOps Principles (OpenGitOps)

| Principle | Meaning |
|---|---|
| **1. Declarative** | The entire system (apps, config, infrastructure) is described declaratively. (You already do this with YAML manifests.) |
| **2. Versioned and Immutable** | The desired state is stored in Git. Every change is a commit with full history, authorship, and traceability. |
| **3. Pulled Automatically** | Software agents (ArgoCD, Flux) pull the desired state from Git and apply it. No manual `kubectl apply`. No CI pipeline pushing to the cluster. |
| **4. Continuously Reconciled** | Agents continuously compare actual cluster state with desired state in Git. Drift is detected and corrected automatically. |

### GitOps vs Traditional CI/CD

```
Traditional CI/CD (Push Model):
  Developer → Git Push → CI Pipeline → kubectl apply → Cluster
                                             ↑
                          CI has cluster credentials (security risk)
                          No drift detection after deployment

GitOps (Pull Model):
  Developer → Git Push → Git Repo ← ArgoCD watches
                                       ↓
                              ArgoCD applies to Cluster
                                       ↓
                              Continuous reconciliation
                                       ↓
                              Manual changes are reverted
```

### Why GitOps Matters

| Benefit | How |
|---|---|
| **Audit trail** | Every change is a Git commit. `git log` = deployment history. `git blame` = who deployed what. |
| **Rollback** | `git revert` = cluster rollback. No fumbling with `helm rollback` or `kubectl rollout undo`. |
| **Disaster recovery** | Cluster dies? Point ArgoCD at the same Git repo → entire cluster rebuilt. |
| **Drift detection** | Someone does `kubectl edit` directly? ArgoCD detects the diff and can auto-revert. |
| **Code review for infra** | Deployment = pull request. Team reviews before merge. Merge = deploy. |
| **Multi-cluster** | Same Git repo, multiple clusters. Change once, deploy everywhere. |

---

## 12.2 — ArgoCD Architecture

ArgoCD is the most popular GitOps tool for Kubernetes. It runs inside your cluster and continuously syncs it with Git.

```
┌──────────────────────────────────────────────────────┐
│                    ArgoCD                              │
│                                                        │
│  ┌─────────────┐   ┌──────────────┐   ┌────────────┐ │
│  │  API Server  │   │  Repo Server  │   │ Application│ │
│  │  (UI + API)  │   │  (Git Clone)  │   │ Controller │ │
│  └──────┬──────┘   └──────┬───────┘   └─────┬──────┘ │
│         │                  │                  │        │
│         │          ┌───────┴───────┐          │        │
│         │          │    Redis      │          │        │
│         │          │   (Cache)     │          │        │
│         │          └───────────────┘          │        │
│         │                                     │        │
│  ┌──────┴─────────────────────────────────────┴──┐    │
│  │              Kubernetes API Server              │    │
│  └─────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
         ↕                    ↕
    Users/UI/CLI         Git Repository
```

| Component | Purpose |
|---|---|
| **API Server** | Exposes the ArgoCD REST API and serves the Web UI. Handles authentication (SSO, OIDC, LDAP). |
| **Repo Server** | Clones Git repos, renders Helm charts / Kustomize overlays / plain YAML, and returns the generated manifests. Stateless, cacheable. |
| **Application Controller** | The brain. Watches ArgoCD Application CRs, compares desired state (from Git) with actual state (in cluster), and performs sync operations. |
| **Redis** | Caching layer for the API server and controller. Stores repo metadata, app state cache. |
| **Dex** | Optional. Handles SSO/OIDC authentication for the Web UI. |

---

## 12.3 — Installation

### Install ArgoCD on Minikube

```powershell
# Create the namespace
kubectl create namespace argocd

# Install ArgoCD (stable release)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all pods to be ready
kubectl get pods -n argocd -w

# Expected pods:
# argocd-application-controller-0
# argocd-repo-server-xxx
# argocd-server-xxx
# argocd-redis-xxx
# argocd-dex-server-xxx
```

### Access the UI

```powershell
# Port-forward the ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Access: https://localhost:8080
# Username: admin
# Password: retrieve the initial password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
```

### Install the ArgoCD CLI

```powershell
# Windows — using winget
winget install argoproj.argocd

# Or Chocolatey
choco install argocd-cli

# Login
argocd login localhost:8080 --insecure

# Change the admin password
argocd account update-password
```

---

## 12.4 — The Application CRD

ArgoCD extends Kubernetes with its own CRDs. The primary one is `Application` — it defines which Git repo to sync with which cluster/namespace.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-webapp
  namespace: argocd              # Applications MUST live in the argocd namespace
spec:
  project: default               # ArgoCD project (for RBAC/grouping)

  source:
    repoURL: https://github.com/myorg/k8s-manifests.git
    targetRevision: main          # Branch, tag, or commit SHA
    path: apps/webapp             # Directory containing manifests

  destination:
    server: https://kubernetes.default.svc   # Target cluster (in-cluster)
    namespace: production         # Target namespace

  syncPolicy:
    automated:                    # Auto-sync when Git changes
      prune: true                 # Delete resources removed from Git
      selfHeal: true              # Revert manual cluster changes
    syncOptions:
      - CreateNamespace=true      # Create namespace if it doesn't exist
    retry:
      limit: 5                   # Retry sync on failure
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Key Fields

| Field | Purpose |
|---|---|
| `source.repoURL` | Git repository URL (HTTPS or SSH) |
| `source.targetRevision` | Branch (`main`), tag (`v1.2.0`), or commit SHA (`a1b2c3d`) |
| `source.path` | Directory within the repo containing manifests |
| `destination.server` | Target cluster API server URL |
| `destination.namespace` | Target namespace for the resources |
| `syncPolicy.automated` | Enable auto-sync (otherwise manual sync required) |
| `syncPolicy.automated.prune` | Delete cluster resources that are no longer in Git |
| `syncPolicy.automated.selfHeal` | Revert manual changes (drift correction) |

---

## 12.5 — Source Types: What ArgoCD Can Deploy

ArgoCD doesn't just do plain YAML. It supports multiple manifest formats:

### Plain YAML Manifests

```yaml
source:
  repoURL: https://github.com/myorg/k8s-manifests.git
  path: apps/webapp              # Directory with .yaml files
  targetRevision: main
```

### Helm Charts (From Git)

```yaml
source:
  repoURL: https://github.com/myorg/helm-charts.git
  path: charts/webapp
  targetRevision: main
  helm:
    releaseName: my-webapp
    valueFiles:
      - values-production.yaml
    values: |
      replicaCount: 5
      image:
        tag: "2.0.0"
    parameters:
      - name: service.type
        value: LoadBalancer
```

### Helm Charts (From a Helm Repository)

```yaml
source:
  repoURL: https://charts.bitnami.com/bitnami    # Helm repo URL
  chart: nginx                                     # Chart name
  targetRevision: 15.0.0                           # Chart version
  helm:
    releaseName: my-nginx
    values: |
      replicaCount: 3
```

### Kustomize

```yaml
source:
  repoURL: https://github.com/myorg/k8s-manifests.git
  path: overlays/production      # Directory with kustomization.yaml
  targetRevision: main
  kustomize:
    namePrefix: prod-
    images:
      - myapp=myregistry/myapp:v2.0
```

### Jsonnet / Config Management Plugins

ArgoCD also supports Jsonnet and custom config management plugins for specialized templating tools.

---

## 12.6 — Sync Policies: Manual vs Automated

### Manual Sync (Default)

ArgoCD detects drift but waits for you to approve the sync:

```powershell
# Trigger a sync manually
argocd app sync my-webapp

# Via kubectl
kubectl patch app my-webapp -n argocd --type merge -p '{"operation":{"sync":{"revision":"main"}}}'
```

In the UI, you click **"Sync"** and optionally select which resources to sync.

### Automated Sync

```yaml
syncPolicy:
  automated:
    prune: true        # Delete orphaned resources
    selfHeal: true     # Fix drift automatically
```

| Setting | Behavior |
|---|---|
| `automated: {}` | Auto-sync on Git changes, but DON'T delete removed resources, DON'T fix drift |
| `prune: true` | If you remove a manifest from Git, ArgoCD deletes the resource from the cluster |
| `selfHeal: true` | If someone runs `kubectl edit` to change something, ArgoCD reverts it within 3 minutes |

> [!WARNING]
> **`prune: true` can be dangerous.** If you accidentally delete a file from Git, ArgoCD will delete the corresponding resource from the cluster. Use with caution in production. Some teams enable auto-sync but disable prune, requiring manual prune operations.

### Sync Windows

Restrict when syncs can happen (e.g., no deployments during business hours):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  syncWindows:
    - kind: deny
      schedule: "0 9 * * 1-5"    # Deny syncs Mon-Fri 9:00 AM
      duration: 8h               # For 8 hours (until 5:00 PM)
      clusters: ["*"]
    - kind: allow
      schedule: "0 2 * * *"      # Allow syncs at 2:00 AM daily
      duration: 2h
      manualSync: true            # But still allow manual syncs
```

---

## 12.7 — Sync Waves and Hooks

Control the **order** in which resources are synced:

### Sync Waves

Resources with lower wave numbers sync first:

```yaml
# Namespace must exist before anything else
apiVersion: v1
kind: Namespace
metadata:
  name: production
  annotations:
    argocd.argoproj.io/sync-wave: "-1"    # Wave -1: sync first

---
# ConfigMap must exist before the Deployment
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  annotations:
    argocd.argoproj.io/sync-wave: "0"     # Wave 0: sync second

---
# Deployment depends on ConfigMap
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  annotations:
    argocd.argoproj.io/sync-wave: "1"     # Wave 1: sync third

---
# Ingress depends on the Service (which depends on the Deployment)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  annotations:
    argocd.argoproj.io/sync-wave: "2"     # Wave 2: sync last
```

### Resource Hooks

Run Jobs or other resources at specific sync phases:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync           # Run BEFORE main sync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: myapp:v2
          command: ["./migrate", "--up"]
```

| Hook | When |
|---|---|
| `PreSync` | Before the sync starts |
| `Sync` | During the sync (alongside other resources) |
| `PostSync` | After all resources are synced and healthy |
| `SyncFail` | When the sync fails |
| `Skip` | Don't apply this resource during sync |

---

## 12.8 — Multi-Environment Strategy

The most common question: how do you manage dev/staging/production from one repo?

### Strategy 1: Directory-Based (Simplest)

```
repo/
├── apps/
│   ├── dev/
│   │   ├── deployment.yaml      # replicas: 1, image: nginx:latest
│   │   └── service.yaml
│   ├── staging/
│   │   ├── deployment.yaml      # replicas: 2, image: nginx:1.27
│   │   └── service.yaml
│   └── production/
│       ├── deployment.yaml      # replicas: 5, image: nginx:1.27
│       └── service.yaml
```

One ArgoCD Application per environment:

```yaml
# dev-app.yaml
spec:
  source:
    path: apps/dev
  destination:
    namespace: dev

# production-app.yaml
spec:
  source:
    path: apps/production
  destination:
    namespace: production
```

**Pros**: Simple, explicit. **Cons**: Duplicated YAML across environments.

### Strategy 2: Kustomize Overlays (Recommended)

```
repo/
├── base/                        # Shared base manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── dev/
│   │   ├── kustomization.yaml   # patches: replicas=1, image=latest
│   │   └── patch-replicas.yaml
│   ├── staging/
│   │   ├── kustomization.yaml   # patches: replicas=2
│   │   └── patch-replicas.yaml
│   └── production/
│       ├── kustomization.yaml   # patches: replicas=5, resources added
│       └── patch-replicas.yaml
```

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
namePrefix: prod-
patches:
  - path: patch-replicas.yaml
images:
  - name: nginx
    newTag: "1.27"
```

```yaml
# overlays/production/patch-replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 5
```

ArgoCD natively understands Kustomize — just point `source.path` to the overlay directory.

**Pros**: DRY (Don't Repeat Yourself), patches only what changes. **Cons**: Kustomize learning curve.

### Strategy 3: Helm Values Per Environment

```
repo/
├── chart/
│   ├── Chart.yaml
│   ├── templates/
│   └── values.yaml              # Defaults
├── values-dev.yaml              # Dev overrides
├── values-staging.yaml
└── values-production.yaml
```

```yaml
source:
  path: chart
  helm:
    valueFiles:
      - ../values-production.yaml
```

---

## 12.9 — ApplicationSets: Fleet Management

When you have many applications or clusters, creating individual Application CRs is tedious. **ApplicationSet** is a template that generates Applications automatically.

### Git Generator (One App Per Directory)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-apps
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/myorg/k8s-manifests.git
        revision: main
        directories:
          - path: apps/*                    # One Application per subdirectory
  template:
    metadata:
      name: '{{path.basename}}'            # Directory name becomes app name
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/k8s-manifests.git
        targetRevision: main
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

Add a new directory under `apps/` → ArgoCD automatically creates a new Application.

### List Generator (Explicit Apps/Clusters)

```yaml
spec:
  generators:
    - list:
        elements:
          - cluster: dev
            url: https://dev-cluster.example.com
            namespace: apps
          - cluster: staging
            url: https://staging-cluster.example.com
            namespace: apps
          - cluster: production
            url: https://prod-cluster.example.com
            namespace: apps
  template:
    metadata:
      name: 'webapp-{{cluster}}'
    spec:
      source:
        path: 'overlays/{{cluster}}'
      destination:
        server: '{{url}}'
        namespace: '{{namespace}}'
```

### Cluster Generator

Automatically creates an Application for every registered cluster:

```yaml
spec:
  generators:
    - clusters: {}            # All clusters registered in ArgoCD
  template:
    metadata:
      name: 'monitoring-{{name}}'
    spec:
      source:
        path: monitoring/
      destination:
        server: '{{server}}'
        namespace: monitoring
```

---

## 12.10 — ArgoCD RBAC and Projects

### Projects

Group applications and enforce policies:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-frontend
  namespace: argocd
spec:
  description: "Frontend team's applications"
  
  # Allowed source repos
  sourceRepos:
    - https://github.com/myorg/frontend-*

  # Allowed destinations
  destinations:
    - namespace: frontend-*
      server: https://kubernetes.default.svc
    - namespace: staging-frontend
      server: https://kubernetes.default.svc

  # Allowed resource types (whitelist)
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace
  namespaceResourceWhitelist:
    - group: "apps"
      kind: Deployment
    - group: ""
      kind: Service
    - group: ""
      kind: ConfigMap

  # Denied resource types (blacklist)
  namespaceResourceBlacklist:
    - group: ""
      kind: Secret              # Frontend team can't create Secrets
```

### RBAC Policies

```
# argocd-rbac-cm ConfigMap
p, role:frontend-team, applications, get, team-frontend/*, allow
p, role:frontend-team, applications, sync, team-frontend/*, allow
p, role:frontend-team, applications, create, team-frontend/*, deny
g, frontend-devs, role:frontend-team
```

---

## 12.11 — Health Checks and App Status

ArgoCD monitors application health beyond just "resources exist":

| Status | Meaning |
|---|---|
| **Healthy** | All resources are running and healthy (Deployments rolled out, Pods ready) |
| **Progressing** | Resources are being updated (rolling update in progress) |
| **Degraded** | Some resources are unhealthy (Pod CrashLoopBackOff, Deployment not available) |
| **Suspended** | Application sync is paused |
| **Missing** | Resources in Git don't exist in the cluster |
| **Unknown** | ArgoCD can't determine health |

### Custom Health Checks

Define health logic for CRDs:

```lua
-- Custom health check for a WebApp CRD
hs = {}
if obj.status ~= nil then
  if obj.status.phase == "Running" then
    hs.status = "Healthy"
    hs.message = "WebApp is running"
  elseif obj.status.phase == "Failed" then
    hs.status = "Degraded"
    hs.message = "WebApp has failed"
  else
    hs.status = "Progressing"
    hs.message = "WebApp is starting"
  end
end
return hs
```

---

## 12.12 — ArgoCD CLI Reference

```powershell
# Login
argocd login <server> --insecure

# Application management
argocd app create my-app --repo <url> --path <path> --dest-server <cluster> --dest-namespace <ns>
argocd app list
argocd app get my-app
argocd app sync my-app
argocd app diff my-app                   # Preview what would change
argocd app history my-app
argocd app rollback my-app <revision>
argocd app delete my-app

# Cluster management
argocd cluster add <context-name>        # Register a cluster
argocd cluster list

# Repo management
argocd repo add <url> --username <user> --password <pass>
argocd repo add <url> --ssh-private-key-path <path>
argocd repo list

# Project management
argocd proj create <name>
argocd proj list
```

---

## 12.13 — Flux: The Alternative

**Flux** is the other major GitOps tool. It takes a different architectural approach:

| Feature | ArgoCD | Flux |
|---|---|---|
| **Architecture** | Centralized (single controller + UI) | Distributed (multiple specialized controllers) |
| **UI** | Built-in Web UI | No built-in UI (use Weave GitOps UI) |
| **CRDs** | `Application`, `AppProject`, `ApplicationSet` | `GitRepository`, `Kustomization`, `HelmRelease`, `HelmRepository` |
| **Multi-tenancy** | Projects with RBAC | Namespace-scoped controllers |
| **Sync approach** | Application-level sync | Resource-level sync |
| **Helm support** | Via Application source | Dedicated `HelmRelease` CRD |
| **Learning curve** | Lower (single concept) | Higher (multiple CRDs to learn) |
| **Community** | Larger, more tutorials | Strong CNCF backing |

> [!TIP]
> **When to choose which**: ArgoCD if you want a UI, simpler mental model, and wide community support. Flux if you want a more Kubernetes-native approach, namespace-level multi-tenancy, or deep Kustomize integration.

---

## Phase 12 — Cheat Sheet

| Command | Purpose |
|---|---|
| `kubectl apply -n argocd -f <install-manifest>` | Install ArgoCD |
| `kubectl port-forward svc/argocd-server -n argocd 8080:443` | Access ArgoCD UI |
| `argocd login localhost:8080 --insecure` | CLI login |
| `argocd app create <name> --repo <url> --path <path> --dest-server <cluster> --dest-namespace <ns>` | Create an Application |
| `argocd app list` | List all applications |
| `argocd app get <name>` | Application details |
| `argocd app sync <name>` | Trigger manual sync |
| `argocd app diff <name>` | Preview changes |
| `argocd app history <name>` | View sync history |
| `argocd app rollback <name> <id>` | Rollback to a previous sync |
| `argocd app delete <name>` | Delete application |
| `argocd cluster add <context>` | Register external cluster |
| `argocd repo add <url>` | Register a Git repository |
| `kubectl get applications -n argocd` | List ArgoCD apps via kubectl |
| `kubectl describe application <name> -n argocd` | App details via kubectl |

---

## 🔬 Lab Exercise 12: Deploy a GitOps Pipeline with ArgoCD

### Objective

Set up ArgoCD, create a Git-managed application, trigger syncs, observe self-healing, and implement a multi-environment setup.

### Step 1: Install ArgoCD

```powershell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd -w
```

### Step 2: Access the UI

```powershell
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get the initial admin password
$pass = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($pass))
```

Login at `https://localhost:8080` with username `admin`.

### Step 3: Create a Git Repository With Manifests

Create a **public** GitHub/GitLab repo (or use an existing one) called `k8s-gitops-lab` with this structure:

```
k8s-gitops-lab/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    └── dev/
        ├── kustomization.yaml
        └── patch-replicas.yaml
```

**base/deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitops-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitops-app
  template:
    metadata:
      labels:
        app: gitops-app
    spec:
      containers:
        - name: web
          image: nginx:1.27
          ports:
            - containerPort: 80
```

**base/service.yaml**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: gitops-app-svc
spec:
  selector:
    app: gitops-app
  ports:
    - port: 80
      targetPort: 80
```

**base/kustomization.yaml**:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

**overlays/dev/kustomization.yaml**:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
namePrefix: dev-
patches:
  - path: patch-replicas.yaml
```

**overlays/dev/patch-replicas.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitops-app
spec:
  replicas: 2
```

Push to Git.

### Step 4: Create the ArgoCD Application

```yaml
# lab12-argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: lab12-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-username>/k8s-gitops-lab.git
    targetRevision: main
    path: overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: lab12-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```powershell
kubectl apply -f lab12-argocd-app.yaml
```

### Step 5: Verify the Sync

```powershell
# Check the application status
kubectl get applications -n argocd
argocd app get lab12-dev

# Verify resources were created
kubectl get all -n lab12-dev
```

Visit the ArgoCD UI — you should see `lab12-dev` as Healthy and Synced.

### Step 6: Test Self-Healing

```powershell
# Manually scale the deployment (simulating drift)
kubectl scale deployment dev-gitops-app -n lab12-dev --replicas=5

# Watch ArgoCD revert it back to 2 (as specified in Git)
kubectl get pods -n lab12-dev -w
# Within ~3 minutes, it should scale back to 2
```

### Step 7: Test Git-Driven Update

Edit `overlays/dev/patch-replicas.yaml` in your Git repo — change replicas to 4. Push the commit.

```powershell
# ArgoCD auto-syncs within 3 minutes (default polling interval)
# Or trigger manually:
argocd app sync lab12-dev

# Verify
kubectl get pods -n lab12-dev
# Should see 4 pods
```

### What to Submit

1. Your ArgoCD Application manifest (`lab12-argocd-app.yaml`)
2. Screenshot or output of `argocd app get lab12-dev` showing Healthy + Synced
3. Output showing self-healing (replicas reverting after manual edit)
4. Output showing Git-driven update (replicas changing after Git push)

---

> **⏸️ CHECKPOINT**: Submit your lab results or request to skip. Next: **Phase 13: Observability** — Prometheus, Grafana, and Loki for metrics, dashboards, and logs.

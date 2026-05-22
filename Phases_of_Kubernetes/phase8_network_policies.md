# Phase 8: Network Policies — Kubernetes Firewalls

> **Goal**: Lock down pod-to-pod communication. By default, every Pod can talk to every other Pod — that's a security nightmare. Network Policies let you define firewall rules that control which Pods can communicate with which, on which ports, in which direction.

---

## 8.1 — The Default: Everything Can Talk to Everything

Out of the box, Kubernetes has a **flat, open network**. Any Pod can reach any other Pod across any namespace on any port. There are zero restrictions.

```
Pod A (frontend)  ──────►  Pod B (backend)     ✅
Pod A (frontend)  ──────►  Pod C (database)    ✅  ← This should NOT be allowed
Pod B (backend)   ──────►  Pod C (database)    ✅
Pod X (attacker)  ──────►  Pod C (database)    ✅  ← Definitely not allowed
```

In a real environment:
- The **frontend** should only talk to the **backend**
- The **backend** should only talk to the **database**
- The **database** should accept connections only from the **backend**
- Nothing else should reach the database

**Network Policies** enforce exactly this.

---

## 8.2 — Prerequisites: You Need a CNI That Supports Network Policies

> [!CAUTION]
> **Creating a NetworkPolicy resource does NOT enforce it unless your CNI plugin supports it.** The default Minikube CNI (kindnet) does **NOT** enforce Network Policies. You must use a CNI that does.

### CNI Plugins With Network Policy Support

| CNI Plugin | Supports NetworkPolicy | Notes |
|---|---|---|
| **Calico** | ✅ Yes | Most popular for on-prem and cloud. Also supports GlobalNetworkPolicy (cluster-wide). |
| **Cilium** | ✅ Yes | eBPF-based. Supports L7 policies (HTTP path/method filtering). Most advanced. |
| **Weave Net** | ✅ Yes | Simple to set up. Good for learning. |
| **Antrea** | ✅ Yes | VMware-backed, good for enterprise. |
| **Flannel** | ❌ No | Only provides networking, no policy enforcement. |
| **kindnet** | ❌ No | Minikube default. Policies are accepted but silently ignored. |

### Setting Up Calico on Minikube

```powershell
# Start Minikube with Calico CNI
minikube start --cni=calico

# Or if Minikube is already running, enable the Calico addon
minikube start --network-plugin=cni
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Verify Calico pods are running
kubectl get pods -n kube-system -l k8s-app=calico-node
# NAME                READY   STATUS    RESTARTS   AGE
# calico-node-xxxxx   1/1     Running   0          2m
```

> [!IMPORTANT]
> For all the examples and the lab in this phase, make sure you're running a CNI that enforces Network Policies. `minikube start --cni=calico` is the simplest path.

---

## 8.3 — Network Policy Anatomy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: default
spec:
  # WHO does this policy apply to?
  podSelector:
    matchLabels:
      app: backend              # Applies to all Pods with label app=backend

  # WHICH directions are controlled?
  policyTypes:
    - Ingress                   # Incoming traffic rules
    - Egress                    # Outgoing traffic rules

  # WHAT incoming traffic is allowed?
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend     # Allow traffic FROM frontend pods
      ports:
        - protocol: TCP
          port: 8080            # Only on port 8080

  # WHAT outgoing traffic is allowed?
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database     # Allow traffic TO database pods
      ports:
        - protocol: TCP
          port: 5432            # Only on port 5432
```

### The Four Key Fields

| Field | Purpose |
|---|---|
| `podSelector` | **Target**: Which Pods this policy applies to. Empty `{}` = all Pods in the namespace. |
| `policyTypes` | Which directions are controlled: `Ingress`, `Egress`, or both. |
| `ingress` | List of allowed **incoming** traffic rules. Each rule specifies `from` (sources) and `ports`. |
| `egress` | List of allowed **outgoing** traffic rules. Each rule specifies `to` (destinations) and `ports`. |

### Critical Rule: Default Deny

Once you apply a NetworkPolicy with a `policyType` to a Pod:
- **All traffic of that type is denied by default** for that Pod
- Only traffic explicitly allowed by the policy's rules is permitted

```
No NetworkPolicy on Pod    →  All traffic allowed (wide open)
NetworkPolicy with Ingress →  All INCOMING traffic denied, except what's explicitly allowed
NetworkPolicy with Egress  →  All OUTGOING traffic denied, except what's explicitly allowed
```

> [!WARNING]
> This is the #1 source of confusion. Applying a NetworkPolicy with `policyTypes: [Ingress]` but an **empty** `ingress: []` list means "deny ALL incoming traffic." The mere presence of the policyType enables deny-by-default for that direction.

---

## 8.4 — Traffic Selectors: Three Ways to Define Sources/Destinations

### 1. podSelector — By Pod Labels (Same Namespace)

```yaml
ingress:
  - from:
      - podSelector:
            matchLabels:
              app: frontend         # Allow from Pods with app=frontend in SAME namespace
```

### 2. namespaceSelector — By Namespace Labels

```yaml
ingress:
  - from:
      - namespaceSelector:
            matchLabels:
              environment: production    # Allow from ANY Pod in namespaces labeled environment=production
```

> [!TIP]
> Namespaces need labels too! Label your namespaces:
> ```powershell
> kubectl label namespace production environment=production
> kubectl label namespace monitoring purpose=monitoring
> ```

### 3. ipBlock — By CIDR Range (External Traffic)

```yaml
ingress:
  - from:
      - ipBlock:
            cidr: 10.0.0.0/8           # Allow from this IP range
            except:
              - 10.0.1.0/24            # But NOT from this subnet
```

### Combining Selectors: AND vs OR

This is subtle and critical:

#### OR Logic (Separate List Items)

```yaml
ingress:
  - from:
      - podSelector:                    # Rule 1: FROM pods with app=frontend
          matchLabels:
            app: frontend
      - namespaceSelector:              # Rule 2: OR FROM any pod in namespace labeled env=staging
          matchLabels:
            env: staging
```

Each `-` under `from:` is an **OR** condition. Traffic matching **either** rule is allowed.

#### AND Logic (Same List Item)

```yaml
ingress:
  - from:
      - podSelector:                    # Rule: FROM pods with app=frontend
          matchLabels:                  # AND in a namespace labeled env=staging
            app: frontend
        namespaceSelector:
          matchLabels:
            env: staging
```

When `podSelector` and `namespaceSelector` are in the **same list item** (no `-` before `namespaceSelector`), they're **AND** conditions. Traffic must match **both**.

> [!CAUTION]
> This single dash difference changes the entire meaning of your policy. Double-check your YAML indentation carefully.
>
> ```yaml
> # OR — two separate items (note the dash before each)
> - from:
>     - podSelector: ...
>     - namespaceSelector: ...
>
> # AND — single item combining both (note NO dash before namespaceSelector)
> - from:
>     - podSelector: ...
>       namespaceSelector: ...
> ```

---

## 8.5 — Common Policy Patterns

### Pattern 1: Default Deny All Ingress (Namespace-Wide)

The first thing you should apply to any production namespace:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}              # Apply to ALL pods in this namespace
  policyTypes:
    - Ingress                  # Deny all incoming traffic
  # No ingress rules = nothing is allowed in
```

Now nothing can reach any Pod in `production` unless another NetworkPolicy explicitly allows it.

### Pattern 2: Default Deny All Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  # No egress rules = nothing can go out
```

> [!WARNING]
> Denying all egress also blocks **DNS resolution** (port 53). Your Pods won't be able to resolve Service names. You almost always need a companion policy that allows DNS:

### Pattern 3: Allow DNS (Required When Denying Egress)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}              # All pods
  policyTypes:
    - Egress
  egress:
    - to: []                   # To anywhere
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

### Pattern 4: Default Deny Everything (The Zero-Trust Starting Point)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

Nothing in, nothing out. Then selectively add allow policies for each microservice.

### Pattern 5: Allow Ingress From Specific Pods

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend                 # Target: backend pods
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend        # Allow: from frontend pods only
      ports:
        - protocol: TCP
          port: 8080               # On port 8080 only
```

### Pattern 6: Allow Ingress From a Specific Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring-scrape
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              purpose: monitoring     # Allow from monitoring namespace
          podSelector:
            matchLabels:
              app: prometheus         # Specifically from Prometheus pods
      ports:
        - protocol: TCP
          port: 9090                  # Metrics port only
```

### Pattern 7: Allow Egress to External APIs

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: payment-service
  policyTypes:
    - Egress
  egress:
    # Allow DNS
    - ports:
        - protocol: UDP
          port: 53
    # Allow HTTPS to the external payment gateway
    - to:
        - ipBlock:
            cidr: 203.0.113.0/24      # Payment gateway IP range
      ports:
        - protocol: TCP
          port: 443
```

---

## 8.6 — How Policies Combine: The Additive (Union) Model

Multiple NetworkPolicies selecting the same Pod are **additive**. The effective policy is the **union** of all rules:

```
Policy A allows ingress from frontend on port 80
Policy B allows ingress from monitoring on port 9090

Effective: ingress from frontend:80 OR monitoring:9090
```

Policies are **never subtracted**. You cannot create a policy that "removes" an allow rule from another policy. If any policy allows the traffic, it's allowed.

> [!NOTE]
> **Order doesn't matter.** There's no priority system. All policies that select a Pod contribute to its effective access rules. This is why the pattern is: start with a default-deny, then add specific allow policies.

---

## 8.7 — Real-World Architecture: Three-Tier Isolation

Here's how you'd lock down a typical three-tier app:

```
Internet → [Ingress Controller] → Frontend → Backend → Database
```

```yaml
# 1. Default deny everything in the namespace
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: app
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]

# 2. Allow DNS for all pods
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: app
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
    - ports:
        - protocol: UDP
          port: 53

# 3. Frontend: allow ingress from Ingress Controller, allow egress to backend
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: app
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - port: 80
  egress:
    - to:
        - podSelector:
            matchLabels:
              tier: backend
      ports:
        - port: 8080

# 4. Backend: allow ingress from frontend, allow egress to database
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: app
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: frontend
      ports:
        - port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              tier: database
      ports:
        - port: 5432

# 5. Database: allow ingress from backend only, no egress needed
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: app
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: backend
      ports:
        - port: 5432
  # Egress: only DNS (inherited from allow-dns policy)
```

Result:
```
Frontend  →  Backend   ✅ (port 8080)
Frontend  →  Database  ❌ (blocked)
Backend   →  Database  ✅ (port 5432)
Database  →  Frontend  ❌ (blocked)
Random Pod → Anything  ❌ (blocked by default deny)
```

---

## 8.8 — Debugging Network Policies

### Step 1: Verify Policies Exist and Select the Right Pods

```powershell
# List all network policies
kubectl get networkpolicies
kubectl get netpol                    # Short alias

# Describe a policy — see which pods it matches
kubectl describe netpol backend-policy

# See which pods a policy selects
kubectl get pods -l app=backend       # Match against podSelector
```

### Step 2: Test Connectivity

```powershell
# Spin up a test pod and try to reach a target
kubectl run test-client --image=busybox:1.36 --rm -it --restart=Never --labels="app=frontend" -- wget -qO- --timeout=3 http://backend-svc:8080

# If it times out → traffic is blocked by a NetworkPolicy
# If it connects → traffic is allowed

# Test from a pod WITHOUT the allowed label
kubectl run test-outsider --image=busybox:1.36 --rm -it --restart=Never --labels="app=outsider" -- wget -qO- --timeout=3 http://backend-svc:8080
# Should timeout → blocked
```

### Step 3: Check if Your CNI Is Enforcing

```powershell
# Verify Calico is running
kubectl get pods -n kube-system -l k8s-app=calico-node

# Check Calico-specific policy status
kubectl get felixconfiguration -o yaml    # If using Calico

# Describe the node to check CNI
kubectl describe node minikube | findstr "CNI"
```

### Common Gotchas

| Problem | Cause | Fix |
|---|---|---|
| Policy exists but traffic isn't blocked | CNI doesn't support NetworkPolicy (Flannel, kindnet) | Switch to Calico or Cilium |
| DNS stops working after egress deny | Forgot to allow port 53/UDP | Add a DNS allow policy |
| Pod can't reach external APIs | Egress deny blocks all external traffic | Add ipBlock egress rule for the API IP range |
| Ingress from another namespace doesn't work | Namespace isn't labeled | `kubectl label namespace <ns> <key>=<value>` |
| Policy seems to have no effect | `podSelector` doesn't match any pods | Check labels with `kubectl get pods --show-labels` |

---

## Phase 8 — kubectl Cheat Sheet

| Command | Purpose |
|---|---|
| `kubectl get networkpolicies` / `kubectl get netpol` | List network policies |
| `kubectl describe netpol <name>` | Policy details + matched pods |
| `kubectl apply -f netpol.yaml` | Create/update a network policy |
| `kubectl delete netpol <name>` | Remove a network policy |
| `kubectl get pods --show-labels` | Verify pod labels match selectors |
| `kubectl label namespace <ns> <key>=<value>` | Label a namespace for namespaceSelector |
| `kubectl get pods -n kube-system -l k8s-app=calico-node` | Verify Calico is running |
| `kubectl run test --image=busybox:1.36 --rm -it --restart=Never --labels="app=X" -- wget ...` | Test connectivity with specific labels |

---

## 🔬 Lab Exercise 8: Implement Zero-Trust Networking for a Three-Tier App

### Objective

Deploy a frontend, backend, and database tier. Then implement network policies that enforce strict traffic flow: frontend → backend → database, nothing else.

### Prerequisites

```powershell
# Ensure Minikube is running with Calico
minikube start --cni=calico

# Wait for Calico to be ready
kubectl get pods -n kube-system -l k8s-app=calico-node -w
```

### Step 1: Deploy the Three Tiers

Create `lab8-app.yaml` with three Deployments and three ClusterIP Services:

**Frontend** (Deployment + Service):
- Deployment: `frontend`, 1 replica, image `hashicorp/http-echo`, args `["-text=Frontend OK", "-listen=:80"]`
- Labels: `app: lab8`, `tier: frontend`
- Service: `frontend-svc`, port 80 → targetPort 80

**Backend** (Deployment + Service):
- Deployment: `backend`, 1 replica, image `hashicorp/http-echo`, args `["-text=Backend OK", "-listen=:8080"]`
- Labels: `app: lab8`, `tier: backend`
- Service: `backend-svc`, port 8080 → targetPort 8080

**Database** (Deployment + Service):
- Deployment: `database`, 1 replica, image `hashicorp/http-echo`, args `["-text=Database OK", "-listen=:5432"]`
- Labels: `app: lab8`, `tier: database`
- Service: `database-svc`, port 5432 → targetPort 5432

```powershell
kubectl apply -f lab8-app.yaml
```

### Step 2: Verify Open Network (Before Policies)

```powershell
# Everything should work — no policies yet
kubectl run t1 --image=busybox:1.36 --rm -it --restart=Never -- wget -qO- --timeout=3 http://frontend-svc:80
kubectl run t2 --image=busybox:1.36 --rm -it --restart=Never -- wget -qO- --timeout=3 http://backend-svc:8080
kubectl run t3 --image=busybox:1.36 --rm -it --restart=Never -- wget -qO- --timeout=3 http://database-svc:5432
# All three should return OK
```

### Step 3: Apply Default Deny + DNS Allow

Create `lab8-default-deny.yaml`:
- Default deny all ingress and egress for all pods in the namespace
- Allow DNS egress (port 53 UDP/TCP) for all pods

```powershell
kubectl apply -f lab8-default-deny.yaml

# Test — everything should now be BLOCKED
kubectl run t4 --image=busybox:1.36 --rm -it --restart=Never -- wget -qO- --timeout=3 http://backend-svc:8080
# Expected: wget: download timed out
```

### Step 4: Apply Tier-Specific Allow Policies

Create `lab8-allow-policies.yaml` with three policies:

1. **Frontend policy**: Allow ingress from any Pod (simulating Ingress Controller). Allow egress to backend on port 8080.
2. **Backend policy**: Allow ingress from frontend on port 8080. Allow egress to database on port 5432.
3. **Database policy**: Allow ingress from backend on port 5432. No egress (except DNS from the default policy).

```powershell
kubectl apply -f lab8-allow-policies.yaml
```

### Step 5: Verify the Firewall Rules

```powershell
# Allowed path: frontend → backend ✅
kubectl run tf --image=busybox:1.36 --rm -it --restart=Never --labels="tier=frontend,app=lab8" -- wget -qO- --timeout=3 http://backend-svc:8080
# Expected: "Backend OK"

# Blocked path: frontend → database ❌
kubectl run tf2 --image=busybox:1.36 --rm -it --restart=Never --labels="tier=frontend,app=lab8" -- wget -qO- --timeout=3 http://database-svc:5432
# Expected: timeout

# Allowed path: backend → database ✅
kubectl run tb --image=busybox:1.36 --rm -it --restart=Never --labels="tier=backend,app=lab8" -- wget -qO- --timeout=3 http://database-svc:5432
# Expected: "Database OK"

# Blocked: random pod → anything ❌
kubectl run outsider --image=busybox:1.36 --rm -it --restart=Never --labels="app=hacker" -- wget -qO- --timeout=3 http://database-svc:5432
# Expected: timeout
```

### What to Submit

1. Your `lab8-app.yaml` (three Deployments + three Services)
2. Your `lab8-default-deny.yaml` (default deny + DNS allow)
3. Your `lab8-allow-policies.yaml` (three tier-specific policies)
4. Output of `kubectl get netpol` (listing all policies)
5. Test outputs showing allowed paths succeed and blocked paths timeout

---

> **⏸️ CHECKPOINT**: Submit your lab results or request to skip. Next: **Phase 9: Horizontal Pod Autoscaler (HPA)** — automatic scaling based on CPU, memory, and custom metrics.

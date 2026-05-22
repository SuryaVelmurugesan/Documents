# Phase 3: Service Discovery & Cluster Networking

> **Goal**: Understand how Pods communicate with each other, how Services provide stable endpoints, and how external traffic reaches your applications via Ingress.

---

## 3.1 — The Kubernetes Networking Model

Kubernetes networking follows three iron-clad rules. Every CNI (Container Network Interface) plugin must implement all three:

| Rule | What It Means |
|---|---|
| **1. Every Pod gets its own IP address** | No NAT between Pods. Pod A on Node 1 can talk directly to Pod B on Node 2 using Pod B's IP. No port mapping games. |
| **2. Pods on any node can communicate with Pods on any other node without NAT** | The network is flat. A Pod at `10.244.1.5` can reach `10.244.2.8` directly, regardless of which node they're on. |
| **3. Agents on a node (kubelet, kube-proxy) can communicate with all Pods on that node** | System components have full network access to local Pods. |

### But Wait — Pod IPs Are Ephemeral

Here's the problem: Pods are cattle. They die, they're replaced, and the replacement gets a **new IP address**. If your frontend Pod hardcodes the backend Pod's IP, it breaks the moment that backend Pod restarts.

```
Pod "backend-abc12" at 10.244.1.5  →  dies
Pod "backend-def34" at 10.244.1.9  →  new replacement
                         ↑
              Frontend is still trying 10.244.1.5 — BROKEN
```

This is the exact problem **Services** solve.

---

## 3.2 — Services: Stable Networking for Ephemeral Pods

A Service is a stable abstraction that provides:
- A **stable IP address** (ClusterIP) that never changes
- A **stable DNS name** that resolves to that IP
- **Load balancing** across all matching Pods

A Service finds its target Pods using — you guessed it — **label selectors**.

### The Basic Service Manifest

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  selector:
    app: backend          # Find all Pods with this label
    tier: api
  ports:
    - protocol: TCP
      port: 80            # Port the SERVICE listens on
      targetPort: 8080    # Port the POD's container listens on
  type: ClusterIP         # Default type
```

The traffic flow:
```
Client → backend-svc:80 (ClusterIP) → kube-proxy rules → Pod:8080
```

> [!IMPORTANT]
> **`port` vs `targetPort`**: This trips up everyone. `port` is what consumers of the Service use. `targetPort` is what your container is actually listening on. They can be different. If `targetPort` is omitted, it defaults to the value of `port`.

---

## 3.3 — Service Types: ClusterIP, NodePort, LoadBalancer

Kubernetes offers three built-in Service types, each building on the previous one:

### ClusterIP (Default)

```yaml
spec:
  type: ClusterIP
```

- Gets a virtual IP from the cluster's internal IP range (e.g., `10.96.0.1`)
- **Only reachable from inside the cluster**
- This is what 90% of your services should use — internal service-to-service communication

```powershell
kubectl get svc backend-svc
# NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
# backend-svc   ClusterIP   10.96.45.123   <none>        80/TCP    5m
```

Test it from inside the cluster:
```powershell
# Run a temporary pod to curl the service
kubectl run test-curl --image=curlimages/curl --rm -it --restart=Never -- curl http://backend-svc:80
```

### NodePort

```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080     # Optional: K8s auto-assigns from 30000-32767 if omitted
```

- Everything ClusterIP does, PLUS...
- Opens a static port on **every node** in the cluster
- External traffic can reach the service via `<any-node-ip>:<nodePort>`

```
External Client → NodeIP:30080 → Service:80 → Pod:8080
```

```powershell
kubectl get svc
# NAME          TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# backend-svc   NodePort   10.96.45.123   <none>        80:30080/TCP   5m

# With Minikube, get the accessible URL:
minikube service backend-svc --url
# http://192.168.49.2:30080
```

> [!WARNING]
> NodePort is not production-grade for external traffic. It exposes raw ports on your nodes, doesn't provide TLS termination, and requires clients to know node IPs. Use it for development/testing only.

### LoadBalancer

```yaml
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
```

- Everything NodePort does, PLUS...
- Provisions an **external load balancer** from your cloud provider (AWS ELB, GCP LB, Azure LB)
- Gets a real external IP that routes to the NodePorts

```powershell
kubectl get svc
# NAME          TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
# backend-svc   LoadBalancer   10.96.45.123   34.120.55.100   80:30080/TCP   5m
```

> [!NOTE]
> On Minikube, LoadBalancer services stay in `<pending>` for EXTERNAL-IP because there's no cloud provider. Use `minikube tunnel` in a separate terminal to simulate a load balancer:
> ```powershell
> minikube tunnel
> ```
> This assigns a routable IP to your LoadBalancer services.

### Visual Summary

```
                            ┌─────────────────────────────────┐
                            │          LoadBalancer            │
                            │  (Cloud LB → External IP)       │
                            │                                 │
                            │     ┌─────────────────────┐     │
                            │     │      NodePort        │     │
                            │     │  (Every Node:30XXX)  │     │
                            │     │                      │     │
                            │     │  ┌──────────────┐    │     │
                            │     │  │  ClusterIP   │    │     │
                            │     │  │ (Internal IP)│    │     │
                            │     │  └──────┬───────┘    │     │
                            │     │         │            │     │
                            │     └─────────┼────────────┘     │
                            └───────────────┼─────────────────-┘
                                            │
                                   ┌────────┼────────┐
                                   ▼        ▼        ▼
                                 Pod 1    Pod 2    Pod 3
```

Each type is a superset of the one below it.

---

## 3.4 — EndpointSlices: The Machinery Behind Services

When you create a Service, Kubernetes automatically creates **EndpointSlice** objects that track the IP addresses and ports of all Pods matching the Service's selector.

```powershell
# See the EndpointSlices for a service
kubectl get endpointslices -l kubernetes.io/service-name=backend-svc
# NAME                  ADDRESSTYPE   PORTS   ENDPOINTS                     AGE
# backend-svc-abc12     IPv4          8080    10.244.1.5,10.244.2.8,...     5m

# Detailed view
kubectl describe endpointslice backend-svc-abc12
```

When a Pod starts or stops, the EndpointSlice controller updates these objects. kube-proxy watches EndpointSlices and updates the iptables/IPVS rules on each node accordingly.

```
Pod labeled app=backend starts → EndpointSlice updated → kube-proxy updates iptables → traffic flows
Pod dies                       → EndpointSlice updated → kube-proxy removes rule   → no traffic to dead Pod
```

> [!NOTE]
> You'll rarely interact with EndpointSlices directly. But when a Service "isn't working," checking EndpointSlices is a critical debugging step — it tells you if the Service can actually see its Pods:
> ```powershell
> # If ENDPOINTS is empty, your selector doesn't match any Pods
> kubectl get endpoints backend-svc
> ```

---

## 3.5 — Internal DNS Resolution

Kubernetes runs an internal DNS server (CoreDNS) that automatically creates DNS records for every Service. This is how Pods find each other **by name** instead of by IP.

### DNS Format

```
<service-name>.<namespace>.svc.cluster.local
```

For a Service called `backend-svc` in the `default` namespace:

| DNS Name | Works From |
|---|---|
| `backend-svc` | Same namespace only |
| `backend-svc.default` | Any namespace |
| `backend-svc.default.svc.cluster.local` | Anywhere (fully qualified) |

### Testing DNS Resolution

```powershell
# Spin up a temporary DNS debug pod
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- nslookup backend-svc

# Expected output:
# Server:    10.96.0.10
# Address:   10.96.0.10:53
#
# Name:      backend-svc.default.svc.cluster.local
# Address:   10.96.45.123
```

### Cross-Namespace Communication

If your backend is in the `production` namespace and your frontend is in the `staging` namespace:

```yaml
# Frontend code/config would reference:
http://backend-svc.production:80
# or fully qualified:
http://backend-svc.production.svc.cluster.local:80
```

> [!TIP]
> **In your application code, always use the Service DNS name, never the Pod IP.** A Pod IP will change. `backend-svc.default.svc.cluster.local` will resolve to the correct ClusterIP forever (as long as the Service exists).

---

## 3.6 — Ingress: HTTP Routing Into the Cluster

Services (especially LoadBalancer type) work, but they have limitations:
- Each LoadBalancer Service gets its own external IP/LB — expensive at scale
- No host-based or path-based routing
- No TLS termination

**Ingress** solves this by providing a single entry point that routes HTTP/HTTPS traffic to multiple Services based on rules.

### The Two-Part System

1. **Ingress Resource** — The YAML you write. Defines routing rules (host, path → Service).
2. **Ingress Controller** — The actual reverse proxy that reads Ingress resources and configures itself. Kubernetes does NOT include one by default — you must install it.

### Installing an Ingress Controller on Minikube

```powershell
# Minikube has a built-in NGINX Ingress Controller addon
minikube addons enable ingress

# Verify it's running
kubectl get pods -n ingress-nginx
# NAME                                        READY   STATUS    RESTARTS   AGE
# ingress-nginx-controller-5c8d66c76d-xxxxx   1/1     Running   0          30s
```

### Ingress Resource Manifest

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /    # Controller-specific config
spec:
  ingressClassName: nginx                             # Which controller handles this
  rules:
    - host: myapp.local                               # Hostname to match
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc                    # Route to this Service
                port:
                  number: 80

          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-svc                     # API requests go here
                port:
                  number: 80
```

This routes:
- `http://myapp.local/` → `frontend-svc:80`
- `http://myapp.local/api` → `backend-svc:80`

### Making It Work Locally

Since `myapp.local` doesn't exist in public DNS, you need to add it to your hosts file:

```powershell
# Get the Minikube IP
minikube ip
# 192.168.49.2

# Add to C:\Windows\System32\drivers\etc\hosts (run as Administrator)
# Add this line:
# 192.168.49.2  myapp.local
```

Then test:
```powershell
curl http://myapp.local/
curl http://myapp.local/api
```

### Multiple Hosts

```yaml
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-svc
                port:
                  number: 80

    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
```

### TLS Termination

```yaml
spec:
  tls:
    - hosts:
        - myapp.local
      secretName: myapp-tls-secret    # K8s Secret containing the TLS cert and key
  rules:
    - host: myapp.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 80
```

> [!NOTE]
> We'll cover Secrets in Phase 4. For now, know that TLS certs are stored as Kubernetes Secrets and referenced by the Ingress resource.

---

## 3.7 — Putting It All Together: The Full Traffic Path

```
User's Browser
      │
      ▼
[ Ingress Controller ]  ← Single entry point, reads Ingress rules
      │
      ├─── host: app.com, path: /      → frontend-svc (ClusterIP) → Frontend Pods
      │
      └─── host: app.com, path: /api   → backend-svc  (ClusterIP) → Backend Pods
                                                │
                                                ▼
                                        [ Database Pod ]
                                        reached via db-svc (ClusterIP)
```

Key insight: **Ingress handles external-to-cluster traffic. Services handle cluster-internal traffic.** They're complementary, not competing.

---

## Phase 3 — kubectl Cheat Sheet

| Command | Purpose |
|---|---|
| `kubectl apply -f service.yaml` | Create/update a Service |
| `kubectl get svc` | List all services |
| `kubectl get svc <name> -o wide` | Service details with selector info |
| `kubectl describe svc <name>` | Full service details + endpoints |
| `kubectl get endpoints <name>` | See which Pod IPs back a Service |
| `kubectl get endpointslices -l kubernetes.io/service-name=<name>` | EndpointSlice details |
| `kubectl run tmp --image=curlimages/curl --rm -it --restart=Never -- curl <url>` | Test connectivity from inside the cluster |
| `kubectl run tmp --image=busybox:1.36 --rm -it --restart=Never -- nslookup <svc>` | Test DNS resolution |
| `minikube service <name> --url` | Get the accessible URL for a NodePort/LB service |
| `minikube tunnel` | Simulate cloud LoadBalancer (assigns external IPs) |
| `minikube addons enable ingress` | Install NGINX Ingress Controller on Minikube |
| `kubectl get ingress` | List Ingress resources |
| `kubectl describe ingress <name>` | Ingress routing details |
| `kubectl get pods -n ingress-nginx` | Verify Ingress Controller is running |
| `minikube ip` | Get the Minikube node IP (for hosts file) |
| `kubectl port-forward svc/<name> 8080:80` | Forward local port to a Service (quick testing) |

> [!TIP]
> **`kubectl port-forward`** is your best friend for quick local testing. It bypasses the need for NodePort, LoadBalancer, or Ingress entirely:
> ```powershell
> # Forward your local port 8080 to the Service's port 80
> kubectl port-forward svc/backend-svc 8080:80
> # Now visit http://localhost:8080 in your browser
> ```
> Use it during development. Don't rely on it in production.

---

## 🔬 Lab Exercise 3: Wire Up a Frontend and Backend with Services and Ingress

### Objective

Deploy a two-tier application with separate frontend and backend Deployments, expose them via Services, and route external traffic through an Ingress.

### Step 1: Deploy the Backend

Create `lab3-backend.yaml`:
- **Deployment** named `backend` with 2 replicas
- Container: `hashicorp/http-echo` with args `["-text=Hello from the Backend!"]`
  - This is a tiny Go app that returns the text on any HTTP request on port 5678
- Container port: `5678`
- Labels: `app: lab3`, `tier: backend`

Then create a **ClusterIP Service** named `backend-svc`:
- Selector: `app: lab3, tier: backend`
- Port: `80`, targetPort: `5678`

You can put both in the same YAML file, separated by `---`:

```yaml
# Deployment here
---
# Service here
```

Apply it:
```powershell
kubectl apply -f lab3-backend.yaml
```

### Step 2: Deploy the Frontend

Create `lab3-frontend.yaml`:
- **Deployment** named `frontend` with 2 replicas
- Container: `hashicorp/http-echo` with args `["-text=Hello from the Frontend!"]`
- Container port: `5678`
- Labels: `app: lab3`, `tier: frontend`

Then create a **ClusterIP Service** named `frontend-svc`:
- Selector: `app: lab3, tier: frontend`
- Port: `80`, targetPort: `5678`

Apply it:
```powershell
kubectl apply -f lab3-frontend.yaml
```

### Step 3: Verify Service-to-Service Communication

```powershell
# Test that the backend Service works from inside the cluster
kubectl run test-curl --image=curlimages/curl --rm -it --restart=Never -- curl http://backend-svc:80
# Expected: "Hello from the Backend!"

# Test the frontend
kubectl run test-curl2 --image=curlimages/curl --rm -it --restart=Never -- curl http://frontend-svc:80
# Expected: "Hello from the Frontend!"

# Test DNS resolution
kubectl run test-dns --image=busybox:1.36 --rm -it --restart=Never -- nslookup backend-svc
```

### Step 4: Create an Ingress

Enable the Ingress controller:
```powershell
minikube addons enable ingress
```

Create `lab3-ingress.yaml`:
- Route `lab3.local/` → `frontend-svc:80`
- Route `lab3.local/api` → `backend-svc:80`
- Use `ingressClassName: nginx`
- Add annotation: `nginx.ingress.kubernetes.io/rewrite-target: /`

Apply and verify:
```powershell
kubectl apply -f lab3-ingress.yaml
kubectl get ingress
kubectl describe ingress lab3-ingress
```

Update your hosts file (as Administrator):
```
<minikube-ip>  lab3.local
```

Test:
```powershell
curl http://lab3.local/
curl http://lab3.local/api
```

### What to Submit

1. Your `lab3-backend.yaml` and `lab3-frontend.yaml` manifests
2. Your `lab3-ingress.yaml` manifest
3. Output of `kubectl get svc` showing both services
4. Output of `kubectl get endpoints backend-svc` (proving the Service found its Pods)
5. Output of the curl tests (both in-cluster and through Ingress)

---

> **⏸️ CHECKPOINT**: Submit your lab results or request to skip. Next up is **Phase 4: Configuration & Secrets Externalization** — where we learn to decouple configuration from container images using ConfigMaps and Secrets.

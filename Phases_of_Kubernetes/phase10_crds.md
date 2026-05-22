# Phase 10: Custom Resource Definitions (CRDs)

> **Goal**: Extend the Kubernetes API with your own resource types. CRDs are how tools like Istio, ArgoCD, cert-manager, and Prometheus integrate natively with Kubernetes — and how you can teach Kubernetes to understand your domain-specific concepts.

---

## 10.1 — Why CRDs Exist

Kubernetes ships with built-in resources: Pods, Deployments, Services, ConfigMaps. But what if you need to manage something Kubernetes doesn't know about?

- A **Certificate** that should be auto-renewed before expiry
- A **VirtualService** that defines traffic routing rules for a service mesh
- A **PostgresCluster** that represents a managed database with replicas and backups
- A **WebApp** that bundles a Deployment + Service + Ingress into one object

You could build these with raw YAML and scripts, but CRDs let you teach Kubernetes **new nouns**. Once registered, your custom resource works exactly like a built-in one — you can `kubectl get`, `kubectl describe`, `kubectl apply`, watch, label, and RBAC-control it.

```
Before CRD:  kubectl get certificates    → error: the server doesn't have a resource type "certificates"
After CRD:   kubectl get certificates    → Lists your Certificate objects
```

---

## 10.2 — How CRDs Work Under the Hood

```
1. You apply a CRD manifest to the cluster
        ↓
2. API Server registers a new REST endpoint:
   /apis/<group>/<version>/namespaces/<ns>/<plural>
        ↓
3. You can now create Custom Resources (CRs) of that type
        ↓
4. CRs are stored in etcd just like built-in resources
        ↓
5. A controller (Operator) watches for these CRs and acts on them
```

### The Two Pieces

| Piece | What It Is | Who Creates It |
|---|---|---|
| **CRD** (Custom Resource Definition) | The *schema* — defines the new resource type (name, fields, validation rules) | Platform team / tool vendor |
| **CR** (Custom Resource) | An *instance* — an actual object of that type, with specific values | Application developer / user |

Think of it like a database:
- CRD = `CREATE TABLE` (defines the structure)
- CR = `INSERT INTO` (adds a row)

---

## 10.3 — CRD Manifest: Full Anatomy

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: webapps.platform.example.com     # MUST be: <plural>.<group>
spec:
  group: platform.example.com            # API group (your domain, reversed)

  names:
    plural: webapps                      # URL path: /apis/.../webapps
    singular: webapp                     # Used in kubectl output
    kind: WebApp                         # PascalCase — the resource Kind
    shortNames:                          # Aliases for kubectl
      - wa
    categories:                          # Group in `kubectl get all`-like commands
      - all
      - platform

  scope: Namespaced                      # "Namespaced" or "Cluster"

  versions:
    - name: v1alpha1                     # Version name
      served: true                       # Is this version served by the API?
      storage: true                      # Is this the storage version in etcd?

      schema:
        openAPIV3Schema:                 # Validation schema
          type: object
          properties:
            spec:
              type: object
              required: ["image", "replicas"]    # Required fields
              properties:
                image:
                  type: string
                  description: "Container image to deploy"
                replicas:
                  type: integer
                  description: "Number of pod replicas"
                  minimum: 1
                  maximum: 100
                  default: 1
                port:
                  type: integer
                  description: "Container port to expose"
                  default: 80
                ingress:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                      default: false
                    host:
                      type: string
                env:
                  type: array
                  items:
                    type: object
                    required: ["name", "value"]
                    properties:
                      name:
                        type: string
                      value:
                        type: string
            status:
              type: object
              properties:
                readyReplicas:
                  type: integer
                availableEndpoints:
                  type: string
                phase:
                  type: string
                  enum: ["Pending", "Running", "Failed", "Succeeded"]

      # Additional columns shown in `kubectl get webapps`
      additionalPrinterColumns:
        - name: Image
          type: string
          jsonPath: .spec.image
        - name: Replicas
          type: integer
          jsonPath: .spec.replicas
        - name: Phase
          type: string
          jsonPath: .status.phase
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp

      # Enable the /status subresource
      subresources:
        status: {}
```

### Key Fields Explained

| Field | Purpose |
|---|---|
| `group` | Your API group. Convention: reverse domain (`mycompany.com` → `mycompany.com`). Avoids collisions with other CRDs. |
| `names.kind` | The resource type name in PascalCase (e.g., `WebApp`). Used in manifests: `kind: WebApp`. |
| `names.plural` | The URL path and kubectl noun: `kubectl get webapps`. |
| `names.shortNames` | Aliases: `kubectl get wa` instead of `kubectl get webapps`. |
| `names.categories` | Grouping: `kubectl get platform` returns all resources in the "platform" category. |
| `scope` | `Namespaced` (per-namespace, like Pods) or `Cluster` (global, like Nodes). |
| `versions[].schema` | OpenAPI v3 validation. Every field that users can set must be defined here. |
| `versions[].served` | Whether this version is available via the API. Set to `false` to deprecate a version. |
| `versions[].storage` | Which version is stored in etcd. Exactly one version must be `storage: true`. |
| `additionalPrinterColumns` | Custom columns in `kubectl get` output. Uses JSONPath to extract values. |
| `subresources.status` | Enables the `/status` subresource. Allows controllers to update `.status` without modifying `.spec`. |

---

## 10.4 — Validation: OpenAPI v3 Schema

The schema enforces structure and constraints on Custom Resources. Without it, users can put anything in the manifest.

### Supported Validation Rules

```yaml
properties:
  name:
    type: string
    minLength: 3
    maxLength: 63
    pattern: "^[a-z][a-z0-9-]*$"          # Regex pattern

  replicas:
    type: integer
    minimum: 1
    maximum: 100
    default: 2                             # Default if not specified

  protocol:
    type: string
    enum: ["HTTP", "HTTPS", "gRPC"]        # Allowed values only

  tags:
    type: array
    items:
      type: string
    minItems: 1
    maxItems: 10

  config:
    type: object
    additionalProperties:                  # Free-form key-value map
      type: string

  timeout:
    type: string
    pattern: "^[0-9]+(s|m|h)$"            # e.g., "30s", "5m", "1h"

  # x-kubernetes extensions
  immutableField:
    type: string
    x-kubernetes-validations:
      - rule: "self == oldSelf"             # CEL validation (K8s 1.25+)
        message: "This field is immutable"
```

> [!TIP]
> **CEL (Common Expression Language) validations** (`x-kubernetes-validations`) are powerful. They let you write complex rules like "field A is required when field B is true" or "this field cannot be changed after creation." Available in K8s 1.25+.

---

## 10.5 — Creating and Using Custom Resources

### Step 1: Apply the CRD

```powershell
kubectl apply -f webapp-crd.yaml

# Verify it's registered
kubectl get crd webapps.platform.example.com

# Check the new API endpoint
kubectl api-resources | findstr webapp
# webapps   wa   platform.example.com/v1alpha1   true   WebApp
```

### Step 2: Create a Custom Resource (CR)

```yaml
# my-webapp.yaml
apiVersion: platform.example.com/v1alpha1
kind: WebApp
metadata:
  name: my-frontend
  namespace: default
  labels:
    team: frontend
spec:
  image: nginx:1.27
  replicas: 3
  port: 80
  ingress:
    enabled: true
    host: frontend.example.com
  env:
    - name: LOG_LEVEL
      value: info
    - name: ENVIRONMENT
      value: production
```

```powershell
kubectl apply -f my-webapp.yaml
```

### Step 3: Interact With It Like Any K8s Resource

```powershell
# List custom resources
kubectl get webapps
kubectl get wa                          # Short name
# NAME          IMAGE        REPLICAS   PHASE   AGE
# my-frontend   nginx:1.27   3                  30s

# Describe
kubectl describe webapp my-frontend

# Get as YAML
kubectl get webapp my-frontend -o yaml

# Edit
kubectl edit webapp my-frontend

# Delete
kubectl delete webapp my-frontend

# Label
kubectl label webapp my-frontend tier=frontend

# Watch
kubectl get webapp -w

# Filter by label
kubectl get webapp -l team=frontend

# Get all resources in the "platform" category
kubectl get platform
```

### Validation In Action

```yaml
# This will be REJECTED by the API server:
apiVersion: platform.example.com/v1alpha1
kind: WebApp
metadata:
  name: bad-webapp
spec:
  image: nginx:1.27
  replicas: 200            # Exceeds maximum of 100
```

```
Error: WebApp.platform.example.com "bad-webapp" is invalid:
  spec.replicas: Invalid value: 200: must be less than or equal to 100
```

---

## 10.6 — The Status Subresource

When `subresources.status` is enabled, the `.status` field is handled separately from `.spec`:

```
/apis/platform.example.com/v1alpha1/namespaces/default/webapps/my-frontend          ← reads/writes .spec
/apis/platform.example.com/v1alpha1/namespaces/default/webapps/my-frontend/status    ← reads/writes .status
```

**Why this matters**:
- Users set `.spec` (desired state): "I want 3 replicas of nginx:1.27"
- Controllers set `.status` (observed state): "Currently 2 replicas ready, phase: Running"
- RBAC can grant different permissions: users can edit `.spec`, only the controller can edit `.status`
- `kubectl apply` on a manifest doesn't overwrite `.status`

### Updating Status From a Controller

```powershell
# Patch the status subresource (typically done by a controller, not manually)
kubectl patch webapp my-frontend --type=merge --subresource=status -p '{"status":{"readyReplicas":3,"phase":"Running"}}'

# Now kubectl get shows the Phase column
kubectl get wa
# NAME          IMAGE        REPLICAS   PHASE     AGE
# my-frontend   nginx:1.27   3          Running   5m
```

---

## 10.7 — Versioning CRDs

As your CRD evolves, you'll need multiple versions:

```yaml
spec:
  versions:
    - name: v1alpha1
      served: true              # Still available
      storage: false            # No longer the storage version

    - name: v1beta1
      served: true              # Available
      storage: false

    - name: v1
      served: true              # Available
      storage: true             # THIS is what's stored in etcd

  conversion:
    strategy: Webhook            # Use a webhook to convert between versions
    webhook:
      clientConfig:
        service:
          name: webapp-conversion
          namespace: system
          path: /convert
      conversionReviewVersions: ["v1"]
```

### Version Evolution Strategy

```
v1alpha1  →  v1beta1  →  v1 (stable)
   ↑            ↑          ↑
 Experimental  Mostly     Stable, backward-compatible
 May break     stable     changes only
```

> [!NOTE]
> For simple CRDs, you can use `strategy: None` (all versions must have the same schema). The `Webhook` strategy is for when field names change, fields are added/removed, or data needs transformation between versions.

---

## 10.8 — CRD vs ConfigMap vs Aggregated API Server

When to use what:

| Approach | Use When |
|---|---|
| **ConfigMap** | Storing configuration data consumed by a single app. No schema validation, no custom kubectl experience, no controller needed. |
| **CRD** | Defining a new domain-specific resource type. Needs schema validation, RBAC, kubectl integration, and a controller to act on it. |
| **Aggregated API Server** | Maximum flexibility — when CRD limitations block you (custom storage, non-CRUD operations, sub-resource logic). Much more complex to build. Used by metrics-server. |

### CRD Limitations

- Storage is always etcd (can't use a custom database)
- Limited to CRUD operations (can't define custom API verbs)
- Validation is schema-only (complex business logic needs a validating webhook)
- No built-in finalization logic (must use finalizers + controller)

---

## 10.9 — Real-World CRDs in the Ecosystem

Almost every Kubernetes add-on uses CRDs. Here are the most common:

| Tool | CRDs It Installs | What They Represent |
|---|---|---|
| **cert-manager** | `Certificate`, `Issuer`, `ClusterIssuer`, `CertificateRequest` | TLS certificates and how to obtain them |
| **Istio** | `VirtualService`, `DestinationRule`, `Gateway`, `ServiceEntry` | Service mesh traffic routing |
| **ArgoCD** | `Application`, `AppProject`, `ApplicationSet` | GitOps application definitions |
| **Prometheus Operator** | `ServiceMonitor`, `PodMonitor`, `PrometheusRule`, `AlertmanagerConfig` | Monitoring targets and alert rules |
| **KEDA** | `ScaledObject`, `ScaledJob`, `TriggerAuthentication` | Event-driven autoscaling configuration |
| **Crossplane** | Provider-specific: `RDSInstance`, `S3Bucket`, `VPC` | Cloud infrastructure as K8s resources |
| **Knative** | `Service`, `Route`, `Configuration`, `Revision` | Serverless workloads |
| **Velero** | `Backup`, `Restore`, `Schedule` | Cluster backup and disaster recovery |

```powershell
# See ALL CRDs installed in your cluster
kubectl get crd

# A typical production cluster might have 50-200+ CRDs
kubectl get crd | Measure-Object -Line
```

> [!TIP]
> When you install any K8s tool via Helm, check what CRDs it creates:
> ```powershell
> kubectl get crd | findstr "cert-manager"
> kubectl get crd | findstr "istio"
> ```
> Understanding the CRDs = understanding the tool's API.

---

## 10.10 — Webhooks: Validation and Mutation

CRDs have basic schema validation, but for complex business logic, you use **admission webhooks**:

### Validating Webhook

Rejects invalid resources:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: webapp-validator
webhooks:
  - name: validate.webapp.platform.example.com
    clientConfig:
      service:
        name: webapp-webhook
        namespace: system
        path: /validate
    rules:
      - apiGroups: ["platform.example.com"]
        apiVersions: ["v1alpha1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["webapps"]
    failurePolicy: Fail                  # Reject if webhook is unavailable
    sideEffects: None
```

### Mutating Webhook

Modifies resources before they're persisted (inject defaults, add sidecars):

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: webapp-mutator
webhooks:
  - name: mutate.webapp.platform.example.com
    clientConfig:
      service:
        name: webapp-webhook
        namespace: system
        path: /mutate
    rules:
      - apiGroups: ["platform.example.com"]
        apiVersions: ["v1alpha1"]
        operations: ["CREATE"]
        resources: ["webapps"]
    failurePolicy: Fail
    sideEffects: None
```

### Request Flow

```
kubectl apply -f webapp.yaml
        ↓
API Server → Mutating Webhooks (add defaults, inject sidecars)
        ↓
API Server → Schema Validation (OpenAPI v3)
        ↓
API Server → Validating Webhooks (business logic checks)
        ↓
Persisted to etcd
```

---

## Phase 10 — kubectl Cheat Sheet

| Command | Purpose |
|---|---|
| `kubectl get crd` | List all Custom Resource Definitions |
| `kubectl describe crd <name>` | CRD details + schema summary |
| `kubectl get crd <name> -o yaml` | Full CRD spec |
| `kubectl delete crd <name>` | Delete CRD (and ALL its custom resources!) |
| `kubectl api-resources \| findstr <name>` | Verify CRD registration |
| `kubectl get <plural>` / `kubectl get <shortName>` | List custom resources |
| `kubectl describe <kind> <name>` | Describe a custom resource |
| `kubectl get <plural> -o yaml` | Full CR spec |
| `kubectl explain <kind>.spec` | View CRD schema documentation |
| `kubectl patch <kind> <name> --subresource=status -p '{...}'` | Update status subresource |

---

## 🔬 Lab Exercise 10: Build a Custom Resource Definition

### Objective

Create a CRD called `WebApp` that represents a complete web application, then create instances and interact with them using kubectl.

### Step 1: Create the CRD

Create `lab10-crd.yaml` with:

- **Group**: `lab.example.com`
- **Kind**: `WebApp`
- **Plural**: `webapps`
- **Short name**: `wa`
- **Scope**: `Namespaced`
- **Category**: `all`
- **Version**: `v1`

Schema (`spec` fields):
| Field | Type | Required | Constraints |
|---|---|---|---|
| `image` | string | Yes | — |
| `replicas` | integer | Yes | min: 1, max: 50, default: 1 |
| `port` | integer | No | default: 80 |
| `serviceType` | string | No | enum: `["ClusterIP", "NodePort", "LoadBalancer"]`, default: `ClusterIP` |

Schema (`status` fields):
| Field | Type |
|---|---|
| `readyReplicas` | integer |
| `phase` | string, enum: `["Pending", "Running", "Failed"]` |

Enable the status subresource.

Add printer columns: Image, Replicas, Service Type, Phase, Age.

```powershell
kubectl apply -f lab10-crd.yaml
```

### Step 2: Verify Registration

```powershell
kubectl api-resources | findstr webapp
kubectl explain webapp.spec
```

### Step 3: Create Custom Resources

Create `lab10-webapps.yaml` with two WebApp instances:

**WebApp 1**:
```yaml
apiVersion: lab.example.com/v1
kind: WebApp
metadata:
  name: frontend
  labels:
    team: ui
spec:
  image: nginx:1.27
  replicas: 3
  port: 80
  serviceType: ClusterIP
```

**WebApp 2**:
```yaml
apiVersion: lab.example.com/v1
kind: WebApp
metadata:
  name: api-server
  labels:
    team: backend
spec:
  image: node:20-alpine
  replicas: 5
  port: 3000
  serviceType: NodePort
```

```powershell
kubectl apply -f lab10-webapps.yaml
```

### Step 4: Interact

```powershell
# List using plural, short name, and category
kubectl get webapps
kubectl get wa
kubectl get all                   # Should include webapps if category is "all"

# Describe
kubectl describe wa frontend

# Filter by label
kubectl get wa -l team=backend

# Edit
kubectl edit wa frontend

# Patch the status
kubectl patch wa frontend --type=merge --subresource=status -p '{"status":{"readyReplicas":3,"phase":"Running"}}'
kubectl patch wa api-server --type=merge --subresource=status -p '{"status":{"readyReplicas":5,"phase":"Running"}}'

# View with status columns
kubectl get wa
```

### Step 5: Test Validation

Try creating an invalid WebApp:
```yaml
apiVersion: lab.example.com/v1
kind: WebApp
metadata:
  name: bad-app
spec:
  image: nginx:1.27
  replicas: 999                  # Should be rejected (max 50)
  serviceType: gRPC              # Should be rejected (not in enum)
```

```powershell
kubectl apply -f lab10-bad.yaml
# Should fail with validation errors
```

### What to Submit

1. Your `lab10-crd.yaml`
2. Your `lab10-webapps.yaml`
3. Output of `kubectl api-resources | findstr webapp`
4. Output of `kubectl get wa` showing both resources with printer columns
5. Output of `kubectl get wa frontend -o yaml` (showing status after patching)
6. The validation error from the invalid WebApp attempt

---

> **⏸️ CHECKPOINT**: Submit your lab results or request to skip. Next: **Phase 11: Operators** — controllers that turn CRDs into self-managing applications.

# Phase 4: Configuration & Secrets Externalization

> **Goal**: Decouple your application configuration from your container images. Inject config data via ConfigMaps, protect sensitive data with Secrets, and expose Pod metadata through the Downward API — so the same image runs in dev, staging, and production with zero code changes.

---

## 4.1 — The Problem: Hardcoded Configuration

Consider this bad pattern:

```dockerfile
# Dockerfile
ENV DATABASE_HOST=prod-db.example.com
ENV DATABASE_PASSWORD=supersecret123
```

What's wrong?
- The image is locked to one environment. You'd need a separate image per environment.
- Credentials are baked into the image layer — anyone with `docker pull` access can read them.
- Changing a config value requires a full image rebuild and redeployment.

Kubernetes solves this with two primitives: **ConfigMaps** (non-sensitive config) and **Secrets** (sensitive data). Both inject data into Pods at runtime, not build time.

---

## 4.2 — ConfigMaps: Non-Sensitive Configuration

A ConfigMap is a key-value store for configuration data. It holds things like:
- Database hostnames, port numbers, feature flags
- Full configuration files (nginx.conf, app.properties)
- Environment-specific settings

### Creating ConfigMaps

#### Method 1: From a YAML manifest (declarative — preferred)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # Simple key-value pairs
  DATABASE_HOST: "postgres.default.svc.cluster.local"
  DATABASE_PORT: "5432"
  LOG_LEVEL: "info"
  FEATURE_DARK_MODE: "true"

  # Entire config files as values (multi-line string)
  nginx.conf: |
    server {
      listen 80;
      server_name localhost;
      location / {
        proxy_pass http://backend-svc:8080;
      }
    }
```

```powershell
kubectl apply -f configmap.yaml
```

#### Method 2: From the command line (imperative — good for quick testing)

```powershell
# From literal key-value pairs
kubectl create configmap app-config --from-literal=DATABASE_HOST=postgres.local --from-literal=LOG_LEVEL=debug

# From an entire file (the filename becomes the key)
kubectl create configmap nginx-config --from-file=nginx.conf

# From a directory (each file becomes a key)
kubectl create configmap configs --from-file=./config-dir/

# From an .env file
kubectl create configmap env-config --from-env-file=app.env
```

#### Inspecting ConfigMaps

```powershell
kubectl get configmaps
kubectl get cm                      # short alias
kubectl describe cm app-config
kubectl get cm app-config -o yaml   # see the full data
```

---

### Consuming ConfigMaps in Pods

There are three ways to inject ConfigMap data into a container:

### Option A: Individual Environment Variables

Cherry-pick specific keys from a ConfigMap:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app
      image: myapp:1.0
      env:
        - name: DB_HOST                      # Env var name in the container
          valueFrom:
            configMapKeyRef:
              name: app-config               # ConfigMap name
              key: DATABASE_HOST             # Key inside the ConfigMap
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DATABASE_PORT
```

Inside the container: `echo $DB_HOST` → `postgres.default.svc.cluster.local`

### Option B: Bulk-Load All Keys as Environment Variables

Inject every key-value pair from a ConfigMap as env vars in one shot:

```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      envFrom:
        - configMapRef:
            name: app-config
        - prefix: CFG_                       # Optional: prefix all keys
          configMapRef:
            name: another-config
```

This creates env vars `DATABASE_HOST`, `DATABASE_PORT`, `LOG_LEVEL`, etc. — one per key. With the prefix, the second ConfigMap's keys become `CFG_KEY_NAME`.

> [!WARNING]
> **Keys that aren't valid env var names** (contain dashes, dots, or start with numbers) will be **silently skipped** when using `envFrom`. For example, a key named `nginx.conf` won't become an env var. Use volume mounts for file-type keys.

### Option C: Mount as Files in a Volume

Mount the ConfigMap as a directory, where each key becomes a filename and each value becomes the file contents:

```yaml
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      volumeMounts:
        - name: config-volume
          mountPath: /etc/nginx/conf.d        # Directory where files appear
          readOnly: true
  volumes:
    - name: config-volume
      configMap:
        name: app-config                      # ConfigMap name
        items:                                # Optional: select specific keys
          - key: nginx.conf                   # Key in the ConfigMap
            path: default.conf                # Filename in the mount directory
```

Result inside the container:
```
/etc/nginx/conf.d/default.conf  ← contents of the "nginx.conf" key
```

If you omit `items`, ALL keys are mounted as files.

> [!TIP]
> **Volume-mounted ConfigMaps auto-update.** If you update the ConfigMap, the mounted files are refreshed within ~30–60 seconds (kubelet sync period). Environment variables do NOT auto-update — they're set at Pod startup and frozen. This is a key architectural decision: use volumes for config that needs hot-reload, env vars for everything else.

### Option D: Mount a Single Key as a Specific File (Without Overwriting the Directory)

By default, mounting a volume at `/etc/nginx/conf.d` replaces the entire directory. To inject a single file without clobbering existing contents, use `subPath`:

```yaml
volumeMounts:
  - name: config-volume
    mountPath: /etc/nginx/conf.d/custom.conf    # Specific file path
    subPath: nginx.conf                          # Key from the ConfigMap
```

> [!CAUTION]
> `subPath` mounts do NOT receive automatic updates when the ConfigMap changes. This is a known limitation. If you need auto-updates, mount the full directory instead.

---

## 4.3 — Secrets: Sensitive Data

Secrets work almost identically to ConfigMaps but are designed for sensitive data: passwords, API keys, TLS certificates, SSH keys, OAuth tokens.

### How Secrets Differ from ConfigMaps

| Aspect | ConfigMap | Secret |
|---|---|---|
| **Intended for** | Non-sensitive config | Passwords, keys, tokens, certs |
| **Data encoding** | Plain text | Base64-encoded (NOT encrypted by default!) |
| **Access control** | Standard RBAC | Should have stricter RBAC |
| **Size limit** | 1 MiB | 1 MiB |
| **etcd storage** | Plain text | Plain text by default (can enable encryption-at-rest) |
| **Env var display** | Visible in `kubectl describe pod` | Visible in `kubectl describe pod` ⚠️ |

> [!CAUTION]
> **Base64 is NOT encryption.** Anyone who can run `kubectl get secret -o yaml` can decode your passwords in 2 seconds. Secrets are only "secret" through RBAC (controlling who can read them) and optional encryption-at-rest in etcd. Don't let the name give you a false sense of security.

### Secret Types

| Type | Purpose | Keys Required |
|---|---|---|
| `Opaque` | General purpose (default) | Any |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials | `.dockerconfigjson` |
| `kubernetes.io/tls` | TLS certificate + private key | `tls.crt`, `tls.key` |
| `kubernetes.io/basic-auth` | Basic auth credentials | `username`, `password` |
| `kubernetes.io/ssh-auth` | SSH private key | `ssh-privatekey` |
| `kubernetes.io/service-account-token` | Service account token (auto-created) | Auto-managed |

### Creating Secrets

#### From the command line

```powershell
# Opaque secret from literals
kubectl create secret generic db-credentials --from-literal=username=admin --from-literal=password=S3cureP@ss!

# From a file
kubectl create secret generic tls-cert --from-file=cert.pem --from-file=key.pem

# TLS secret (special type)
kubectl create secret tls my-tls-secret --cert=cert.pem --key=key.pem

# Docker registry secret
kubectl create secret docker-registry regcred --docker-server=registry.example.com --docker-username=user --docker-password=pass
```

#### From a YAML manifest

Values must be base64-encoded:

```powershell
# Encode your values first
echo -n "admin" | base64
# YWRtaW4=
echo -n "S3cureP@ss!" | base64
# UzNjdXJlUEBzcyE=
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=           # base64 of "admin"
  password: UzNjdXJlUEBzcyE=  # base64 of "S3cureP@ss!"
```

Or use `stringData` to skip the base64 step (Kubernetes encodes it for you):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:                    # Plain text — K8s converts to base64 automatically
  username: admin
  password: "S3cureP@ss!"
```

> [!TIP]
> Use `stringData` in your manifests. It's cleaner and less error-prone. Kubernetes converts it to `data` (base64) internally. Just **never commit `stringData` manifests to Git** — use a secrets manager (Vault, Sealed Secrets, SOPS) in production.

### Consuming Secrets in Pods

Exact same patterns as ConfigMaps — swap `configMapKeyRef`/`configMapRef` for `secretKeyRef`/`secretRef`:

#### As Environment Variables

```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
```

#### Bulk-Load All Keys

```yaml
envFrom:
  - secretRef:
      name: db-credentials
```

#### As Volume-Mounted Files

```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: db-credentials
        defaultMode: 0400              # File permissions: owner read-only
```

Result:
```
/etc/secrets/username  ← contains "admin"
/etc/secrets/password  ← contains "S3cureP@ss!"
```

> [!IMPORTANT]
> **Volume-mount Secrets with `readOnly: true` and restrictive `defaultMode` (e.g., `0400`)**. This limits who can read the files inside the container. It's a defense-in-depth measure.

---

## 4.4 — The Downward API: Pod Metadata as Configuration

Sometimes your application needs to know things about the Pod it's running in — its name, namespace, IP, node name, resource limits. The **Downward API** exposes this metadata as environment variables or files without hardcoding anything.

### Available Fields

| Field Selector | Value |
|---|---|
| `metadata.name` | Pod name |
| `metadata.namespace` | Pod namespace |
| `metadata.uid` | Pod UID |
| `metadata.labels['<key>']` | Specific label value |
| `metadata.annotations['<key>']` | Specific annotation value |
| `status.podIP` | Pod's IP address |
| `spec.nodeName` | Node the Pod is running on |
| `spec.serviceAccountName` | Service account name |
| `status.hostIP` | Node's IP address |

### Resource Fields

| Resource Field | Value |
|---|---|
| `requests.cpu` | CPU requests |
| `requests.memory` | Memory requests |
| `limits.cpu` | CPU limits |
| `limits.memory` | Memory limits |

### As Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-demo
  labels:
    app: demo
    version: v3
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "env | sort && sleep 3600"]
      env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name

        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace

        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP

        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName

        - name: CPU_LIMIT
          valueFrom:
            resourceFieldRef:              # Note: resourceFieldRef, not fieldRef
              containerName: app
              resource: limits.cpu
```

### As Volume-Mounted Files

```yaml
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "cat /etc/podinfo/labels && sleep 3600"]
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
  volumes:
    - name: podinfo
      downwardAPI:
        items:
          - path: labels
            fieldRef:
              fieldPath: metadata.labels
          - path: annotations
            fieldRef:
              fieldPath: metadata.annotations
```

Result:
```
/etc/podinfo/labels      ← contains: app="demo"\nversion="v3"
/etc/podinfo/annotations ← contains all annotations as key=value pairs
```

> [!NOTE]
> **Common use case**: Logging and monitoring agents often need `POD_NAME`, `NAMESPACE`, and `NODE_NAME` to tag metrics and logs with origin information. The Downward API eliminates the need for service discovery or API calls from the application.

---

## 4.5 — Combining Everything: A Real-World Pod Spec

Here's a Pod that uses ConfigMaps, Secrets, and the Downward API together:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: production-app
  labels:
    app: production-app
    environment: prod
spec:
  containers:
    - name: app
      image: myapp:2.0
      ports:
        - containerPort: 8080

      # --- Environment variables from multiple sources ---
      env:
        # From ConfigMap (individual keys)
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL

        # From Secret (individual keys)
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password

        # From Downward API
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name

        # Plain value (static)
        - name: APP_VERSION
          value: "2.0.0"

      # Bulk-load remaining config
      envFrom:
        - configMapRef:
            name: feature-flags
            prefix: FF_                    # FF_DARK_MODE, FF_BETA_ENABLED, etc.

      # --- Volume mounts ---
      volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
          readOnly: true
        - name: tls-certs
          mountPath: /etc/tls
          readOnly: true

  # --- Volumes ---
  volumes:
    - name: nginx-config
      configMap:
        name: nginx-configmap
    - name: tls-certs
      secret:
        secretName: tls-secret
        defaultMode: 0400
```

---

## 4.6 — Immutability: Locking Down ConfigMaps and Secrets

In Kubernetes 1.21+, you can make ConfigMaps and Secrets **immutable** — no further changes allowed after creation:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v2
immutable: true                 # Cannot be modified after creation
data:
  LOG_LEVEL: "warn"
```

Benefits:
- **Performance**: kubelet stops polling the API server for changes (reduces API server load significantly at scale)
- **Safety**: Prevents accidental modification of production config

To "update" an immutable ConfigMap: create a new one with a new name (e.g., `app-config-v3`), update your Deployment to reference it, and delete the old one.

---

## Phase 4 — kubectl Cheat Sheet

| Command | Purpose |
|---|---|
| `kubectl create configmap <name> --from-literal=key=value` | Create ConfigMap from literals |
| `kubectl create configmap <name> --from-file=<path>` | Create ConfigMap from file(s) |
| `kubectl create configmap <name> --from-env-file=<path>` | Create ConfigMap from .env file |
| `kubectl get cm` | List ConfigMaps |
| `kubectl describe cm <name>` | View ConfigMap details |
| `kubectl get cm <name> -o yaml` | View full ConfigMap data |
| `kubectl create secret generic <name> --from-literal=key=val` | Create Opaque secret |
| `kubectl create secret tls <name> --cert=<path> --key=<path>` | Create TLS secret |
| `kubectl create secret docker-registry <name> --docker-server=...` | Create registry secret |
| `kubectl get secrets` | List secrets |
| `kubectl describe secret <name>` | View secret metadata (values hidden) |
| `kubectl get secret <name> -o yaml` | View secret with base64 values |
| `kubectl get secret <name> -o jsonpath='{.data.password}' \| base64 -d` | Decode a secret value (Linux/Mac) |
| `kubectl get secret <name> -o jsonpath='{.data.password}'` then decode | Decode a secret value (PowerShell) |
| `kubectl edit cm <name>` | Edit a ConfigMap in-place |
| `kubectl delete cm <name>` | Delete a ConfigMap |
| `kubectl delete secret <name>` | Delete a Secret |

> [!TIP]
> **Decoding secrets in PowerShell**:
> ```powershell
> $encoded = kubectl get secret db-credentials -o jsonpath='{.data.password}'
> [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($encoded))
> ```

---

## 🔬 Lab Exercise 4: Build a Fully Externalized Application Configuration

### Objective

Deploy an Nginx pod whose behavior is entirely controlled by external configuration — no baked-in config, no hardcoded secrets.

### Step 1: Create the ConfigMap

Create `lab4-configmap.yaml` with the following:
- **Name**: `lab4-config`
- **Data keys**:
  - `APP_ENV`: `"development"`
  - `APP_DEBUG`: `"true"`
  - `WORKER_COUNT`: `"4"`
  - `nginx.conf`: A custom Nginx config that serves a page displaying "Lab 4 Works! Environment: development". Use this:
    ```
    server {
        listen 80;
        server_name localhost;
        location / {
            return 200 'Lab 4 Works! Environment: development\n';
            add_header Content-Type text/plain;
        }
    }
    ```

### Step 2: Create the Secret

Create `lab4-secret.yaml` with:
- **Name**: `lab4-secret`
- **Type**: Opaque
- Use `stringData` (not base64)
- **Keys**:
  - `API_KEY`: `"sk-lab4-a1b2c3d4e5f6"`
  - `DB_PASSWORD`: `"P@ssw0rd!2025"`

### Step 3: Create the Pod

Create `lab4-pod.yaml` with:
- **Name**: `lab4-app`
- **Container**: `nginx:1.27`
- **Environment variables** (from ConfigMap, individual keys): `APP_ENV`, `APP_DEBUG`
- **Environment variables** (from Secret, individual keys): `API_KEY`, `DB_PASSWORD`
- **Environment variables** (Downward API): `POD_NAME` (from `metadata.name`), `POD_IP` (from `status.podIP`)
- **Volume mount**: Mount the `nginx.conf` key from `lab4-config` at `/etc/nginx/conf.d/default.conf` using `subPath`
- **Volume mount**: Mount `lab4-secret` at `/etc/secrets` with `readOnly: true` and `defaultMode: 0400`

### Step 4: Verify Everything

```powershell
# Apply all three
kubectl apply -f lab4-configmap.yaml -f lab4-secret.yaml -f lab4-pod.yaml

# Check the pod is running
kubectl get pod lab4-app

# Verify env vars are injected
kubectl exec lab4-app -- env | findstr "APP_ENV APP_DEBUG API_KEY DB_PASSWORD POD_NAME POD_IP"

# Verify the custom nginx config works
kubectl exec lab4-app -- curl -s localhost
# Should return: "Lab 4 Works! Environment: development"

# Verify secret files are mounted
kubectl exec lab4-app -- ls -la /etc/secrets/
kubectl exec lab4-app -- cat /etc/secrets/API_KEY

# Verify file permissions on secrets
kubectl exec lab4-app -- stat /etc/secrets/API_KEY
```

### What to Submit

1. Your three manifest files: `lab4-configmap.yaml`, `lab4-secret.yaml`, `lab4-pod.yaml`
2. Output of `kubectl exec lab4-app -- env` (showing the injected variables)
3. Output of `kubectl exec lab4-app -- curl -s localhost` (proving the custom Nginx config works)
4. Output of `kubectl exec lab4-app -- ls -la /etc/secrets/` (showing the mounted secret files with correct permissions)

---

> **⏸️ CHECKPOINT**: Submit your lab results or request to skip. Next is **Phase 5: Storage & Persistence Lifecycle** — where we tackle the hard problem: what happens to data when Pods die.

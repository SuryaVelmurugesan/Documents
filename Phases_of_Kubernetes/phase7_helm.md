# Phase 7: Helm — The Kubernetes Package Manager

> **Goal**: Stop copy-pasting YAML. Learn to package, version, parameterize, and distribute Kubernetes applications using Helm charts — the de facto standard for K8s package management.

---

## 7.1 — The Problem Helm Solves

By Phase 6, you've written dozens of YAML files: Deployments, Services, ConfigMaps, Secrets, Ingress, RBAC, PVCs. For a real application, you're looking at 10-20 manifests minimum. Now imagine:

- Deploying the same app to **dev, staging, and prod** — same structure, different values (replicas, image tags, resource limits)
- Managing **versions** — rolling back not just the image but the entire manifest set
- **Sharing** a complex application (like Prometheus or PostgreSQL) as a reusable package
- **Upgrading** a third-party application with a single command

Raw `kubectl apply -f` doesn't solve these. **Helm** does.

### What Is Helm?

Helm is three things:

| Concept | What It Is |
|---|---|
| **Chart** | A package — a directory of templated YAML files + metadata. Think of it as a "recipe" for deploying an application. |
| **Release** | An installed instance of a chart in a cluster. You can install the same chart multiple times (e.g., `postgres-orders`, `postgres-users`). |
| **Repository** | A place where charts are stored and shared. Like npm for Node or PyPI for Python. |

```
Chart (package)  →  helm install  →  Release (running instance)
                                        ↓
                     helm upgrade  →  New revision of the release
                                        ↓
                     helm rollback →  Previous revision restored
```

---

## 7.2 — Installation

```powershell
# Windows — using winget
winget install Helm.Helm

# Windows — using Chocolatey
choco install kubernetes-helm

# Verify
helm version
# version.BuildInfo{Version:"v3.x.x", ...}
```

> [!NOTE]
> Helm 3 is the current major version. Helm 2 used a server-side component called "Tiller" — it was removed in Helm 3 for security reasons. All operations now run client-side using your kubeconfig credentials.

---

## 7.3 — Using Existing Charts: Your First Helm Install

Before creating your own charts, let's use one from a public repository.

### Adding a Repository

```powershell
# Add the Bitnami repository (one of the largest public chart repos)
helm repo add bitnami https://charts.bitnami.com/bitnami

# Update repository index (like apt update)
helm repo update

# Search for charts
helm search repo nginx
# NAME                  CHART VERSION   APP VERSION   DESCRIPTION
# bitnami/nginx         15.x.x         1.27.x        NGINX Open Source for Kubernetes

# Search with all versions
helm search repo nginx --versions

# List your configured repos
helm repo list
```

### Installing a Chart

```powershell
# Install nginx with default values — creates a "release" named my-nginx
helm install my-nginx bitnami/nginx

# Install with custom values (inline)
helm install my-nginx bitnami/nginx --set replicaCount=3 --set service.type=ClusterIP

# Install with a values file (preferred — version-controllable)
helm install my-nginx bitnami/nginx -f my-values.yaml

# Install into a specific namespace (create it if needed)
helm install my-nginx bitnami/nginx --namespace web --create-namespace

# Dry run — render templates without installing
helm install my-nginx bitnami/nginx --dry-run --debug
```

### Inspecting a Release

```powershell
# List installed releases
helm list
helm list -A                    # All namespaces

# See the status of a release
helm status my-nginx

# See the computed values (defaults + your overrides)
helm get values my-nginx
helm get values my-nginx --all  # Including defaults

# See the rendered manifests that were applied
helm get manifest my-nginx

# See release history
helm history my-nginx
```

### Upgrading a Release

```powershell
# Upgrade with new values
helm upgrade my-nginx bitnami/nginx --set replicaCount=5

# Upgrade with a new chart version
helm upgrade my-nginx bitnami/nginx --version 16.0.0

# Upgrade with a values file
helm upgrade my-nginx bitnami/nginx -f production-values.yaml

# Install if not exists, upgrade if it does (idempotent — great for CI/CD)
helm upgrade --install my-nginx bitnami/nginx -f values.yaml
```

### Rolling Back

```powershell
# Rollback to the previous revision
helm rollback my-nginx

# Rollback to a specific revision
helm rollback my-nginx 2

# View history to find revision numbers
helm history my-nginx
# REVISION  STATUS      DESCRIPTION
# 1         superseded  Install complete
# 2         superseded  Upgrade complete
# 3         deployed    Rollback to 2
```

### Uninstalling

```powershell
# Remove the release and all its Kubernetes resources
helm uninstall my-nginx

# Keep the release history (for auditing)
helm uninstall my-nginx --keep-history
```

---

## 7.4 — Chart Anatomy: What's Inside a Chart

```
mychart/
├── Chart.yaml            # Chart metadata (name, version, dependencies)
├── values.yaml           # Default configuration values
├── charts/               # Dependency charts (sub-charts)
├── templates/            # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── serviceaccount.yaml
│   ├── hpa.yaml
│   ├── _helpers.tpl      # Named template definitions (partial templates)
│   ├── NOTES.txt         # Post-install instructions (displayed after helm install)
│   └── tests/
│       └── test-connection.yaml   # Helm test pods
├── .helmignore           # Files to exclude from packaging (like .gitignore)
└── README.md             # Documentation
```

### Chart.yaml — The Identity Card

```yaml
apiVersion: v2                    # v2 = Helm 3 charts
name: mywebapp
description: A full-stack web application chart
type: application                 # "application" or "library"
version: 1.2.0                    # Chart version (SemVer — YOUR packaging version)
appVersion: "3.5.1"               # App version (the version of the software being deployed)
keywords:
  - web
  - nginx
maintainers:
  - name: Platform Team
    email: platform@example.com
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled   # Only include if postgresql.enabled=true
```

> [!IMPORTANT]
> **`version` vs `appVersion`**: `version` is the chart's own version — increment this when you change templates, defaults, or structure. `appVersion` is the version of the application the chart deploys (e.g., nginx 1.27). They are independent.

### values.yaml — The Knobs and Dials

```yaml
# values.yaml — all configurable parameters with sane defaults
replicaCount: 2

image:
  repository: nginx
  tag: "1.27"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: nginx
  host: myapp.local

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

configMap:
  logLevel: info
  environment: production
```

Users override these with `-f custom-values.yaml` or `--set key=value`.

---

## 7.5 — Template Language Deep Dive

Helm templates use **Go templates** with Sprig functions. This is where the power (and complexity) lives.

### Built-In Objects

| Object | What It Contains |
|---|---|
| `.Values` | Values from `values.yaml` + user overrides |
| `.Chart` | Data from `Chart.yaml` (name, version, appVersion) |
| `.Release` | Release info (name, namespace, revision, isInstall, isUpgrade) |
| `.Capabilities` | Cluster capabilities (K8s version, API versions) |
| `.Template` | Current template info (name, basePath) |

### Basic Templating

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-webapp        # Release name injected
  labels:
    app.kubernetes.io/name: {{ .Chart.Name }}
    app.kubernetes.io/version: {{ .Chart.AppVersion }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  replicas: {{ .Values.replicaCount }}     # From values.yaml
  selector:
    matchLabels:
      app: {{ .Release.Name }}-webapp
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-webapp
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 80
```

### Functions and Pipelines

```yaml
# Functions — transform values
name: {{ .Release.Name | upper }}                 # MY-RELEASE
name: {{ .Release.Name | lower }}                 # my-release
name: {{ .Release.Name | quote }}                 # "my-release"
name: {{ .Values.image.tag | default "latest" }}  # Use "latest" if tag is empty

# Pipelines — chain functions (like Unix pipes)
annotations:
  checksum: {{ .Values.configMap | toYaml | sha256sum }}

# indent/nindent — crucial for YAML formatting
resources:
{{ toYaml .Values.resources | indent 10 }}
# or (nindent = newline + indent, cleaner)
resources:
  {{- toYaml .Values.resources | nindent 10 }}
```

> [!TIP]
> **`{{-` vs `{{`**: The dash trims whitespace. `{{- ` trims whitespace before the tag, ` -}}` trims after. Use `{{-` liberally to avoid blank lines in your rendered YAML. Whitespace bugs are the #1 source of Helm template errors.

### Flow Control

```yaml
# IF / ELSE
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-ingress
spec:
  ingressClassName: {{ .Values.ingress.className }}
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ .Release.Name }}-svc
                port:
                  number: {{ .Values.service.port }}
{{- end }}

# RANGE — iterate over lists/maps
{{- range .Values.extraEnvironmentVars }}
- name: {{ .name }}
  value: {{ .value | quote }}
{{- end }}

# WITH — change the scope
{{- with .Values.resources }}
resources:
  {{- toYaml . | nindent 10 }}
{{- end }}
```

### Named Templates (_helpers.tpl)

Reusable template snippets defined in `_helpers.tpl`:

```yaml
# templates/_helpers.tpl

{{/*
Generate standard labels for all resources
*/}}
{{- define "mywebapp.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version }}
{{- end }}

{{/*
Generate the fullname: release-name-chart-name, truncated to 63 chars
*/}}
{{- define "mywebapp.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Generate selector labels (subset of full labels, used in matchLabels)
*/}}
{{- define "mywebapp.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

Use them with `include`:

```yaml
# templates/deployment.yaml
metadata:
  name: {{ include "mywebapp.fullname" . }}
  labels:
    {{- include "mywebapp.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "mywebapp.selectorLabels" . | nindent 6 }}
```

> [!NOTE]
> **`include` vs `template`**: Always use `include`. It returns a string you can pipe through functions (`| nindent 4`). `template` outputs directly and can't be piped.

### NOTES.txt — Post-Install Instructions

```
# templates/NOTES.txt
Thank you for installing {{ .Chart.Name }}!

Your release "{{ .Release.Name }}" has been deployed to namespace "{{ .Release.Namespace }}".

To access the application:
{{- if eq .Values.service.type "NodePort" }}
  export NODE_PORT=$(kubectl get --namespace {{ .Release.Namespace }} -o jsonpath="{.spec.ports[0].nodePort}" services {{ include "mywebapp.fullname" . }})
  export NODE_IP=$(kubectl get nodes --namespace {{ .Release.Namespace }} -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
{{- else if eq .Values.service.type "ClusterIP" }}
  kubectl port-forward svc/{{ include "mywebapp.fullname" . }} 8080:{{ .Values.service.port }}
  echo http://localhost:8080
{{- end }}
```

---

## 7.6 — Creating a Chart From Scratch

```powershell
# Scaffold a new chart
helm create mywebapp

# This generates the full directory structure with sensible defaults
# Edit templates/ and values.yaml to match your application

# Validate your chart (lint for errors)
helm lint mywebapp/

# Render templates locally (see the final YAML without installing)
helm template my-release mywebapp/
helm template my-release mywebapp/ -f prod-values.yaml

# Dry-run against the cluster (validates with the API server)
helm install my-release mywebapp/ --dry-run --debug

# Install for real
helm install my-release mywebapp/
```

---

## 7.7 — Dependencies (Sub-Charts)

Your chart can depend on other charts (e.g., include PostgreSQL as a sub-chart):

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "12.12.10"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled           # Toggle with values
  - name: redis
    version: "18.6.1"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

```powershell
# Download dependencies into charts/ directory
helm dependency update mywebapp/

# This creates:
# mywebapp/charts/postgresql-12.12.10.tgz
# mywebapp/charts/redis-18.6.1.tgz
```

Override sub-chart values by nesting under the dependency name:

```yaml
# values.yaml
postgresql:
  enabled: true
  auth:
    postgresPassword: "mypassword"
    database: "myapp"
  primary:
    persistence:
      size: 10Gi

redis:
  enabled: false    # Don't install Redis
```

---

## 7.8 — Helm Hooks

Hooks let you run actions at specific points in the release lifecycle:

```yaml
# templates/db-migrate-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-db-migrate
  annotations:
    "helm.sh/hook": pre-upgrade           # Run BEFORE upgrade
    "helm.sh/hook-weight": "0"            # Order (lower = runs first)
    "helm.sh/hook-delete-policy": hook-succeeded   # Clean up after success
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: myapp:{{ .Values.image.tag }}
          command: ["./migrate", "--up"]
```

| Hook | When It Runs |
|---|---|
| `pre-install` | Before any resources are created |
| `post-install` | After all resources are created |
| `pre-upgrade` | Before an upgrade begins |
| `post-upgrade` | After an upgrade completes |
| `pre-delete` | Before resources are deleted |
| `post-delete` | After resources are deleted |
| `pre-rollback` | Before a rollback |
| `post-rollback` | After a rollback |
| `test` | When `helm test` is run |

---

## 7.9 — Packaging and Sharing

```powershell
# Package a chart into a .tgz archive
helm package mywebapp/
# Creates: mywebapp-1.2.0.tgz

# Push to an OCI registry (Helm 3.8+)
helm push mywebapp-1.2.0.tgz oci://registry.example.com/charts

# Install from OCI
helm install my-release oci://registry.example.com/charts/mywebapp --version 1.2.0
```

---

## Phase 7 — Helm Cheat Sheet

| Command | Purpose |
|---|---|
| `helm repo add <name> <url>` | Add a chart repository |
| `helm repo update` | Update repo index |
| `helm search repo <keyword>` | Search for charts |
| `helm show values <chart>` | View a chart's default values |
| `helm show chart <chart>` | View Chart.yaml metadata |
| `helm install <release> <chart>` | Install a chart |
| `helm install <release> <chart> -f values.yaml` | Install with custom values |
| `helm install <release> <chart> --set key=val` | Install with inline override |
| `helm install <release> <chart> --dry-run --debug` | Preview without installing |
| `helm upgrade <release> <chart>` | Upgrade a release |
| `helm upgrade --install <release> <chart>` | Install or upgrade (idempotent) |
| `helm rollback <release> [revision]` | Rollback a release |
| `helm list` / `helm list -A` | List releases |
| `helm status <release>` | Release status |
| `helm history <release>` | Release revision history |
| `helm get values <release>` | View applied values |
| `helm get manifest <release>` | View rendered manifests |
| `helm uninstall <release>` | Delete a release |
| `helm create <name>` | Scaffold a new chart |
| `helm lint <chart-dir>` | Validate chart syntax |
| `helm template <release> <chart-dir>` | Render templates locally |
| `helm dependency update <chart-dir>` | Download sub-chart dependencies |
| `helm package <chart-dir>` | Package chart into .tgz |
| `helm push <tgz> oci://<registry>` | Push to OCI registry |
| `helm test <release>` | Run chart tests |

---

## 🔬 Lab Exercise 7: Build and Deploy a Full-Stack Helm Chart

### Objective

Create a Helm chart from scratch that deploys an Nginx frontend with a configurable ConfigMap, Service, optional Ingress, and resource management — all parameterized through `values.yaml`.

### Step 1: Scaffold the Chart

```powershell
helm create lab7-webapp
```

### Step 2: Clean the Scaffolding

Delete everything inside `templates/` (we'll write it from scratch). Keep `_helpers.tpl` and `NOTES.txt`.

### Step 3: Write `_helpers.tpl`

Define three named templates:
- `lab7-webapp.fullname` — `<release-name>-lab7-webapp`, truncated to 63 chars
- `lab7-webapp.labels` — Standard labels (name, instance, version, managed-by)
- `lab7-webapp.selectorLabels` — Just name + instance (for matchLabels)

### Step 4: Write `values.yaml`

Create configurable parameters:
```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.27"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  host: lab7.local

resources:
  requests:
    cpu: 100m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 128Mi

config:
  welcomeMessage: "Hello from Helm Lab 7!"
```

### Step 5: Write the Templates

Create these template files:

1. **`configmap.yaml`** — A ConfigMap containing an `nginx.conf` that returns `{{ .Values.config.welcomeMessage }}`
2. **`deployment.yaml`** — Deployment using all the values above, mounting the ConfigMap
3. **`service.yaml`** — Service with configurable type and port
4. **`ingress.yaml`** — Only rendered if `ingress.enabled` is true

Use named templates from `_helpers.tpl` for all names and labels.

### Step 6: Validate

```powershell
# Lint
helm lint lab7-webapp/

# Render templates (preview the YAML)
helm template my-lab7 lab7-webapp/

# Render with overrides
helm template my-lab7 lab7-webapp/ --set replicaCount=5 --set config.welcomeMessage="Custom message!"
```

### Step 7: Install and Test

```powershell
# Install
helm install my-lab7 lab7-webapp/

# Verify
kubectl get all -l app.kubernetes.io/instance=my-lab7
kubectl exec <pod-name> -- curl -s localhost

# Upgrade — change the message
helm upgrade my-lab7 lab7-webapp/ --set config.welcomeMessage="Upgraded via Helm!"
kubectl exec <pod-name> -- curl -s localhost

# View history
helm history my-lab7

# Rollback
helm rollback my-lab7 1

# Clean up
helm uninstall my-lab7
```

### What to Submit

1. Your `Chart.yaml` and `values.yaml`
2. Your `_helpers.tpl`
3. Your `deployment.yaml` template (showing Go template usage)
4. Output of `helm template my-lab7 lab7-webapp/` (rendered YAML)
5. Output of `helm list` showing the installed release
6. Output of `helm history my-lab7` after the upgrade and rollback

---

> **⏸️ CHECKPOINT**: Submit your lab results or request to skip. Next: **Phase 8: Network Policies** — Kubernetes firewalls for pod-to-pod traffic.

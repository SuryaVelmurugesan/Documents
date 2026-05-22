# Phase 5: Storage & Persistence Lifecycle

> **Goal**: Understand how data survives Pod restarts and rescheduling. Master the PV/PVC abstraction, dynamic provisioning with StorageClasses, and know when to reach for StatefulSets over Deployments.

---

## 5.1 — The Core Problem: Containers Are Ephemeral

By default, everything inside a container's filesystem is **ephemeral** — it vanishes the instant the container stops. This includes:
- Application logs written to disk
- Database data files
- User-uploaded files
- Cache data

```
Pod starts  → container writes data to /var/data
Pod crashes → kubelet restarts the container
            → /var/data is GONE — fresh filesystem from the image
```

For stateless apps (web servers, API gateways), this is fine. For anything that stores data (databases, message queues, file uploads), you need **Volumes**.

---

## 5.2 — Ephemeral Volumes: emptyDir and hostPath

These are the simplest volume types. They don't involve any external storage system.

### emptyDir — Scratch Space That Lives With the Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c", "echo 'Hello from writer' > /data/message.txt && sleep 3600"]
      volumeMounts:
        - name: shared-data
          mountPath: /data

    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "sleep 5 && cat /data/message.txt && sleep 3600"]
      volumeMounts:
        - name: shared-data
          mountPath: /data

  volumes:
    - name: shared-data
      emptyDir: {}
```

**Behavior**:
- Created when the Pod is assigned to a node
- Starts empty (hence the name)
- Shared between all containers in the Pod
- **Survives container restarts** within the same Pod
- **Destroyed when the Pod is deleted** or rescheduled to another node
- Stored on the node's disk (or in memory with `emptyDir: { medium: Memory }`)

**Use cases**: Scratch space, shared data between sidecar containers, cache, temp files.

```yaml
# RAM-backed emptyDir — blazing fast, but counts against memory limits
volumes:
  - name: cache
    emptyDir:
      medium: Memory
      sizeLimit: 256Mi     # Important: set a limit or it can eat all node RAM
```

### hostPath — Mount a Node's Filesystem (Use With Extreme Caution)

```yaml
volumes:
  - name: host-data
    hostPath:
      path: /var/log/host-logs    # Path on the HOST node
      type: DirectoryOrCreate     # Create if it doesn't exist
```

**Behavior**:
- Mounts a file or directory from the host node's filesystem into the Pod
- Data persists across Pod restarts **only if the Pod lands on the same node**
- Completely node-specific — no portability

> [!CAUTION]
> **`hostPath` is a security risk and a portability nightmare.** It gives containers access to the host filesystem. A malicious container could read `/etc/shadow` or write to `/var/run/docker.sock`. In production, it's almost always the wrong choice. Use PersistentVolumes instead. The only valid use cases are node-level agents (log collectors, monitoring daemons) running as DaemonSets.

---

## 5.3 — The PersistentVolume (PV) / PersistentVolumeClaim (PVC) Model

For real persistent storage, Kubernetes introduces a two-object abstraction that **decouples storage provisioning from consumption**:

```
Cluster Admin                          Application Developer
     │                                        │
     ▼                                        ▼
 PersistentVolume (PV)              PersistentVolumeClaim (PVC)
 "Here is a 50Gi disk               "I need a 10Gi disk
  on AWS EBS, ReadWriteOnce"          with ReadWriteOnce access"
     │                                        │
     └────────────── BINDING ─────────────────┘
                        │
                        ▼
                  Pod mounts the PVC
```

**Why two objects?**
- **Separation of concerns**: The admin manages infrastructure (disks, NFS shares, cloud volumes). The developer just says "I need X amount of storage."
- **Portability**: The Pod spec references a PVC, not a specific disk. Move from AWS to GCP? Change the PV, not the Pod.

### PersistentVolume (PV) — The Actual Storage

A PV represents a piece of storage that has been provisioned — either manually by an admin or dynamically by a StorageClass.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-manual-50gi
spec:
  capacity:
    storage: 50Gi                      # Size of the volume

  accessModes:
    - ReadWriteOnce                    # Can be mounted read-write by ONE node

  persistentVolumeReclaimPolicy: Retain  # What happens when PVC is deleted

  storageClassName: manual             # Links to a StorageClass (or "manual" for static)

  # The actual backend — depends on your infrastructure
  hostPath:                            # For local/minikube only!
    path: /mnt/data/pv-50gi
    type: DirectoryOrCreate
```

### Access Modes

| Mode | Abbreviation | Meaning |
|---|---|---|
| `ReadWriteOnce` | `RWO` | Mounted as read-write by a **single node** |
| `ReadOnlyMany` | `ROX` | Mounted as read-only by **many nodes** |
| `ReadWriteMany` | `RWX` | Mounted as read-write by **many nodes** |
| `ReadWriteOncePod` | `RWOP` | Mounted as read-write by a **single Pod** (K8s 1.27+) |

> [!IMPORTANT]
> Access modes are about **nodes**, not Pods (except RWOP). `ReadWriteOnce` means one *node* — multiple Pods on the same node can still mount it. Most cloud block storage (AWS EBS, GCP PD, Azure Disk) only supports `RWO`. For `RWX`, you need network-attached storage like NFS, CephFS, or cloud file storage (EFS, Filestore).

### PersistentVolumeClaim (PVC) — The Request

A PVC is a developer's request for storage. Kubernetes finds a PV that satisfies the request and binds them together.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-claim
spec:
  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 10Gi                 # I need at least 10Gi

  storageClassName: manual           # Must match the PV's storageClassName
```

### Binding Rules

Kubernetes matches a PVC to a PV based on:
1. **StorageClassName** must match
2. **Access modes** must be compatible
3. **Capacity**: PV must be >= PVC's requested size
4. **Label selectors** (optional, for specific PV targeting)

```powershell
kubectl get pv
# NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS
# pv-manual-50gi   50Gi       RWO            Retain           Bound    default/app-data-claim  manual

kubectl get pvc
# NAME             STATUS   VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# app-data-claim   Bound    pv-manual-50gi   50Gi       RWO            manual         30s
```

### Using a PVC in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-storage
spec:
  containers:
    - name: app
      image: nginx:1.27
      volumeMounts:
        - name: persistent-data
          mountPath: /usr/share/nginx/html    # Data survives Pod restarts
  volumes:
    - name: persistent-data
      persistentVolumeClaim:
        claimName: app-data-claim             # Reference the PVC, not the PV
```

---

## 5.4 — PV Lifecycle and Reclaim Policies

A PV goes through these phases:

```
Available → Bound → Released → (Retained / Deleted / Recycled)
```

| Phase | Meaning |
|---|---|
| `Available` | PV exists, no PVC is bound to it |
| `Bound` | PV is bound to a PVC |
| `Released` | PVC was deleted, but PV still holds the data |
| `Failed` | Automatic reclamation failed |

### Reclaim Policies

What happens to the PV (and its data) when the PVC is deleted:

| Policy | Behavior | Use Case |
|---|---|---|
| `Retain` | PV is kept with data intact. Admin must manually clean up and re-create. | Production databases — you never want automatic data deletion |
| `Delete` | PV and the underlying storage (cloud disk) are deleted automatically. | Dev/test environments, dynamically provisioned volumes |
| `Recycle` | ⚠️ **Deprecated**. Was `rm -rf /volume/*`. Don't use it. | Legacy — replaced by dynamic provisioning |

> [!WARNING]
> **`Delete` means `Delete`.** If your cloud PV has `reclaimPolicy: Delete` and someone runs `kubectl delete pvc my-claim`, the underlying EBS volume / GCP PD is permanently destroyed along with all its data. Set `Retain` for anything you care about.

---

## 5.5 — StorageClasses and Dynamic Provisioning

Manually creating PVs (static provisioning) doesn't scale. In production, you use **StorageClasses** to let Kubernetes create PVs automatically when a PVC is submitted.

### How Dynamic Provisioning Works

```
Developer creates PVC with storageClassName: "fast-ssd"
            ↓
StorageClass "fast-ssd" tells K8s: use the AWS EBS provisioner, create gp3 volumes
            ↓
K8s automatically creates a PV (and the underlying cloud disk)
            ↓
PVC is bound to the new PV
            ↓
Pod mounts the PVC — done
```

No admin intervention required.

### StorageClass Manifest

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs       # Cloud-specific provisioner
parameters:
  type: gp3                              # EBS volume type
  fsType: ext4
reclaimPolicy: Delete                    # Auto-delete when PVC is removed
allowVolumeExpansion: true               # Allow resizing PVCs later
volumeBindingMode: WaitForFirstConsumer  # Don't create disk until a Pod needs it
```

### Common Provisioners

| Cloud / Platform | Provisioner | Typical Parameters |
|---|---|---|
| AWS EBS | `ebs.csi.aws.com` | `type: gp3`, `iops: 3000` |
| GCP PD | `pd.csi.storage.gke.io` | `type: pd-ssd` |
| Azure Disk | `disk.csi.azure.com` | `skuName: Premium_LRS` |
| Minikube | `k8s.io/minikube-hostpath` | (default, auto-configured) |
| NFS | `nfs.csi.k8s.io` | NFS server + share path |

### Using Dynamic Provisioning (What You Write Day-to-Day)

With dynamic provisioning, you usually **skip creating PVs entirely**. Just create a PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: fast-ssd       # The StorageClass handles the rest
```

```powershell
kubectl apply -f pvc.yaml

# Watch the dynamic provisioning happen
kubectl get pvc my-data
# STATUS: Pending → Bound (within seconds)

kubectl get pv
# A new PV was auto-created!
# NAME                                       CAPACITY   STORAGECLASS   STATUS
# pvc-a1b2c3d4-5678-9012-abcd-ef1234567890   20Gi       fast-ssd       Bound
```

### Minikube's Default StorageClass

Minikube comes with a pre-configured StorageClass called `standard`:

```powershell
kubectl get storageclass
# NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE
# standard (default)   k8s.io/minikube-hostpath   Delete          Immediate
```

Since it's marked as `(default)`, any PVC that omits `storageClassName` will use it automatically:

```yaml
# This PVC will use the "standard" StorageClass on Minikube
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: auto-provisioned
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  # storageClassName omitted → uses the default StorageClass
```

> [!TIP]
> **Volume expansion**: If your StorageClass has `allowVolumeExpansion: true`, you can grow a PVC by editing its `resources.requests.storage`. The underlying disk is resized live. You can **never shrink** a PVC — only grow.

---

## 5.6 — StatefulSets: When Deployments Aren't Enough

Deployments treat Pods as interchangeable cattle — any Pod can be replaced by any other. This breaks for stateful applications where:
- Each Pod needs a **stable, unique identity** (hostname)
- Each Pod needs its **own persistent volume** (not shared)
- Pods must start and stop in **ordered sequence**
- Pods need **stable network identifiers** (DNS names)

Examples: databases (MySQL, PostgreSQL), distributed systems (Kafka, Elasticsearch, ZooKeeper), any clustered application with leader election.

### Deployment vs. StatefulSet

| Feature | Deployment | StatefulSet |
|---|---|---|
| **Pod names** | Random hash: `webapp-7d9f8b6c5f-abc12` | Ordinal index: `db-0`, `db-1`, `db-2` |
| **Startup order** | All at once (parallel) | Sequential: `db-0` → `db-1` → `db-2` |
| **Shutdown order** | All at once | Reverse: `db-2` → `db-1` → `db-0` |
| **Pod replacement** | New random name | Same name: if `db-1` dies, new `db-1` is created |
| **Storage** | Shared PVC (all pods use same volume) | Per-pod PVC via `volumeClaimTemplates` |
| **Network identity** | No stable hostname | Stable DNS: `db-0.db-headless.default.svc.cluster.local` |

### StatefulSet Manifest

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db-headless          # REQUIRED — the headless Service for DNS
  replicas: 3
  selector:
    matchLabels:
      app: database

  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql

  # THIS is the key difference — each Pod gets its OWN PVC
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard
        resources:
          requests:
            storage: 10Gi
```

### The Headless Service (Required for StatefulSets)

A **headless Service** (ClusterIP set to `None`) gives each Pod a stable DNS name:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-headless
spec:
  clusterIP: None                   # Headless — no load balancing, direct Pod DNS
  selector:
    app: database
  ports:
    - port: 3306
      targetPort: 3306
```

This creates DNS records:
```
db-0.db-headless.default.svc.cluster.local → Pod db-0's IP
db-1.db-headless.default.svc.cluster.local → Pod db-1's IP
db-2.db-headless.default.svc.cluster.local → Pod db-2's IP
```

When `db-1` is rescheduled (new IP), the DNS record updates automatically. The hostname stays the same.

### What Happens With volumeClaimTemplates

```powershell
kubectl get pvc
# NAME        STATUS   VOLUME                                     CAPACITY   STORAGECLASS
# data-db-0   Bound    pvc-111aaa-...                              10Gi       standard
# data-db-1   Bound    pvc-222bbb-...                              10Gi       standard
# data-db-2   Bound    pvc-333ccc-...                              10Gi       standard
```

Each Pod gets its own PVC named `<volumeClaimTemplate-name>-<statefulset-name>-<ordinal>`.

> [!IMPORTANT]
> **PVCs from volumeClaimTemplates are NOT deleted when the StatefulSet is deleted or scaled down.** This is intentional — you don't want to lose database data because someone scaled down. You must manually delete the PVCs if you want to reclaim storage:
> ```powershell
> kubectl delete pvc data-db-0 data-db-1 data-db-2
> ```

### Scaling StatefulSets

```powershell
# Scale up: db-3, db-4 are created sequentially
kubectl scale statefulset db --replicas=5

# Scale down: db-4 is deleted first, then db-3 (reverse order)
kubectl scale statefulset db --replicas=3
```

---

## 5.7 — Decision Guide: Which Storage Approach?

```
Do you need data to persist beyond the Pod's lifetime?
│
├── NO  → emptyDir (scratch space, shared between containers)
│
└── YES
    │
    ├── Is this a single-instance workload? (one Pod)
    │   └── Deployment + PVC (simple, good enough for dev/staging databases)
    │
    └── Is this a clustered/replicated workload? (multiple Pods each needing their own data)
        └── StatefulSet + volumeClaimTemplates + Headless Service
            (databases, Kafka, Elasticsearch, etc.)
```

---

## Phase 5 — kubectl Cheat Sheet

| Command | Purpose |
|---|---|
| `kubectl get pv` | List PersistentVolumes |
| `kubectl get pvc` | List PersistentVolumeClaims |
| `kubectl describe pv <name>` | PV details + bound PVC |
| `kubectl describe pvc <name>` | PVC details + bound PV + events |
| `kubectl get storageclass` / `kubectl get sc` | List StorageClasses |
| `kubectl describe sc <name>` | StorageClass details + provisioner |
| `kubectl get statefulset` / `kubectl get sts` | List StatefulSets |
| `kubectl describe sts <name>` | StatefulSet details + events |
| `kubectl rollout status sts/<name>` | Watch StatefulSet rollout |
| `kubectl scale sts <name> --replicas=N` | Scale a StatefulSet |
| `kubectl delete pvc <name>` | Delete a PVC (and possibly its PV) |
| `kubectl patch pvc <name> -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'` | Expand a PVC |
| `kubectl get events --field-selector involvedObject.kind=PersistentVolumeClaim` | PVC-related events |

> [!TIP]
> **Debugging storage issues**: When a PVC is stuck in `Pending`, run:
> ```powershell
> kubectl describe pvc <name>
> ```
> Check the Events section. Common causes:
> - No PV matches the request (wrong StorageClass, insufficient capacity, incompatible access mode)
> - StorageClass provisioner isn't installed
> - `volumeBindingMode: WaitForFirstConsumer` — PV won't be created until a Pod actually needs it

---

## 🔬 Lab Exercise 5: Deploy a Stateful MySQL Instance with Persistent Storage

### Objective

Deploy a MySQL database using both approaches — first with a simple Deployment + PVC, then with a StatefulSet — and observe the differences.

### Part A: Deployment + PVC (Single Instance)

#### Step 1: Create the Secret

Create `lab5-secret.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
stringData:
  root-password: "Lab5RootPass!"
```

#### Step 2: Create the PVC

Create `lab5-pvc.yaml`:
- **Name**: `mysql-data-claim`
- **Access mode**: `ReadWriteOnce`
- **Storage**: `2Gi`
- **StorageClass**: omit (use the Minikube default)

#### Step 3: Create the Deployment

Create `lab5-deployment.yaml`:
- **Name**: `mysql-deploy`
- **Replicas**: 1 (single instance)
- **Container**: `mysql:8.0`
- **Port**: 3306
- **Environment variable**: `MYSQL_ROOT_PASSWORD` from the secret key `root-password`
- **Volume mount**: Mount `mysql-data-claim` at `/var/lib/mysql`

#### Step 4: Apply and Test

```powershell
kubectl apply -f lab5-secret.yaml -f lab5-pvc.yaml -f lab5-deployment.yaml

# Wait for the pod to be ready
kubectl get pods -l app=mysql -w

# Verify the PVC is bound
kubectl get pvc mysql-data-claim

# Connect to MySQL and create test data
kubectl exec -it <mysql-pod-name> -- mysql -uroot -pLab5RootPass! -e "CREATE DATABASE lab5test; USE lab5test; CREATE TABLE messages (id INT, text VARCHAR(100)); INSERT INTO messages VALUES (1, 'Hello from Lab 5!');"

# Delete the pod (Deployment will recreate it)
kubectl delete pod <mysql-pod-name>

# Wait for the new pod, then verify data persisted
kubectl exec -it <new-mysql-pod-name> -- mysql -uroot -pLab5RootPass! -e "SELECT * FROM lab5test.messages;"
# You should see: 1 | Hello from Lab 5!
```

### Part B: StatefulSet (Replicated — Conceptual)

#### Step 5: Create the Headless Service

Create `lab5-headless-svc.yaml`:
- **Name**: `mysql-headless`
- **clusterIP**: `None`
- **Selector**: match your StatefulSet pod labels
- **Port**: 3306

#### Step 6: Create the StatefulSet

Create `lab5-statefulset.yaml`:
- **Name**: `mysql-sts`
- **serviceName**: `mysql-headless`
- **Replicas**: 2
- **Container**: `mysql:8.0`, port 3306
- **Env**: `MYSQL_ROOT_PASSWORD` from `mysql-secret`
- **volumeClaimTemplates**: name `data`, 2Gi, `ReadWriteOnce`
- **volumeMount**: mount `data` at `/var/lib/mysql`

#### Step 7: Apply and Observe

```powershell
kubectl apply -f lab5-headless-svc.yaml -f lab5-statefulset.yaml

# Watch pods come up IN ORDER
kubectl get pods -w
# mysql-sts-0   Running  (first)
# mysql-sts-1   Running  (only after mysql-sts-0 is Ready)

# See per-pod PVCs
kubectl get pvc
# data-mysql-sts-0   Bound   ...
# data-mysql-sts-1   Bound   ...

# Verify stable DNS names
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- nslookup mysql-sts-0.mysql-headless

# Delete mysql-sts-1 and watch it come back with the SAME name
kubectl delete pod mysql-sts-1
kubectl get pods -w
# mysql-sts-1   re-created (same name, same PVC)
```

### What to Submit

1. Your manifests: `lab5-pvc.yaml`, `lab5-deployment.yaml`, `lab5-statefulset.yaml`, `lab5-headless-svc.yaml`
2. Output of `kubectl get pvc` (showing both the standalone PVC and the auto-created StatefulSet PVCs)
3. Output proving data survived the Pod deletion in Part A (`SELECT * FROM lab5test.messages`)
4. Output of `kubectl get pods` showing the StatefulSet pods with ordinal names (`mysql-sts-0`, `mysql-sts-1`)
5. Output of `nslookup mysql-sts-0.mysql-headless` proving stable DNS

---

> **⏸️ CHECKPOINT**: Submit your lab results or request to skip. Last phase coming up: **Phase 6: Application Health, Security, & Troubleshooting** — probes, resource limits, RBAC, and the debugging toolkit that separates junior from senior K8s admins.

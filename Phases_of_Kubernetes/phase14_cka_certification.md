# Phase 14: CKA Certification Prep — Exam Strategy, Practice & Pricing

> **Goal**: Pass the Certified Kubernetes Administrator (CKA) exam. This phase covers the exact exam format, pricing, domain weights, speed techniques, practice scenarios, and a mock exam — everything you need to walk in confident.

---

## 14.1 — What Is the CKA?

The **Certified Kubernetes Administrator (CKA)** is a performance-based exam administered by the **Linux Foundation** in partnership with the **Cloud Native Computing Foundation (CNCF)**. It validates your ability to perform the responsibilities of a Kubernetes administrator.

Key fact: **This is NOT a multiple-choice exam.** You solve real problems on a live Kubernetes cluster using a terminal.

### Exam Details at a Glance

| Detail | Value |
|---|---|
| **Administered by** | Linux Foundation / CNCF |
| **Format** | Performance-based (hands-on, live cluster) |
| **Duration** | 2 hours |
| **Number of questions** | 15–20 tasks |
| **Passing score** | 66% |
| **Kubernetes version** | Current stable (check at exam time — as of 2025, ~v1.30+) |
| **Proctoring** | Online via PSI Bridge (webcam + screen share) |
| **Retakes** | 1 free retake included with purchase |
| **Certification validity** | 3 years |
| **Prerequisites** | None (but real hands-on experience recommended) |
| **Allowed resources** | kubernetes.io/docs, kubernetes.io/blog, github.com/kubernetes (open during exam) |
| **Language** | English, Japanese, Simplified Chinese, German, Spanish, Portuguese |

---

## 14.2 — Pricing

### CKA Exam Pricing (as of 2025)

| Item | Price (USD) |
|---|---|
| **CKA Exam** | **$395** |
| **CKA Exam + Killer.sh Simulator** | Included (2 simulator sessions come with exam purchase) |
| **CKA Retake** | **Free** (1 retake included with original purchase) |

### Bundle Deals (Save Money)

| Bundle | Includes | Price (USD) | Savings |
|---|---|---|---|
| **CKA + CKS Bundle** | CKA + Certified Kubernetes Security Specialist | **$595** | ~$195 off |
| **CKA + CKAD Bundle** | CKA + Certified Kubernetes Application Developer | **$595** | ~$195 off |
| **CKA + CKAD + CKS Bundle** | All three K8s certifications | **$845** | ~$340 off |
| **Kubestronaut Bundle** | CKA + CKAD + CKS + KCNA + KCSA (all 5) | **$1,095** | ~$580 off |

> [!TIP]
> **Watch for sales.** The Linux Foundation runs major discounts (30-50% off) during:
> - **Black Friday / Cyber Monday** (November)
> - **KubeCon** events (March/April and November)
> - **Kubernetes Anniversary** (June)
> - **End-of-year sales** (December)
>
> A CKA exam that costs $395 can often be purchased for **$198–$237** during these sales. Worth waiting for if you're not in a rush.

### Additional Costs (Optional but Recommended)

| Resource | Price | Notes |
|---|---|---|
| **Killer.sh Simulator** | Included with exam | 2 sessions, 36 hours each. Questions harder than the real exam. |
| **KodeKloud CKA Course** | $16.60/mo or ~$200/year | Highly rated, includes built-in practice labs |
| **Udemy — Mumshad Mannambeth CKA Course** | ~$15-20 (on sale) | Most popular CKA course, includes KodeKloud lab access |
| **A Cloud Guru / Pluralsight** | ~$30/mo | CKA learning path included |
| **Killer Koda (free)** | Free | Browser-based K8s playgrounds for practice |
| **Local Minikube/Kind** | Free | Practice on your own machine |

### Related Kubernetes Certifications

| Certification | Focus | Price | Difficulty |
|---|---|---|---|
| **KCNA** (Kubernetes & Cloud Native Associate) | Foundational knowledge (multiple choice) | $250 | ★★☆☆☆ |
| **CKAD** (Certified Kubernetes Application Developer) | Deploying and configuring apps | $395 | ★★★☆☆ |
| **CKA** (Certified Kubernetes Administrator) | Cluster administration | $395 | ★★★★☆ |
| **CKS** (Certified Kubernetes Security Specialist) | Security hardening (requires CKA first) | $395 | ★★★★★ |
| **KCSA** (Kubernetes & Cloud Security Associate) | Security fundamentals (multiple choice) | $250 | ★★☆☆☆ |

> [!NOTE]
> **Kubestronaut**: Pass all 5 (KCNA + CKAD + CKA + CKS + KCSA) and earn the "Kubestronaut" title — a community recognition from CNCF. You get a special badge, jacket, and listing on the CNCF Kubestronaut page.

---

## 14.3 — The Exam Environment

### What You'll See

- A **Linux terminal** (Ubuntu-based) in your browser
- Access to **one or more Kubernetes clusters** (you switch between them using `kubectl config use-context`)
- A **built-in notepad** (for copying/pasting within the exam)
- Access to **kubernetes.io documentation** in a browser tab within the exam environment

### What's Pre-Installed

| Tool | Available |
|---|---|
| `kubectl` | ✅ Yes, with autocompletion |
| `vim` / `nano` | ✅ Yes |
| `tmux` | ✅ Yes |
| `jq` | ✅ Yes |
| `curl` / `wget` | ✅ Yes |
| `systemctl` | ✅ Yes (for kubelet/node tasks) |
| `kubeadm` | ✅ Yes (for cluster tasks) |
| `etcdctl` | ✅ Yes (for backup/restore) |
| `openssl` | ✅ Yes (for certificate tasks) |
| Helm | ❌ Not guaranteed |

### Proctoring Rules

- Webcam ON the entire time (face visible)
- Clean desk, no second monitors
- No one else in the room
- Government-issued photo ID required
- No headphones/earbuds
- You CAN drink water (clear glass/bottle)
- You CAN use the built-in notepad
- You CANNOT use local files, bookmarks, or notes
- You CANNOT use ChatGPT, Stack Overflow, or any other resource outside kubernetes.io

---

## 14.4 — Exam Domains and Weights

The CKA exam covers these domains with the following weights:

| Domain | Weight | Key Topics |
|---|---|---|
| **Cluster Architecture, Installation & Configuration** | 25% | kubeadm cluster setup, RBAC, upgrade clusters, etcd backup/restore |
| **Workloads & Scheduling** | 15% | Deployments, rolling updates, ConfigMaps, Secrets, resource limits, node affinity, taints/tolerations |
| **Services & Networking** | 20% | Services (ClusterIP/NodePort/LB), Ingress, NetworkPolicies, DNS, CNI basics |
| **Storage** | 10% | PV, PVC, StorageClass, volume types |
| **Troubleshooting** | 30% | Application failure, cluster component failure, networking issues, node troubleshooting |

```
Troubleshooting ██████████████████████████████ 30%
Cluster Setup   █████████████████████████ 25%
Networking      ████████████████████ 20%
Workloads       ███████████████ 15%
Storage         ██████████ 10%
```

> [!IMPORTANT]
> **Troubleshooting is 30%** — the single largest domain. If you can debug pods, nodes, and networking quickly, you're already a third of the way to passing.

---

## 14.5 — Domain-by-Domain Study Guide

### Domain 1: Cluster Architecture, Installation & Configuration (25%)

#### What You Must Be Able to Do

| Skill | Commands/Concepts |
|---|---|
| **Bootstrap a cluster with kubeadm** | `kubeadm init`, `kubeadm join`, passing flags |
| **Upgrade a cluster** | `kubeadm upgrade plan`, `kubeadm upgrade apply`, drain nodes, uncordon |
| **Manage RBAC** | Create Roles, ClusterRoles, RoleBindings, ClusterRoleBindings, ServiceAccounts |
| **Backup and restore etcd** | `etcdctl snapshot save`, `etcdctl snapshot restore` |
| **Manage kubeconfig files** | `kubectl config use-context`, set-cluster, set-credentials |

#### etcd Backup and Restore (High-Priority Exam Topic)

```bash
# Backup etcd
ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify the backup
ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-backup.db --write-table

# Restore etcd (to a new data directory)
ETCDCTL_API=3 etcdctl snapshot restore /tmp/etcd-backup.db \
  --data-dir=/var/lib/etcd-restored

# Then update the etcd static pod manifest to use the new data directory:
# Edit /etc/kubernetes/manifests/etcd.yaml → change --data-dir and hostPath
```

#### Cluster Upgrade (Step-by-Step)

```bash
# 1. Upgrade kubeadm on the control plane
apt update && apt install -y kubeadm=1.30.0-1.1

# 2. Plan the upgrade
kubeadm upgrade plan

# 3. Apply the upgrade
kubeadm upgrade apply v1.30.0

# 4. Drain the control plane node
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# 5. Upgrade kubelet and kubectl
apt install -y kubelet=1.30.0-1.1 kubectl=1.30.0-1.1
systemctl daemon-reload && systemctl restart kubelet

# 6. Uncordon the node
kubectl uncordon <node-name>

# 7. Repeat steps 1, 4-6 for each worker node
```

---

### Domain 2: Workloads & Scheduling (15%)

#### What You Must Be Able to Do

| Skill | Commands/Concepts |
|---|---|
| **Create and manage Deployments** | `kubectl create deployment`, scale, rolling updates, rollbacks |
| **ConfigMaps and Secrets** | Create from literals/files, mount as env vars/volumes |
| **Resource limits** | requests/limits, LimitRanges, ResourceQuotas |
| **Node affinity** | `nodeSelector`, `nodeAffinity`, `podAffinity`, `podAntiAffinity` |
| **Taints and Tolerations** | `kubectl taint`, toleration specs |
| **Static Pods** | Create manifests in `/etc/kubernetes/manifests/` |
| **DaemonSets** | Deploy a pod on every node |
| **Jobs and CronJobs** | One-off and scheduled batch tasks |

#### Taints and Tolerations

```bash
# Taint a node (NoSchedule = new pods won't be scheduled)
kubectl taint node worker-1 key=value:NoSchedule

# Remove a taint
kubectl taint node worker-1 key=value:NoSchedule-

# Toleration in a Pod spec
tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
```

#### Node Affinity

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: disktype
              operator: In
              values:
                - ssd
```

---

### Domain 3: Services & Networking (20%)

#### What You Must Be Able to Do

| Skill | Commands/Concepts |
|---|---|
| **Expose Deployments** | `kubectl expose`, Service types (ClusterIP, NodePort, LB) |
| **NetworkPolicies** | Ingress/egress rules, podSelector, namespaceSelector |
| **Ingress** | Ingress resources, Ingress controllers |
| **DNS** | Service DNS names, troubleshoot DNS with nslookup/dig |
| **CNI** | Basic understanding, install/verify CNI plugin |

#### Quick NetworkPolicy Template

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-specific
  namespace: target-ns
spec:
  podSelector:
    matchLabels:
      app: target
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: allowed-source
      ports:
        - port: 80
          protocol: TCP
```

---

### Domain 4: Storage (10%)

#### What You Must Be Able to Do

| Skill | Commands/Concepts |
|---|---|
| **Create PV and PVC** | YAML manifests, binding, access modes |
| **StorageClasses** | Dynamic provisioning, default StorageClass |
| **Mount volumes in Pods** | volumeMounts, volumes, subPath |
| **Expand PVCs** | Patch PVC resources, allowVolumeExpansion |

---

### Domain 5: Troubleshooting (30%)

#### What You Must Be Able to Do

| Skill | Commands |
|---|---|
| **Debug failing Pods** | `describe`, `logs`, `logs --previous`, `exec`, `get events` |
| **Debug failing nodes** | `kubectl get nodes`, `describe node`, `ssh` to node, check `kubelet` |
| **Fix kubelet issues** | `systemctl status kubelet`, `journalctl -u kubelet`, check certs/config |
| **Fix networking** | DNS resolution, Service endpoints, kube-proxy, CNI |
| **Fix control plane** | Static pod manifests in `/etc/kubernetes/manifests/`, etcd health |

#### The Troubleshooting Flowchart

```
Problem: Pod not running
├── Status: Pending
│   ├── Check: kubectl describe pod → Events
│   ├── Cause: Insufficient resources → Free up resources or add nodes
│   ├── Cause: PVC not bound → Fix PVC/PV/StorageClass
│   └── Cause: Taint prevents scheduling → Add toleration or remove taint
│
├── Status: ImagePullBackOff
│   ├── Check: kubectl describe pod → Events
│   └── Cause: Wrong image name/tag or missing imagePullSecret
│
├── Status: CrashLoopBackOff
│   ├── Check: kubectl logs <pod> --previous
│   ├── Cause: Application error → Fix app config
│   ├── Cause: Wrong command/args → Fix in spec
│   └── Cause: Missing ConfigMap/Secret → Create missing resource
│
├── Status: Running but not Ready
│   ├── Check: kubectl describe pod → Readiness probe
│   └── Cause: Readiness probe failing → Fix probe or app
│
└── Node: NotReady
    ├── Check: kubectl describe node → Conditions
    ├── SSH to node → systemctl status kubelet
    ├── Check: journalctl -u kubelet -n 50
    ├── Cause: kubelet stopped → systemctl start kubelet
    ├── Cause: kubelet misconfigured → check /var/lib/kubelet/config.yaml
    └── Cause: Certificate expired → renew with kubeadm certs renew
```

---

## 14.6 — Speed Techniques (Critical for Time Management)

You have **2 hours for 15-20 tasks**. That's **6-8 minutes per task**. Speed matters.

### Essential Shell Setup (Do This FIRST in the Exam)

```bash
# Alias kubectl (saves hundreds of keystrokes)
alias k=kubectl

# Enable autocomplete
source <(kubectl completion bash)
complete -o default -F __start_kubectl k

# Quick shorthand aliases
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deployments'
alias kgn='kubectl get nodes'
alias kd='kubectl describe'
alias kdp='kubectl describe pod'
alias kl='kubectl logs'
alias ka='kubectl apply -f'
alias kdel='kubectl delete'

# Set default namespace for the current context
kubectl config set-context --current --namespace=<namespace>

# Export for easier YAML generation
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"
```

### Generate YAML Instead of Writing From Scratch

**NEVER write YAML from memory in the exam.** Generate it:

```bash
# Generate a Pod YAML
k run nginx --image=nginx:1.27 $do > pod.yaml

# Generate a Deployment YAML
k create deployment webapp --image=nginx:1.27 --replicas=3 $do > deploy.yaml

# Generate a Service YAML (ClusterIP)
k expose deployment webapp --port=80 --target-port=8080 $do > svc.yaml

# Generate a NodePort Service
k expose deployment webapp --port=80 --type=NodePort $do > svc.yaml

# Generate a Job
k create job backup --image=busybox -- /bin/sh -c "echo hello" $do > job.yaml

# Generate a CronJob
k create cronjob backup --image=busybox --schedule="*/5 * * * *" -- /bin/sh -c "echo hello" $do > cron.yaml

# Generate a ConfigMap
k create configmap app-config --from-literal=key1=val1 --from-literal=key2=val2 $do > cm.yaml

# Generate a Secret
k create secret generic db-creds --from-literal=password=pass123 $do > secret.yaml

# Generate a ServiceAccount
k create sa my-sa $do > sa.yaml

# Generate a Role
k create role pod-reader --verb=get,list,watch --resource=pods $do > role.yaml

# Generate a RoleBinding
k create rolebinding pod-reader-binding --role=pod-reader --serviceaccount=default:my-sa $do > rb.yaml

# Generate a ClusterRole
k create clusterrole node-reader --verb=get,list --resource=nodes $do > cr.yaml

# Generate a ClusterRoleBinding
k create clusterrolebinding node-reader-binding --clusterrole=node-reader --serviceaccount=default:my-sa $do > crb.yaml

# Generate an Ingress
k create ingress my-ingress --rule="myhost.com/=my-svc:80" $do > ingress.yaml

# Generate a NetworkPolicy (not available — use docs template)
```

> [!IMPORTANT]
> **The `$do` export is your biggest time saver.** Instead of writing 30 lines of YAML, you generate 90% of it in one command, then edit the remaining 10%. This alone can save you 20-30 minutes across the exam.

### Vim Essentials (The Exam Uses vim)

```vim
" YAML mode settings (add to ~/.vimrc at exam start)
set tabstop=2
set shiftwidth=2
set expandtab
set autoindent
set number

" Useful commands
:set paste        " Paste without auto-indent corruption
:set nopaste      " Back to normal mode
dd                " Delete current line
yy                " Copy current line
p                 " Paste below
5dd               " Delete 5 lines
u                 " Undo
Ctrl+r            " Redo
/pattern          " Search
:%s/old/new/g     " Find and replace all
:wq               " Save and quit
```

---

## 14.7 — Time Management Strategy

### Recommended Approach

```
Minutes 0-5:     Set up aliases, vim config, explore the environment
Minutes 5-90:    Work through questions IN ORDER, skip anything >8 minutes
Minutes 90-110:  Return to skipped questions
Minutes 110-120: Review flagged answers, verify resources exist
```

### Scoring Strategy

| Question Weight | Time Budget | Strategy |
|---|---|---|
| 2-4% | 2-3 minutes | Quick wins. Don't skip these. |
| 5-7% | 5-6 minutes | Standard questions. Do thoroughly. |
| 8-13% | 8-10 minutes | Hard questions. If stuck after 5 min, flag and move on. |

### Critical Rules

1. **Read the question COMPLETELY** before typing. Missed requirements = lost points.
2. **Check which context/cluster** each question specifies. `kubectl config use-context <name>` FIRST.
3. **Check which namespace** the question specifies. Set it: `k config set-context --current --namespace=<ns>`
4. **Verify your work.** After applying: `k get <resource>`, `k describe <resource>`, check events.
5. **Partial credit exists.** If you can't complete everything, get as far as you can.
6. **Use the documentation.** kubernetes.io/docs is your open-book reference. Bookmark nothing — use the search bar.

---

## 14.8 — Practice Scenarios (Exam-Style)

### Scenario 1: RBAC (Typical 5-7% question)

> Create a ServiceAccount named `deploy-sa` in namespace `apps`. Create a Role named `deploy-role` that allows `get`, `list`, `create`, and `delete` on `deployments` in the `apps` namespace. Bind the Role to the ServiceAccount.

```bash
k create ns apps
k create sa deploy-sa -n apps
k create role deploy-role --verb=get,list,create,delete --resource=deployments -n apps
k create rolebinding deploy-role-binding --role=deploy-role --serviceaccount=apps:deploy-sa -n apps

# Verify
k auth can-i create deployments --as=system:serviceaccount:apps:deploy-sa -n apps
# yes
```

### Scenario 2: Troubleshoot a Node (Typical 8-13% question)

> Node `worker-2` is showing `NotReady`. Investigate and fix the issue.

```bash
# Step 1: Check node status
k get nodes
k describe node worker-2    # Look at Conditions and Events

# Step 2: SSH to the node
ssh worker-2

# Step 3: Check kubelet
systemctl status kubelet
# If inactive/dead:
systemctl start kubelet
systemctl enable kubelet

# If still failing, check logs:
journalctl -u kubelet -n 50

# Common fixes:
# - Wrong config: check /var/lib/kubelet/config.yaml
# - Expired cert: kubeadm certs renew all
# - Wrong CA: check --kubeconfig path
```

### Scenario 3: etcd Backup (Typical 5-7% question)

> Create a snapshot of the etcd database and save it to `/opt/etcd-backup.db`. The etcd is running on the control plane node with certificates at `/etc/kubernetes/pki/etcd/`.

```bash
ETCDCTL_API=3 etcdctl snapshot save /opt/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify
ETCDCTL_API=3 etcdctl snapshot status /opt/etcd-backup.db --write-table
```

### Scenario 4: NetworkPolicy (Typical 7% question)

> Create a NetworkPolicy named `allow-web` in namespace `production` that allows ingress traffic to pods labeled `app=web` only from pods labeled `app=api` on port 80.

```bash
cat <<EOF | k apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api
      ports:
        - port: 80
          protocol: TCP
EOF
```

### Scenario 5: Create a Multi-Container Pod (Typical 4-5% question)

> Create a pod named `multi-pod` in namespace `default` with two containers:
> - Container 1: name `nginx`, image `nginx:1.27`, port 80
> - Container 2: name `sidecar`, image `busybox:1.36`, command `["sh", "-c", "while true; do echo heartbeat; sleep 5; done"]`

```bash
# Generate base YAML, then edit to add second container
k run multi-pod --image=nginx:1.27 --port=80 $do > multi-pod.yaml
# Edit to add the sidecar container
vim multi-pod.yaml
k apply -f multi-pod.yaml
```

### Scenario 6: Persistent Volume (Typical 5% question)

> Create a PersistentVolume named `pv-log` with 100Mi capacity, `ReadWriteMany` access mode, hostPath `/pv/log`. Create a PVC named `pvc-log` that requests 50Mi. Mount it in a pod.

```yaml
# PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-log
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: /pv/log
---
# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-log
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Mi
```

### Scenario 7: Cluster Upgrade (Typical 8-13% question)

> Upgrade the control plane node from v1.29.0 to v1.30.0. Upgrade kubelet and kubectl as well.

```bash
# On the control plane:
apt update
apt install -y kubeadm=1.30.0-1.1
kubeadm upgrade plan
kubeadm upgrade apply v1.30.0

kubectl drain controlplane --ignore-daemonsets --delete-emptydir-data
apt install -y kubelet=1.30.0-1.1 kubectl=1.30.0-1.1
systemctl daemon-reload
systemctl restart kubelet
kubectl uncordon controlplane
```

---

## 14.9 — kubernetes.io Documentation Navigation

Since the docs are open during the exam, know where to find things fast:

| Topic | Documentation Path |
|---|---|
| **Pod spec** | Concepts → Workloads → Pods |
| **Deployments** | Concepts → Workloads → Deployments |
| **Services** | Concepts → Services, Load Balancing, and Networking → Service |
| **Ingress** | Concepts → Services → Ingress |
| **NetworkPolicy** | Concepts → Services → Network Policies |
| **PV/PVC** | Concepts → Storage → Persistent Volumes |
| **ConfigMaps** | Concepts → Configuration → ConfigMaps |
| **Secrets** | Concepts → Configuration → Secrets |
| **RBAC** | Reference → API Access Control → RBAC |
| **kubeadm** | Reference → Setup Tools → kubeadm |
| **etcd backup** | Tasks → Administer a Cluster → Operating etcd |
| **Cluster upgrade** | Tasks → Administer a Cluster → Upgrade a Cluster |
| **Taints/Tolerations** | Concepts → Scheduling → Taints and Tolerations |
| **Node Affinity** | Concepts → Scheduling → Assigning Pods to Nodes |
| **Static Pods** | Tasks → Configure Pods → Create Static Pods |
| **Troubleshooting** | Tasks → Troubleshooting |

> [!TIP]
> **Use the search bar on kubernetes.io.** Don't navigate the menu tree — it's slow. Type "NetworkPolicy example" or "etcd backup" in the search and get straight to the relevant page. Bookmark nothing — the search is faster.

---

## 14.10 — Recommended Study Plan

### 4-Week Preparation Plan

| Week | Focus | Activities |
|---|---|---|
| **Week 1** | Foundation review | Complete Phases 1-6 of this curriculum. Set up Minikube. Practice `kubectl` daily. |
| **Week 2** | Core exam topics | RBAC, etcd backup/restore, cluster upgrades, NetworkPolicies, PV/PVC. Practice imperative generation (`$do` pattern). |
| **Week 3** | Troubleshooting + Speed | Practice troubleshooting scenarios (break things, fix them). Time yourself. Do Killer.sh Session 1. |
| **Week 4** | Mock exams + Review | Killer.sh Session 2. Review weak areas. Practice the full time management strategy. |

### Recommended Resources

| Resource | Type | Cost | Rating |
|---|---|---|---|
| **Mumshad Mannambeth — CKA (Udemy)** | Video course + labs | ~$15 on sale | ⭐⭐⭐⭐⭐ (Best overall) |
| **KodeKloud CKA Course** | Video + interactive labs | $16.60/mo | ⭐⭐⭐⭐⭐ (Best labs) |
| **Killer.sh** | Exam simulator | Included with exam | ⭐⭐⭐⭐⭐ (Must do) |
| **Killer Koda** | Free K8s playgrounds | Free | ⭐⭐⭐⭐ (Good practice) |
| **This curriculum (Phases 1-13)** | Structured reference | Free | You're here! |
| **kubernetes.io/docs** | Official docs | Free | Reference bible |
| **CKA Exercises (GitHub — dgkanatsios)** | Practice questions | Free | ⭐⭐⭐⭐ |

---

## 14.11 — Day-of-Exam Checklist

### Before the Exam

- [ ] Government-issued photo ID ready
- [ ] Webcam working, good lighting on face
- [ ] Stable internet connection (wired preferred)
- [ ] Clean desk, no papers/notes/books visible
- [ ] Close all applications except the exam browser
- [ ] Clear browser cache and tabs
- [ ] Water in a clear container (only allowed drink)
- [ ] Quiet room, no one else present
- [ ] Test PSI Bridge software 24 hours before
- [ ] Know your `.vimrc` and alias setup by heart

### First 5 Minutes of the Exam

```bash
# 1. Set up aliases
alias k=kubectl
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"

# 2. Enable autocomplete
source <(kubectl completion bash)
complete -o default -F __start_kubectl k

# 3. Set vim config
cat >> ~/.vimrc << 'EOF'
set tabstop=2
set shiftwidth=2
set expandtab
set autoindent
set number
EOF

# 4. Check available contexts
k config get-contexts

# 5. Start working
```

---

## 🔬 Lab Exercise 14 (Capstone Mock Exam): 10 CKA-Style Questions

### Rules
- Time limit: **90 minutes** (slightly harder than real exam's pace)
- Use only `kubernetes.io/docs` as reference
- Set up your aliases first
- Track your time per question

---

### Question 1 (4%) — Create a Pod

Create a pod named `exam-pod` in namespace `exam` using image `redis:7` with a label `tier=cache`. The namespace must be created if it doesn't exist.

---

### Question 2 (5%) — ConfigMap and Secret

In namespace `exam`:
- Create a ConfigMap named `app-config` with key `APP_MODE=production`
- Create a Secret named `app-secret` with key `DB_PASS=exampass123`
- Create a pod named `config-pod` using image `nginx:1.27` that mounts the ConfigMap key as env var `APP_MODE` and the Secret key as env var `DB_PASS`

---

### Question 3 (7%) — RBAC

In namespace `secure`:
- Create ServiceAccount `audit-sa`
- Create a Role `audit-role` allowing `get`, `list`, `watch` on `pods` and `services`
- Create a RoleBinding `audit-binding` that binds `audit-role` to `audit-sa`
- Verify: `kubectl auth can-i list pods --as=system:serviceaccount:secure:audit-sa -n secure` should return `yes`

---

### Question 4 (7%) — Deployment with Rolling Update

Create a Deployment named `web-deploy` in namespace `exam`:
- 4 replicas
- Image: `nginx:1.26`
- Strategy: RollingUpdate, maxSurge=1, maxUnavailable=0
- Then upgrade the image to `nginx:1.27` and verify the rollout succeeds

---

### Question 5 (7%) — NetworkPolicy

In namespace `exam`, create a NetworkPolicy named `restrict-db`:
- Applied to pods labeled `role=database`
- Allow ingress ONLY from pods labeled `role=backend` on port 3306
- Deny all other ingress

---

### Question 6 (7%) — PersistentVolume and PVC

- Create a PV named `exam-pv`: 200Mi, `ReadWriteOnce`, hostPath `/mnt/exam-data`
- Create a PVC named `exam-pvc` in namespace `exam`: 100Mi, `ReadWriteOnce`
- Create a pod `storage-pod` in namespace `exam` using image `busybox:1.36` that mounts `exam-pvc` at `/data` and runs `sleep 3600`

---

### Question 7 (8%) — Troubleshoot a Failing Pod

A pod named `broken-pod` exists in namespace `exam` but is in `CrashLoopBackOff`. Debug and fix it.

Create this pod first to simulate the issue:
```bash
k run broken-pod -n exam --image=busybox:1.36 --command -- /bin/sh -c "cat /config/app.conf"
```
(It crashes because `/config/app.conf` doesn't exist. Fix it by creating a ConfigMap with the file and mounting it.)

---

### Question 8 (8%) — Expose and Ingress

- Expose `web-deploy` (from Q4) as a ClusterIP Service named `web-svc` on port 80
- Create an Ingress named `web-ingress` that routes `exam.local/` to `web-svc` on port 80

---

### Question 9 (13%) — etcd Backup

Take a snapshot of etcd and save it to `/opt/exam-etcd-backup.db`.

etcd endpoint: `https://127.0.0.1:2379`
CA cert: `/etc/kubernetes/pki/etcd/ca.crt`
Server cert: `/etc/kubernetes/pki/etcd/server.crt`
Server key: `/etc/kubernetes/pki/etcd/server.key`

Verify the snapshot.

---

### Question 10 (13%) — Node Troubleshooting

Worker node `worker-1` is `NotReady`. SSH into the node, diagnose the issue, and bring the node back to `Ready` status.

Simulate:
```bash
# On the worker node:
ssh worker-1
systemctl stop kubelet
```

Fix it and verify `kubectl get nodes` shows `Ready`.

---

### Scoring Guide

| Question | Points | Topic | Difficulty |
|---|---|---|---|
| Q1 | 4% | Pod creation | Easy |
| Q2 | 5% | ConfigMap + Secret | Easy |
| Q3 | 7% | RBAC | Medium |
| Q4 | 7% | Deployment + rolling update | Medium |
| Q5 | 7% | NetworkPolicy | Medium |
| Q6 | 7% | PV/PVC | Medium |
| Q7 | 8% | Troubleshooting | Medium-Hard |
| Q8 | 8% | Service + Ingress | Medium |
| Q9 | 13% | etcd backup | Hard |
| Q10 | 13% | Node troubleshooting | Hard |

**Target**: Complete all 10 in 90 minutes. Passing = 66% (get at least Q1-Q8 right).

---

> ## 🎓 FULL CURRICULUM COMPLETE
>
> ### Your Journey: 14 Phases
>
> | Phase | Topic | Level |
> |---|---|---|
> | 1 | Architecture & Pod Fundamentals | Beginner |
> | 2 | Declarative Deployments & Scaling | Beginner |
> | 3 | Service Discovery & Networking | Beginner |
> | 4 | Configuration & Secrets | Beginner |
> | 5 | Storage & Persistence | Intermediate |
> | 6 | Health, Security & Troubleshooting | Intermediate |
> | 7 | Helm | Advanced |
> | 8 | Network Policies | Advanced |
> | 9 | HPA & Autoscaling | Advanced |
> | 10 | Custom Resource Definitions | Advanced |
> | 11 | Operators | Advanced |
> | 12 | GitOps with ArgoCD | Advanced |
> | 13 | Observability Stack | Advanced |
> | 14 | CKA Certification Prep | Certification |
>
> ### What's Next?
>
> 1. **Complete the labs** — Go back and do the hands-on exercises for every phase
> 2. **Buy the CKA exam** — Watch for Linux Foundation sales (30-50% off)
> 3. **Practice daily** — 1 hour of `kubectl` practice daily for 4 weeks
> 4. **Do Killer.sh** — Use both sessions, review every wrong answer
> 5. **Take the exam** — You're ready
>
> Good luck, future CKA. 🚀

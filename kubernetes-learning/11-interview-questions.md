# Kubernetes Interview Questions and Answers

## Table of Contents
1. [Architecture](#architecture)
2. [Pods and Controllers](#pods-and-controllers)
3. [Services and Networking](#services-and-networking)
4. [Storage](#storage)
5. [Configuration and Secrets](#configuration-and-secrets)
6. [Scheduling](#scheduling)
7. [Security and RBAC](#security-and-rbac)
8. [Scaling](#scaling)
9. [Helm and Package Management](#helm)
10. [Troubleshooting](#troubleshooting)
11. [Advanced Scenarios](#advanced-scenarios)

---

## Architecture

### Q1: Explain the Kubernetes architecture.

**Answer:**
Kubernetes follows a master-worker architecture:

**Control Plane (Master):**
- **kube-apiserver:** Front-end for the control plane, handles all REST requests, the only component that talks to etcd
- **etcd:** Consistent, highly-available key-value store for all cluster data
- **kube-scheduler:** Assigns pods to nodes based on resource requirements and constraints
- **kube-controller-manager:** Runs controller processes (node, replication, endpoints, etc.)
- **cloud-controller-manager:** Cloud-specific control logic (routes, load balancers)

**Worker Nodes:**
- **kubelet:** Agent that ensures containers are running in a Pod
- **kube-proxy:** Network proxy that maintains network rules
- **Container Runtime:** Software that runs containers (containerd, CRI-O)

### Q2: What is the role of etcd in Kubernetes?

**Answer:**
etcd is a distributed key-value store that stores ALL cluster state:
- Configuration data (deployments, services, etc.)
- Cluster state (which pods are running, where)
- Secrets and ConfigMaps
- Node information

**Critical points:**
- Uses Raft consensus algorithm
- Should always have odd number of members (3, 5, 7)
- Must be backed up regularly
- If lost, entire cluster state is lost
- Only kube-apiserver communicates with etcd

### Q3: What happens when you run `kubectl apply -f pod.yaml`?

**Answer:**
1. kubectl reads the YAML and sends it to kube-apiserver via HTTPS
2. API server authenticates the user and authorizes the action
3. Admission controllers validate and may mutate the object
4. The Pod object is stored in etcd
5. kube-scheduler sees the new unscheduled Pod
6. Scheduler evaluates all nodes and selects the best one
7. Scheduler updates the Pod spec with nodeName via API server
8. API server stores the update in etcd
9. kubelet on the selected node sees the Pod (via watch)
10. kubelet instructs the container runtime to pull the image and start containers
11. kubelet reports Pod status back to API server
12. Status is updated in etcd

### Q4: What is the reconciliation loop?

**Answer:**
The reconciliation loop is the core pattern used by all Kubernetes controllers:

1. **Observe** the desired state (what you declared in YAML)
2. **Observe** the current state (what exists in the cluster)
3. **Compare** the two states (find the diff)
4. **Act** to make current state match desired state
5. **Repeat** continuously

This is **level-triggered** (not edge-triggered), meaning:
- Controllers can restart and still converge to the correct state
- Applying the same configuration multiple times is idempotent
- The system is self-healing

### Q5: What are the different types of Kubernetes services in the control plane?

**Answer:**
Internal K8s services:
- `kubernetes` (API server) - default namespace, ClusterIP
- `kube-dns` / `coredns` - kube-system namespace
- `metrics-server` - kube-system namespace

The default `kubernetes` Service points to the API server and is automatically created:
```
kubernetes.default.svc.cluster.local → 10.96.0.1
```

---

## Pods and Controllers

### Q6: What is a Pod? Why not run containers directly?

**Answer:**
A Pod is the smallest deployable unit in Kubernetes. It groups one or more containers that:
- Share the same network namespace (same IP, same port space)
- Share volumes
- Share lifecycle (created, scheduled, terminated together)

**Why Pods instead of direct containers:**
1. **Sidecar pattern** - Helper containers (logging, proxying) alongside main app
2. **Shared context** - Containers can communicate via localhost
3. **Atomic scheduling** - All containers scheduled together on same node
4. **Pause container** - Provides network namespace isolation

### Q7: Explain Pod lifecycle phases.

**Answer:**
| Phase | Description |
|-------|-------------|
| **Pending** | Pod accepted, containers not yet created (scheduling, image pull) |
| **Running** | At least one container is running or starting |
| **Succeeded** | All containers terminated successfully (exit code 0) |
| **Failed** | All containers terminated, at least one failed |
| **Unknown** | Pod state cannot be determined |

### Q8: What are the different types of probes?

**Answer:**

| Probe | Purpose | On Failure |
|-------|---------|------------|
| **livenessProbe** | Is the container alive/healthy? | Restart the container |
| **readinessProbe** | Is the container ready to serve traffic? | Remove from Service endpoints |
| **startupProbe** | Has the application finished starting? | No action; disables liveness/readiness until it succeeds |

**When to use startupProbe:**
For slow-starting applications. Without it, a liveness probe with a long `initialDelaySeconds` might delay restarts. With startupProbe:
```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
  # App has up to 300 seconds to start before being killed
```

### Q9: What is the difference between Deployment, StatefulSet, and DaemonSet?

**Answer:**

| Feature | Deployment | StatefulSet | DaemonSet |
|---------|-----------|-------------|-----------|
| **Pod naming** | Random hash | Sequential (app-0, app-1) | Random hash |
| **Pod identity** | Interchangeable | Stable, predictable | Interchangeable |
| **Storage** | Shared or none | Individual PVC per pod | Host-based |
| **Scaling** | Any number | Ordered (0→N up, N→0 down) | One per node |
| **Network** | Single Service | Headless Service | Single Service |
| **Use case** | Stateless apps | Databases, queues | Log agents, monitoring |

### Q10: How does a Rolling Update work?

**Answer:**
A RollingUpdate gradually replaces old pods with new ones:

1. Deployment creates a new ReplicaSet with the updated pod template
2. New ReplicaSet scales UP by `maxSurge` pods
3. Each new pod must pass readiness probe
4. Old ReplicaSet scales DOWN by `maxUnavailable` pods
5. Steps 2-4 repeat until all pods are updated
6. Old ReplicaSet is kept for rollback

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # Max pods above desired (can be 25%)
    maxUnavailable: 0  # Max pods unavailable (can be 25%)
```

### Q11: What are Init Containers?

**Answer:**
Init containers run BEFORE the main application containers and must complete successfully.

**Key characteristics:**
- Run sequentially (one after another)
- Must complete successfully before main containers start
- Cannot have readiness probes (they run to completion)
- Share volumes with main containers

**Use cases:**
- Wait for dependencies (database, external service)
- Database migrations
- Download configuration
- Set up permissions
- Clone git repos

### Q12: What is a sidecar container?

**Answer:**
A sidecar is a helper container that runs alongside the main application container in the same Pod.

**Common patterns:**
| Pattern | Description | Example |
|---------|-------------|---------|
| **Sidecar** | Extends main app functionality | Log shipper (Fluentd) |
| **Ambassador** | Proxies external communication | Database proxy |
| **Adapter** | Standardizes output | Metrics adapter |

---

## Services and Networking

### Q13: What is a Kubernetes Service?

**Answer:**
A Service provides a stable network endpoint for a set of pods. Since pods are ephemeral and get new IPs when restarted, Services solve:
1. **Service discovery** - Stable DNS name
2. **Load balancing** - Distribute traffic across pod replicas
3. **Stable endpoint** - Persistent IP address

### Q14: Explain the different Service types.

**Answer:**

| Type | Description | Use Case |
|------|-------------|----------|
| **ClusterIP** | Internal virtual IP (default) | Internal microservice communication |
| **NodePort** | Exposes on each node at port 30000-32767 | Development, testing |
| **LoadBalancer** | Creates external load balancer (cloud provider) | Production external access |
| **ExternalName** | Maps to external DNS name | External service access |

### Q15: How does kube-proxy work?

**Answer:**
kube-proxy maintains network rules on each node that enable Service functionality:

**iptables mode:**
- Creates iptables rules for each Service
- Random load balancing
- Good for small/medium clusters

**IPVS mode:**
- Uses Linux IP Virtual Server
- Better performance for large clusters
- Multiple load balancing algorithms (RR, LC, DH, SH)

When traffic hits a Service IP:
1. iptables/IPVS intercepts the traffic
2. Selects a backend pod
3. DNATs (rewrites) destination IP to pod IP
4. Pod receives traffic

### Q16: What is an Ingress? How does it differ from a Service?

**Answer:**
- **Service:** L4 (TCP/UDP) load balancing, stable endpoint for pods
- **Ingress:** L7 (HTTP/HTTPS) routing, path-based and host-based routing

**Ingress requires an Ingress Controller** (nginx, traefik, etc.) to function.

```
Internet → Ingress Controller → Ingress Rules → Services → Pods
```

### Q17: What is a Headless Service?

**Answer:**
A Service with `clusterIP: None`. Instead of load balancing, DNS returns all pod IPs directly.

**Use cases:**
- StatefulSets (direct pod access needed)
- Custom service discovery
- When clients need to connect to specific pods

### Q18: What are NetworkPolicies?

**Answer:**
NetworkPolicies control traffic flow between pods (firewall rules at pod level).

**Default behavior:** All pods can communicate with all pods.

Once a NetworkPolicy selects a pod, ONLY matching traffic is allowed; everything else is denied.

---

## Storage

### Q19: Explain the relationship between PV, PVC, and StorageClass.

**Answer:**

**PersistentVolume (PV):**
- Cluster-wide storage resource
- Provisioned by administrator (static) or dynamically
- Independent of any pod

**PersistentVolumeClaim (PVC):**
- User's request for storage
- Binds to a matching PV
- Used by pods

**StorageClass:**
- Defines "classes" of storage
- Enables dynamic provisioning
- Specifies provisioner, parameters, reclaim policy

**Flow:**
1. User creates PVC with a StorageClass
2. StorageClass provisioner creates storage
3. PV is auto-created and bound to PVC
4. Pod uses PVC

### Q20: What are the access modes for PersistentVolumes?

**Answer:**

| Mode | Abbreviation | Description |
|------|-------------|-------------|
| **ReadWriteOnce** | RWO | Mounted as read-write by a single node |
| **ReadOnlyMany** | ROX | Mounted as read-only by many nodes |
| **ReadWriteMany** | RWX | Mounted as read-write by many nodes |
| **ReadWriteOncePod** | RWOP | Mounted as read-write by a single pod (K8s 1.22+) |

### Q21: What happens to data when a Pod is deleted?

**Answer:**
Depends on the volume type:

| Volume Type | Data After Pod Deletion |
|------------|------------------------|
| emptyDir | Lost |
| hostPath | Persists on node |
| PVC (Retain policy) | Persists, PV becomes Released |
| PVC (Delete policy) | Deleted along with storage |

---

## Configuration and Secrets

### Q22: Difference between ConfigMap and Secret?

**Answer:**

| Feature | ConfigMap | Secret |
|---------|-----------|--------|
| **Purpose** | Non-sensitive config | Sensitive data |
| **Encoding** | Plain text | Base64 encoded |
| **Encryption** | No | Optional (enable encryption at rest) |
| **Size limit** | 1MB | 1MB |
| **Volume mode** | 0644 | 0600 (owner read-only) |

### Q23: How to use ConfigMaps in pods?

**Answer:**
1. **Environment variables:** `valueFrom.configMapKeyRef` or `envFrom.configMapRef`
2. **Volume mounts:** Mount as files in a directory
3. **Single file mount:** Use `subPath` to mount a specific key as a file

**Important:** Env vars are set at container start and don't update. Volume mounts update automatically (within ~1 min).

### Q24: Are Kubernetes secrets encrypted?

**Answer:**
**No** - by default, secrets are only base64 encoded, not encrypted.

To encrypt:
1. Enable encryption at rest in etcd (EncryptionConfiguration)
2. Use external secret managers (HashiCorp Vault, AWS Secrets Manager)
3. Use Sealed Secrets or SOPS for Git storage

---

## Scheduling

### Q25: What are Taints and Tolerations?

**Answer:**

**Taints** (on nodes) - Repel pods:
```bash
kubectl taint nodes node1 key=value:NoSchedule
```

**Tolerations** (on pods) - Allow scheduling on tainted nodes:
```yaml
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
```

**Effects:**
| Effect | Behavior |
|--------|----------|
| NoSchedule | Won't schedule new pods |
| PreferNoSchedule | Prefer not to, but allow |
| NoExecute | Evict existing pods too |

### Q26: Difference between nodeSelector and nodeAffinity?

**Answer:**

| Feature | nodeSelector | nodeAffinity |
|---------|-------------|--------------|
| **Complexity** | Simple key-value | Advanced expressions |
| **Operators** | Only equality | In, NotIn, Exists, Gt, Lt |
| **Soft constraints** | No | Yes (preferredDuringScheduling) |
| **Multiple terms** | No | Yes (OR between terms) |

### Q27: What is Pod Anti-Affinity?

**Answer:**
Pod Anti-Affinity prevents pods from being scheduled together.

**Use case:** Spread replicas across nodes/zones for high availability:

```yaml
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchExpressions:
      - key: app
        operator: In
        values: [my-app]
    topologyKey: kubernetes.io/hostname
```

---

## Security and RBAC

### Q28: What is RBAC in Kubernetes?

**Answer:**
Role-Based Access Control manages who can do what in the cluster.

**Objects:**
- **Role/ClusterRole** - Define permissions
- **RoleBinding/ClusterRoleBinding** - Bind roles to users/groups/service accounts

**Role** = namespace-scoped
**ClusterRole** = cluster-wide

### Q29: What is a ServiceAccount?

**Answer:**
A ServiceAccount provides an identity for processes running in a Pod. It's how pods authenticate to the Kubernetes API.

- Every namespace has a `default` service account
- Tokens are mounted at `/var/run/secrets/kubernetes.io/serviceaccount/`
- Use dedicated SAs per application (not default)

### Q30: How to secure a Kubernetes cluster?

**Answer:**
1. Enable RBAC - never disable it
2. Use least privilege for all permissions
3. Enable audit logging
4. Encrypt secrets at rest
5. Use NetworkPolicies (default deny)
6. Enable Pod Security Standards (restricted)
7. Use read-only root filesystem
8. Run containers as non-root
9. Drop all capabilities, add only needed
10. Keep Kubernetes updated
11. Use image scanning
12. Set resource limits on all pods
13. Use dedicated service accounts
14. Disable automountServiceAccountToken unless needed

---

## Scaling

### Q31: Explain HPA, VPA, and Cluster Autoscaler.

**Answer:**

| Tool | Scales | Based On |
|------|--------|----------|
| **HPA** | Number of pods | CPU, memory, custom metrics |
| **VPA** | Pod resource requests/limits | Historical usage |
| **Cluster Autoscaler** | Number of nodes | Pending pods |

**HPA formula:**
```
desiredReplicas = ceil[currentReplicas × (currentMetric / desiredMetric)]
```

### Q32: What is a PodDisruptionBudget?

**Answer:**
PDB ensures minimum availability during voluntary disruptions (node drains, upgrades):

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2
  # OR
  maxUnavailable: 1
  selector:
    matchLabels:
      app: my-app
```

Does NOT protect against involuntary disruptions (node crashes).

---

## Helm

### Q33: What is Helm?

**Answer:**
Helm is the package manager for Kubernetes.

**Key concepts:**
- **Chart** - Package of K8s resources (templates + values)
- **Release** - Running instance of a chart
- **Repository** - Location to store and share charts
- **Values** - Configuration parameters

### Q34: What changed from Helm 2 to Helm 3?

**Answer:**
1. **No Tiller** - Removed server-side component (security improvement)
2. **Release storage** - Now Secrets (not ConfigMaps in kube-system)
3. **Better security** - No cluster-wide admin needed
4. **CRD support** - CRDs in dedicated directory
5. **JSON patch** - Uses JSON merge patch

### Q35: How to rollback a Helm release?

**Answer:**
```bash
# View history
helm history <release>

# Rollback to previous
helm rollback <release>

# Rollback to specific revision
helm rollback <release> 2
```

Rollback creates a new revision with the previous manifest and applies it.

---

## Troubleshooting

### Q36: A pod is in CrashLoopBackOff. How do you troubleshoot?

**Answer:**
1. `kubectl describe pod <name>` - Check events and state
2. `kubectl logs <name>` - Check application logs
3. `kubectl logs <name> --previous` - Check logs from crashed container
4. Check for OOMKilled in `lastState`
5. Check resource limits
6. Check liveness probe configuration
7. Check environment variables and ConfigMaps/Secrets
8. Try running the container command manually

### Q37: How to debug a networking issue?

**Answer:**
1. Check Service has endpoints: `kubectl get endpoints <svc>`
2. Verify pod labels match service selector
3. Test pod-to-pod: `kubectl exec <pod1> -- curl <pod2-ip>`
4. Check NetworkPolicies
5. Test DNS: `kubectl exec <pod> -- nslookup <service>`
6. Check CoreDNS: `kubectl get pods -n kube-system -l k8s-app=kube-dns`
7. Check kube-proxy logs

### Q38: How to handle a node failure?

**Answer:**
1. Check: `kubectl get nodes`
2. Describe: `kubectl describe node <name>`
3. If permanently failed:
   ```bash
   kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
   kubectl delete node <node>
   ```
4. New node auto-registers
5. Pods rescheduled automatically

---

## Advanced Scenarios

### Q39: How does Kubernetes ensure high availability?

**Answer:**
**Control Plane:**
- Multiple API servers behind a load balancer
- etcd cluster with odd members (Raft consensus)
- Scheduler/controller-manager use leader election

**Worker Nodes:**
- Multiple worker nodes across availability zones
- Pod anti-affinity to spread replicas
- Cluster Autoscaler for automatic scaling

**Applications:**
- Multiple replicas across zones
- PodDisruptionBudgets for voluntary disruptions
- Rolling updates for zero-downtime deployments

### Q40: Explain the difference between declarative and imperative management.

**Answer:**

| Aspect | Declarative | Imperative |
|--------|------------|------------|
| **Approach** | Define desired state in YAML | Run commands |
| **Command** | `kubectl apply -f file.yaml` | `kubectl create/deploy/run` |
| **Idempotent** | Yes | No |
| **Version control** | Easy (YAML files) | Hard |
| **GitOps** | Compatible | Not compatible |
| **Best for** | Production, CI/CD | Quick testing |

### Q41: What is GitOps?

**Answer:**
GitOps is an operational framework that uses Git as the single source of truth for infrastructure and applications.

**Principles:**
1. Declarative - Everything described declaratively
2. Versioned - Desired state stored in Git
3. Automated - Software applies changes automatically
4. Self-healing - System continuously reconciles to desired state

**Tools:** ArgoCD, Flux

### Q42: What are Admission Controllers?

**Answer:**
Admission controllers are plugins that intercept requests to the Kubernetes API server before an object is persisted.

**Types:**
- **Validating** - Reject requests that don't meet criteria
- **Mutating** - Modify requests before storing
- **Both** - Do both

**Examples:**
- `PodSecurity` - Enforce pod security standards
- `LimitRanger` - Apply default resource limits
- `ResourceQuota` - Enforce namespace quotas
- `MutatingAdmissionWebhook` - Custom mutations
- `ValidatingAdmissionWebhook` - Custom validation

### Q43: What is the difference between a DaemonSet and a Deployment with node affinity?

**Answer:**

| DaemonSet | Deployment + Node Affinity |
|-----------|---------------------------|
| Automatically runs on ALL nodes | Must specify which nodes |
| Handles new nodes automatically | Need to update for new nodes |
| One pod per node | Can have multiple pods per node |
| Used for infrastructure | Used for applications |

### Q44: How to perform zero-downtime deployments?

**Answer:**
1. Use RollingUpdate strategy with `maxUnavailable: 0`
2. Configure proper readiness probes
3. Use `minReadySeconds` to ensure pods are truly ready
4. Add `terminationGracePeriodSeconds` for graceful shutdown
5. Use PodDisruptionBudgets
6. Use preStop lifecycle hook for graceful draining

```yaml
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  minReadySeconds: 30
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
```

# Scheduling: Taints, Tolerations, Affinity

## Kubernetes Scheduling Process

```
Pod created (pending)
      │
      ▼
┌─────────────────────────┐
│   kube-scheduler         │
│                         │
│  1. Filter (Predicate)  │  ← Which nodes CAN run this pod?
│  2. Score (Priority)    │  ← Which node is BEST?
│  3. Bind                │  ← Assign pod to node
└─────────────────────────┘
      │
      ▼
Pod scheduled → kubelet starts containers
```

## Taints and Tolerations

### Taints (on Nodes)

Repel pods that don't tolerate the taint.

```bash
# Add a taint
kubectl taint nodes node1 key=value:NoSchedule
kubectl taint nodes node1 dedicated=gpu:NoSchedule

# Remove a taint
kubectl taint nodes node1 key=value:NoSchedule-
kubectl taint nodes node1 dedicated-

# View taints
kubectl describe node node1 | grep Taints
```

### Taint Effects

| Effect | Description |
|--------|-------------|
| `NoSchedule` | Pods without matching toleration won't be scheduled |
| `PreferNoSchedule` | Prefer not to schedule, but allow if no other option |
| `NoExecute` | Evicts running pods without matching toleration |

### Tolerations (on Pods)

Allow pods to be scheduled on tainted nodes.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  tolerations:
  # Exact match
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  
  # Exists (match any value for this key)
  - key: "dedicated"
    operator: "Exists"
    effect: "NoSchedule"
  
  # Exists with NoExecute (eviction with grace period)
  - key: "dedicated"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 3600  # Evict after 1 hour
```

### Real-World Example: Dedicated Nodes

```bash
# Taint GPU nodes
kubectl taint nodes gpu-node-1 hardware=gpu:NoSchedule
kubectl taint nodes gpu-node-2 hardware=gpu:NoSchedule
```

```yaml
# GPU workload can only run on GPU nodes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-training
spec:
  replicas: 2
  template:
    spec:
      tolerations:
      - key: "hardware"
        operator: "Equal"
        value: "gpu"
        effect: "NoSchedule"
      nodeSelector:
        hardware: gpu
      containers:
      - name: training
        image: tensorflow:latest
        resources:
          limits:
            nvidia.com/gpu: 1
```

### Master Node Taint

```bash
# By default, control plane nodes have:
node-role.kubernetes.io/control-plane:NoSchedule

# To schedule pods on master:
kubectl taint nodes master node-role.kubernetes.io/control-plane:NoSchedule-
```

## Node Selector

Simple node selection based on labels.

```bash
# Label a node
kubectl label nodes node1 disktype=ssd

# Verify
kubectl get nodes --show-labels | grep node1
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
spec:
  nodeSelector:
    disktype: ssd
  containers:
  - name: app
    image: myapp:1.0
```

**Limitation:** Only supports exact label matching, no complex expressions.

## Node Affinity

Advanced node selection with expressions.

### requiredDuringSchedulingIgnoredDuringExecution

Hard requirement - pod will NOT schedule without it.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: affinity-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/arch
            operator: In
            values: [amd64, arm64]
          - key: disktype
            operator: In
            values: [ssd, nvme]
        - matchExpressions:
          - key: zone
            operator: In
            values: [us-east-1a, us-east-1b]
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: instance-type
            operator: In
            values: [m5.xlarge, m5.2xlarge]
  containers:
  - name: app
    image: myapp:1.0
```

### Operators

| Operator | Description |
|----------|-------------|
| `In` | Label value is in the list |
| `NotIn` | Label value is NOT in the list |
| `Exists` | Label exists (any value) |
| `DoesNotExist` | Label does not exist |
| `Gt` | Greater than |
| `Lt` | Less than |

### Preferred vs Required

```yaml
# HARD requirement (must match)
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
  - matchExpressions:
    - key: disktype
      operator: In
      values: [ssd]

# SOFT preference (nice to have)
preferredDuringSchedulingIgnoredDuringExecution:
- weight: 80
  preference:
    matchExpressions:
    - key: zone
      operator: In
      values: [us-east-1a]
- weight: 20
  preference:
    matchExpressions:
    - key: zone
      operator: In
      values: [us-east-1b]
```

**Weight:** 1-100, higher weight = more preferred

## Pod Affinity and Anti-Affinity

Schedule pods based on other pods' labels.

### Pod Anti-Affinity (Spread Pods)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values: [web-app]
            topologyKey: kubernetes.io/hostname
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: [web-app]
              topologyKey: topology.kubernetes.io/zone
      containers:
      - name: web
        image: nginx:1.25
```

**Result:** Each replica runs on a different node, preferably in different zones.

### Pod Affinity (Co-locate Pods)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cache-app
spec:
  replicas: 2
  template:
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values: [api-app]
            topologyKey: kubernetes.io/hostname
      containers:
      - name: cache
        image: redis:7
```

**Result:** Cache pods run on same nodes as API pods (reduced latency).

### Topology Keys

| Key | Description |
|-----|-------------|
| `kubernetes.io/hostname` | Single node |
| `topology.kubernetes.io/zone` | Availability zone |
| `topology.kubernetes.io/region` | Region |
| `kubernetes.io/os` | OS (linux/windows) |
| Custom labels | Any node label |

### Common Patterns

#### Spread Across Nodes (High Availability)

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

#### Spread Across Zones (Disaster Recovery)

```yaml
podAntiAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100
    podAffinityTerm:
      labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values: [my-app]
      topologyKey: topology.kubernetes.io/zone
```

#### Co-locate with Related Service

```yaml
podAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchExpressions:
      - key: app
        operator: In
        values: [database]
    topologyKey: kubernetes.io/hostname
```

## Complete Scheduling Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-app
spec:
  replicas: 6
  selector:
    matchLabels:
      app: production-app
  template:
    metadata:
      labels:
        app: production-app
    spec:
      # Node selector (simple)
      nodeSelector:
        kubernetes.io/os: linux
      
      # Tolerations (allow on tainted nodes)
      tolerations:
      - key: "dedicated"
        operator: "Equal"
        value: "production"
        effect: "NoSchedule"
      
      # Affinity rules
      affinity:
        # Node affinity (where to run)
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: instance-type
                operator: In
                values: [m5.xlarge, m5.2xlarge]
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: spot
                operator: NotIn
                values: ["true"]
        
        # Pod anti-affinity (spread pods)
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values: [production-app]
            topologyKey: kubernetes.io/hostname
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 50
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                values: [production-app]
              topologyKey: topology.kubernetes.io/zone
      
      containers:
      - name: app
        image: myapp:2.0
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
```

## Scheduling Decision Flow

```
Pod arrives
      │
      ▼
┌─────────────────────────────────────┐
│  Phase 1: Filtering (Predicates)    │
│                                     │
│  □ Node has enough resources?       │
│  □ Node selector matches?           │
│  □ Node affinity satisfied?         │
│  □ Taints tolerated?                │
│  □ Pod anti-affinity satisfied?     │
│  □ Port conflicts?                  │
│  □ Volume constraints met?          │
│                                     │
│  ↓ Pass all? → Candidate nodes      │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  Phase 2: Scoring (Priorities)      │
│                                     │
│  □ Resource balance                 │
│  □ Node affinity preferences        │
│  □ Pod affinity preferences         │
│  □ Image locality                   │
│  □ Taint tolerance                  │
│  □ Zone spreading                   │
│                                     │
│  ↓ Highest score → Selected node    │
└─────────────────┬───────────────────┘
                  │
                  ▼
           Binding
```

## Priority and Preemption

```yaml
# PriorityClass
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000              # Higher = more important
globalDefault: false        # Default for all pods
preemptionPolicy: PreemptLowerPriority  # or Never
description: "High priority for critical workloads"
---
# Use in Pod
apiVersion: v1
kind: Pod
metadata:
  name: critical-pod
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: myapp:1.0
```

**Built-in Priority Classes:**
- `system-cluster-critical` (2000000000)
- `system-node-critical` (2000001000)

## Interview Questions and Answers

### Q: Difference between nodeSelector and nodeAffinity?

- **nodeSelector** - Simple key-value matching, only `In` operator
- **nodeAffinity** - Advanced, supports multiple operators, soft/hard constraints, weights

### Q: What is the difference between taints and node affinity?

- **Taints** - Push mechanism (node repels pods)
- **Node affinity** - Pull mechanism (pod prefers/attracts to nodes)

Used together, they create dedicated nodes.

### Q: What happens when a NoExecute taint is added to a node?

Pods without a matching toleration are immediately evicted. If `tolerationSeconds` is specified, pods are evicted after that duration.

### Q: How does pod anti-affinity differ from pod topology spread constraints?

- **Pod anti-affinity** - Hard/soft constraint to avoid co-location
- **Topology spread constraints** - Evenly distribute pods across topology domains (K8s 1.19+)

### Q: What is the scheduling order of constraints?

1. Taints/Tolerations (filter out)
2. NodeSelector (filter out)
3. Node Affinity required (filter out)
4. Pod Affinity required (filter out)
5. Pod Anti-Affinity required (filter out)
6. Scoring (ranking)

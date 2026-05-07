# Autoscaling and Advanced Workloads

## Horizontal Pod Autoscaler (HPA)

Automatically scales the number of pods based on metrics.

### HPA v2 (Metrics-Based)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  
  # Scaling behavior
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0      # Scale immediately
      policies:
      - type: Pods
        value: 2                          # Add max 2 pods per period
        periodSeconds: 60                 # Evaluation window
      - type: Percent
        value: 50                         # Max 50% increase
        periodSeconds: 60
      selectPolicy: Max                   # Use the policy that adds most pods
    
    scaleDown:
      stabilizationWindowSeconds: 300     # Wait 5 min before scaling down
      policies:
      - type: Pods
        value: 1
        periodSeconds: 180
      selectPolicy: Min                   # Use the policy that removes least pods

  # Metrics
  metrics:
  # CPU-based
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  
  # Memory-based
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 512Mi
  
  # Custom metrics (from Prometheus)
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  
  # Object metrics (from external service)
  - type: Object
    object:
      metric:
        name: queue_depth
      describedObject:
        apiVersion: v1
        kind: Service
        name: worker-service
      target:
        type: Value
        value: "100"
  
  # External metrics
  - type: External
    external:
      metric:
        name: kafka_consumer_lag
        selector:
          matchLabels:
            topic: orders
      target:
        type: AverageValue
        averageValue: "500"
```

### HPA Commands

```bash
# Create HPA (imperative)
kubectl autoscale deployment web-app \
  --min=2 --max=10 --cpu-percent=70

# Check HPA status
kubectl get hpa
kubectl get hpa web-app-hpa -o yaml

# View HPA events
kubectl describe hpa web-app-hpa

# Check current metrics
kubectl top pods
kubectl top nodes
```

### HPA Math

```
desiredReplicas = ceil[currentReplicas * (currentMetricValue / desiredMetricValue)]

# Example:
# Current: 3 pods
# Current CPU: 80%
# Target CPU: 60%
# desiredReplicas = ceil[3 * (80/60)] = ceil[4.0] = 4 pods
```

### Metrics Server

Required for resource-based HPA:

```bash
# Install metrics server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Check if working
kubectl top pods
kubectl top nodes
```

## Vertical Pod Autoscaler (VPA)

Automatically adjusts CPU/memory requests and limits.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  
  updatePolicy:
    updateMode: Auto        # Auto, Recreate, Initial, Off
  
  resourcePolicy:
    containerPolicies:
    - containerName: web-app
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: "2"
        memory: 2Gi
      controlledResources: ["cpu", "memory"]
```

### VPA Update Modes

| Mode | Description |
|------|-------------|
| `Off` | Recommendations only, no auto-update |
| `Initial` | Set resources only on pod creation |
| `Recreate` | Update resources, may recreate pods |
| `Auto` | Update in-place when possible, recreate otherwise |

**Note:** Don't use HPA and VPA together on the same resource (CPU/memory).

## Cluster Autoscaler

Scales the number of nodes in the cluster.

```yaml
# Usually managed by cloud provider
# AWS EKS - Cluster Autoscaler
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
  name: cluster-autoscaler
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-autoscaler
rules:
- apiGroups: [""]
  resources: ["events", "endpoints"]
  verbs: ["create", "patch"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/eviction"]
  verbs: ["create"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch", "update"]
- apiGroups: [""]
  resources: ["nodes/status"]
  verbs: ["patch", "update"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.28.0
        name: cluster-autoscaler
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --balance-similar-node-groups
        - --scale-down-utilization-threshold=0.5
        - --scale-down-delay-after-add=10m
        - --scale-down-unneeded-time=10m
```

### Scaling Hierarchy

```
┌──────────────────────────────┐
│    1. HPA                     │
│    (Scale pods)               │
│    If pods can't be scheduled │
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│    2. Cluster Autoscaler      │
│    (Scale nodes)              │
│    Creates new nodes          │
└──────────────────────────────┘
```

## CronJob Advanced Patterns

### Multiple Schedules

```yaml
# Run every weekday at 9 AM
apiVersion: batch/v1
kind: CronJob
metadata:
  name: weekday-report
spec:
  schedule: "0 9 * * 1-5"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: report
            image: myapp:report
            command: ["./generate-report.sh"]
```

### Concurrent Policy

```yaml
spec:
  concurrencyPolicy: Forbid  # Skip if previous still running
  # Allow      # Run multiple concurrently
  # Replace    # Cancel previous, run new one
```

### Timezone-Aware (K8s 1.27+)

```yaml
spec:
  schedule: "0 9 * * *"
  timeZone: "America/New_York"
```

## Pod Disruption Budget (PDB)

Ensures minimum availability during voluntary disruptions.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
spec:
  minAvailable: 2            # Minimum number of pods that must be available
  # maxUnavailable: 1        # OR: Maximum pods that can be unavailable
  selector:
    matchLabels:
      app: web-app
```

### PDB During Node Drain

```bash
# When draining a node:
kubectl drain node1 --ignore-daemonsets

# PDB blocks if it would violate minAvailable:
# If you have 3 pods with minAvailable=2:
# - Draining node with 1 pod: ALLOWED (2 pods remain)
# - Draining node with 2 pods: BLOCKED (only 1 would remain)
```

### PDB vs HPA

| Feature | PDB | HPA |
|---------|-----|-----|
| Purpose | Prevent too many disruptions | Scale based on demand |
| Trigger | Voluntary (drain, upgrade) | Metrics (CPU, memory, custom) |
| Action | Block disruption | Add/remove pods |

## StatefulSet Advanced

### Ordered Ready Deployment

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
spec:
  serviceName: kafka-headless
  replicas: 3
  podManagementPolicy: OrderedReady  # OrderedReady or Parallel
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0  # Only update pods with ordinal >= partition
  template:
    metadata:
      labels:
        app: kafka
    spec:
      containers:
      - name: kafka
        image: confluentinc/cp-kafka:7.5.0
        ports:
        - containerPort: 9092
        env:
        - name: KAFKA_BROKER_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.labels['controller-revision-hash']
        - name: KAFKA_ADVERTISED_LISTENERS
          value: "PLAINTEXT://$(POD_NAME).kafka-headless.default.svc.cluster.local:9092"
        volumeMounts:
        - name: kafka-data
          mountPath: /var/lib/kafka/data
  volumeClaimTemplates:
  - metadata:
      name: kafka-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Gi
```

### StatefulSet Identity

```
Pod: kafka-0
  ├── Stable hostname: kafka-0.kafka-headless.default.svc.cluster.local
  ├── Stable storage: PVC kafka-data-kafka-0
  └── Stable ordinal: 0

Pod: kafka-1
  ├── Stable hostname: kafka-1.kafka-headless.default.svc.cluster.local
  ├── Stable storage: PVC kafka-data-kafka-1
  └── Stable ordinal: 1
```

## Resource Quotas and Limit Ranges

### ResourceQuota

Limit total resource consumption in a namespace.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # Compute resources
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    
    # Storage
    requests.storage: 500Gi
    persistentvolumeclaims: "50"
    
    # Object count
    pods: "100"
    services: "50"
    secrets: "100"
    configmaps: "100"
    replicationcontrollers: "50"
    resourcequotas: "10"
    
    # Services
    services.loadbalancers: "5"
    services.nodeports: "10"
    
    # Scopes
    scopes:
    - BestEffort
    - NotBestEffort
    - PriorityClass
```

### LimitRange

Set default requests/limits for containers.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
  # Default limits for containers
  - type: Container
    default:
      cpu: "500m"
      memory: 512Mi
    defaultRequest:
      cpu: "250m"
      memory: 256Mi
    max:
      cpu: "2"
      memory: 4Gi
    min:
      cpu: "50m"
      memory: 64Mi
  
  # Pod-level limits
  - type: Pod
    max:
      cpu: "4"
      memory: 8Gi
    min:
      cpu: "100m"
      memory: 128Mi
  
  # PVC limits
  - type: PersistentVolumeClaim
    max:
      storage: 100Gi
    min:
      storage: 1Gi
```

## Custom Resource Definitions (CRDs)

Extend Kubernetes with custom resources.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              engine:
                type: string
                enum: [mysql, postgresql, mongodb]
              version:
                type: string
              storage:
                type: string
              replicas:
                type: integer
                minimum: 1
                maximum: 10
          status:
            type: object
            properties:
              phase:
                type: string
              endpoint:
                type: string
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
    shortNames:
    - db
```

### Using CRD

```yaml
apiVersion: example.com/v1
kind: Database
metadata:
  name: my-database
spec:
  engine: postgresql
  version: "15"
  storage: 100Gi
  replicas: 3
```

## Operators

Controllers that manage custom resources.

### Operator Pattern

```
┌──────────────────────────────────────┐
│            Operator                   │
│  (Custom Controller)                  │
│                                       │
│  ┌─────────────────────────────────┐ │
│  │         Reconciliation Loop     │ │
│  │                                 │ │
│  │  1. Watch for CR changes        │ │
│  │  2. Read desired state          │ │
│  │  3. Take actions:               │ │
│  │     - Create Deployments        │ │
│  │     - Create Services           │ │
│  │     - Backup data               │ │
│  │     - Upgrade versions          │ │
│  │  4. Update CR status            │ │
│  └─────────────────────────────────┘ │
└──────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────┐
│        Custom Resource               │
│  spec:                                │
│    engine: postgresql                 │
│    version: "15"                      │
│    replicas: 3                        │
│  status:                              │
│    phase: Running                     │
│    endpoint: postgres.default.svc     │
└──────────────────────────────────────┘
```

Popular Operators:
- **Prometheus Operator** - Monitoring
- **Cert-Manager** - Certificate management
- **Rook** - Storage orchestration
- **Strimzi** - Apache Kafka
- **Elastic Cloud on K8s** - Elasticsearch/Kibana
- **Vault Operator** - HashiCorp Vault

## Interview Questions and Answers

### Q: How does HPA calculate the number of replicas?

```
desiredReplicas = ceil[currentReplicas * (currentMetricValue / desiredMetricValue)]
```

The scheduler takes the highest result across all metrics. It respects minReplicas and maxReplicas boundaries.

### Q: HPA vs VPA vs Cluster Autoscaler?

| Tool | What it scales | Based on |
|------|---------------|----------|
| HPA | Number of pods | CPU, memory, custom metrics |
| VPA | Pod resource requests/limits | Historical usage patterns |
| Cluster Autoscaler | Number of nodes | Pending pods that can't be scheduled |

### Q: Can you use HPA and VPA together?

Not on the same metric. HPA scales pods based on CPU/memory utilization, while VPA adjusts CPU/memory requests. If both manage CPU, they conflict. Use HPA for horizontal scaling and VPA in "Off" or "Initial" mode for right-sizing.

### Q: What is a Pod Disruption Budget?

A PDB ensures that a minimum number of pods remain available during voluntary disruptions like node drains, cluster upgrades, or cluster autoscaler scale-downs. It doesn't protect against node failures (involuntary disruptions).

### Q: How does the Cluster Autoscaler work?

1. Detects pods that are in Pending state due to insufficient resources
2. Checks if any node group can accommodate the pending pods
3. Increases the node group size
4. New node registers with the cluster
5. Scheduler places the pending pods on the new node
6. Scale-down: Removes underutilized nodes after a delay

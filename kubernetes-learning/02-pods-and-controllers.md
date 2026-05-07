# Pods and Controllers

## Pods

### What is a Pod?

The smallest deployable unit in Kubernetes. A pod is a logical host for one or more containers that share:
- **Network namespace** - Same IP address, port space, localhost
- **Storage volumes** - Shared volume mounts
- **Lifecycle** - Created, scheduled, and terminated together
- **Resource limits** - CPU/memory defined at pod level

### Pod Architecture

```
┌─────────────────────────────────────────┐
│                  Pod                     │
│  IP: 10.244.1.5                         │
│                                          │
│  ┌───────────────────────────────────┐  │
│  │         Pause Container           │  │
│  │  (sandbox container)              │  │
│  │  - Creates network namespace      │  │
│  │  - Holds the IP address           │  │
│  │  - Minimal (~700KB)              │  │
│  └───────────────────────────────────┘  │
│                                          │
│  ┌──────────────┐  ┌──────────────┐     │
│  │  App Cont.   │  │  Sidecar     │     │
│  │  nginx:1.25  │  │  log-agent   │     │
│  │  Port: 80    │  │  Port: 9090  │     │
│  └──────┬───────┘  └──────┬───────┘     │
│         │                 │             │
│  ┌──────▼─────────────────▼───────┐     │
│  │       Shared Volumes           │     │
│  │  /var/log  (logs)              │     │
│  │  /data     (app data)          │     │
│  └────────────────────────────────┘     │
└─────────────────────────────────────────┘
```

### Complete Pod Specification

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  namespace: default
  labels:
    app: myapp
    tier: frontend
  annotations:
    description: "My application pod"
spec:
  # Scheduling
  nodeName: ""                    # Force specific node
  schedulerName: default-scheduler
  
  # Restart policy
  restartPolicy: Always           # Always, OnFailure, Never
  
  # Image pull policy
  imagePullSecrets:
  - name: registry-secret
  
  # Service account
  serviceAccountName: default
  automountServiceAccountToken: true

  # Host settings
  hostNetwork: false
  hostPID: false
  hostIPC: false
  hostname: myapp
  subdomain: myapp-subdomain
  dnsPolicy: ClusterFirst         # ClusterFirst, Default, None, ClusterFirstWithHostNet

  # Security
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    runAsNonRoot: true

  # Node selection
  nodeSelector:
    disktype: ssd
  
  tolerations:
  - key: "example-key"
    operator: "Exists"
    effect: "NoSchedule"

  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/arch
            operator: In
            values: [amd64, arm64]

  # Init containers
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting; sleep 2; done']
    resources:
      limits:
        cpu: "200m"
        memory: "128Mi"

  # Containers
  containers:
  - name: app
    image: myapp:1.0
    imagePullPolicy: IfNotPresent   # Always, IfNotPresent, Never
    
    # Command override
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello; sleep 10; done"]
    
    # Working directory
    workingDir: /app
    
    # Ports
    ports:
    - name: http
      containerPort: 8080
      protocol: TCP
    - name: metrics
      containerPort: 9090
      protocol: TCP
    
    # Environment
    env:
    - name: APP_ENV
      value: "production"
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: db-host
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: db-secret
    
    # Resources
    resources:
      requests:
        cpu: "250m"
        memory: "256Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
    
    # Volume mounts
    volumeMounts:
    - name: app-data
      mountPath: /data
      readOnly: false
    - name: config-volume
      mountPath: /etc/config
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
    
    # Probes
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: Custom-Header
          value: Awesome
      initialDelaySeconds: 15
      periodSeconds: 10
      timeoutSeconds: 3
      successThreshold: 1
      failureThreshold: 3
    
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
    
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 10
      failureThreshold: 30
    
    # Lifecycle hooks
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo started > /tmp/startup.log"]
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 10"]
    
    # Security
    securityContext:
      runAsUser: 1000
      runAsGroup: 3000
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      capabilities:
        add: ["NET_ADMIN"]
        drop: ["ALL"]
    
    # Termination
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File    # File or FallbackToLogsOnError

  # Volumes
  volumes:
  - name: app-data
    persistentVolumeClaim:
      claimName: my-pvc
  - name: config-volume
    configMap:
      name: app-config
  - name: secret-volume
    secret:
      secretName: db-secret
  - name: empty-dir-volume
    emptyDir:
      sizeLimit: 500Mi
  - name: host-path-volume
    hostPath:
      path: /data
      type: Directory
```

### Pod Lifecycle

```
Pending → Running → Succeeded/Failed
  │         │
  │         └→ Terminated
  │
  └→ (stuck if scheduling fails)
```

**Phases:**
- **Pending:** Pod accepted, containers not yet created
- **Running:** At least one container is running
- **Succeeded:** All containers terminated successfully
- **Failed:** All containers terminated, at least one failed
- **Unknown:** State cannot be determined

### Pod Conditions

- `PodScheduled` - Pod scheduled to a node
- `ContainersReady` - All containers ready
- `Initialized` - Init containers completed
- `Ready` - Pod can serve requests

### Probes Deep Dive

| Probe Type | Purpose | Failure Action |
|------------|---------|----------------|
| **livenessProbe** | Is the container alive? | Restart container |
| **readinessProbe** | Is the container ready to serve? | Remove from Service endpoints |
| **startupProbe** | Has the app started? | No action until it succeeds, then disabled |

**Probe Methods:**

```yaml
# HTTP GET
httpGet:
  path: /healthz
  port: 8080
  scheme: HTTPS
  httpHeaders:
  - name: Authorization
    value: Bearer token

# TCP Socket
tcpSocket:
  port: 3306

# Exec Command
exec:
  command:
  - cat
  - /tmp/healthy
```

### Init Containers

Run BEFORE main containers. Must complete successfully.

```yaml
initContainers:
# 1. Wait for database
- name: wait-for-db
  image: busybox
  command: ['sh', '-c', 'until nc -z db-service 5432; do echo waiting; sleep 2; done']

# 2. Database migration
- name: db-migrate
  image: myapp:migrate
  command: ['npm', 'run', 'migrate']

# 3. Download config
- name: download-config
  image: busybox
  command: ['sh', '-c', 'wget https://config.example.com/app.conf -O /etc/config/app.conf']
  volumeMounts:
  - name: config-volume
    mountPath: /etc/config
```

### Static Pods

Managed directly by kubelet, not through the API server.

```bash
# Located on node at
/etc/kubernetes/manifests/kube-apiserver.yaml
/etc/kubernetes/manifests/kube-controller-manager.yaml
/etc/kubernetes/manifests/kube-scheduler.yaml
```

### Multi-Container Pod Patterns

| Pattern | Description | Example |
|---------|-------------|---------|
| **Sidecar** | Helper container extends main app | Log shipper, proxy |
| **Ambassador** | Proxy for external communication | Database proxy, API gateway |
| **Adapter** | Standardizes output format | Log normalizer, metrics adapter |

## ReplicaSet

Ensures a specified number of pod replicas are running.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
        ports:
        - containerPort: 8080
```

**Note:** ReplicaSets are rarely created directly. Use Deployments instead.

## Deployment

Manages ReplicaSets and provides declarative updates for Pods.

### Basic Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  minReadySeconds: 10
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 600
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.3
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "250m"
            memory: "256Mi"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Update Strategies

#### Rolling Update (Default)

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # Max pods above desired count (can be % or number)
    maxUnavailable: 0    # Max pods unavailable during update
```

**Rolling update process:**
```
Desired: 3 replicas
Current: v1 (3 pods)

Step 1: Create v2 pod (maxSurge=1)
  v1: 3 pods, v2: 1 pod  (total: 4)

Step 2: v2 pod ready, remove v1 pod
  v1: 2 pods, v2: 1 pod  (total: 3)

Step 3: Create v2 pod
  v1: 2 pods, v2: 2 pods  (total: 4)

Step 4: v2 pod ready, remove v1 pod
  v1: 1 pod, v2: 2 pods  (total: 3)

Step 5: Continue until all updated
  v1: 0 pods, v2: 3 pods  (total: 3)
```

#### Recreate

```yaml
strategy:
  type: Recreate
```
Kills all existing pods before creating new ones. Causes downtime.

### Deployment Commands

```bash
# Create
kubectl apply -f deployment.yaml

# Update image
kubectl set image deployment/nginx-deployment nginx=nginx:1.26.0

# Check rollout status
kubectl rollout status deployment/nginx-deployment

# View rollout history
kubectl rollout history deployment/nginx-deployment
kubectl rollout history deployment/nginx-deployment --revision=2

# Rollback
kubectl rollout undo deployment/nginx-deployment
kubectl rollout undo deployment/nginx-deployment --to-revision=2

# Pause and resume rollout
kubectl rollout pause deployment/nginx-deployment
kubectl set image deployment/nginx-deployment nginx=nginx:1.26.0
kubectl rollout resume deployment/nginx-deployment

# Scale
kubectl scale deployment/nginx-deployment --replicas=5

# Check details
kubectl describe deployment/nginx-deployment
```

## StatefulSet

For stateful applications requiring stable identity and storage.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  podManagementPolicy: OrderedReady    # OrderedReady or Parallel
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: mysql
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
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### StatefulSet Characteristics

| Feature | Description |
|---------|-------------|
| **Stable Pod Names** | `mysql-0`, `mysql-1`, `mysql-2` |
| **Stable Network IDs** | DNS: `mysql-0.mysql-headless.default.svc.cluster.local` |
| **Stable Storage** | Each pod gets its own PVC, persists across rescheduling |
| **Ordered Deployment** | Pods created 0→N, deleted N→0 |
| **Ordered Scaling** | Scale up creates next index, scale down removes highest |

### When to Use StatefulSet vs Deployment

| Use Case | Controller |
|----------|-----------|
| Stateless web apps | Deployment |
| Microservices | Deployment |
| Databases (MySQL, PostgreSQL) | StatefulSet |
| Kafka, Zookeeper | StatefulSet |
| Elasticsearch | StatefulSet |
| Redis Cluster | StatefulSet |
| Message queues (RabbitMQ) | StatefulSet |

## DaemonSet

Ensures all (or some) nodes run a copy of a Pod.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluentd:v1.16
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

### Common DaemonSet Use Cases

- Log collection (Fluentd, Logstash)
- Metrics collection (Prometheus Node Exporter)
- Monitoring agents (Datadog, New Relic)
- Storage daemons (Ceph, GlusterFS)
- Network plugins (Calico, Cilium)

## Job and CronJob

### Job

Runs a pod to completion.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  completions: 1          # Total successful completions needed
  parallelism: 1          # Max parallel pods
  backoffLimit: 3         # Retry attempts before marking failed
  activeDeadlineSeconds: 600  # Max runtime in seconds
  ttlSecondsAfterFinished: 3600  # Auto-delete after completion
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migrate
        image: myapp:migrate
        command: ["npm", "run", "migrate"]
        env:
        - name: DB_HOST
          value: "postgres.default.svc.cluster.local"
```

### CronJob

Runs jobs on a schedule.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"              # Every day at 2 AM
  timezone: "America/New_York"       # K8s 1.24+
  concurrencyPolicy: Forbid          # Allow, Forbid, Replace
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 200
  suspend: false
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: myapp:backup
            command: ["./backup.sh"]
```

### Cron Schedule Syntax

```
# ┌───────────── minute (0 - 59)
# │ ┌───────────── hour (0 - 23)
# │ │ ┌───────────── day of month (1 - 31)
# │ │ │ ┌───────────── month (1 - 12)
# │ │ │ │ ┌───────────── day of week (0 - 6, 0=Sunday)
# │ │ │ │ │
# * * * * *

# Examples
0 2 * * *      # Daily at 2 AM
*/15 * * * *   # Every 15 minutes
0 0 * * 0      # Every Sunday at midnight
0 0 1 * *      # First of every month at midnight
0 8-17 * * 1-5 # Mon-Fri, 8 AM to 5 PM
```

## Interview Questions and Answers

### Q: What happens when you run `kubectl apply -f pod.yaml`?

1. `kubectl` sends request to `kube-apiserver`
2. API server authenticates and authorizes the request
3. Admission controllers validate and modify the object
4. Object is stored in `etcd`
5. `kube-scheduler` sees unscheduled Pod, selects a node
6. API server updates Pod spec with nodeName
7. `kubelet` on selected node sees the Pod
8. `kubelet` instructs container runtime to pull image and start containers
9. `kubelet` reports Pod status back to API server
10. Status updated in etcd

### Q: What is the pause container?

Every Pod has a "pause" (sandbox) container that:
- Creates and holds the network namespace
- Provides the IP address for the Pod
- Shares namespaces with other containers in the Pod
- Very small (~700KB), does nothing but sleep

### Q: Difference between liveness, readiness, and startup probes?

- **Liveness:** Restart container if it fails. For deadlocks.
- **Readiness:** Remove from Service endpoints if it fails. For temporary unavailability.
- **Startup:** Delay liveness/readiness checks until app is ready. For slow-starting apps.

### Q: What is the difference between command and args?

- `command` overrides the Docker `ENTRYPOINT`
- `args` overrides the Docker `CMD`
- If you only specify `args`, it appends to the original ENTRYPOINT

### Q: How does a RollingUpdate work internally?

1. Deployment creates a new ReplicaSet with new pod template
2. Old ReplicaSet scales down, new one scales up
3. Controlled by `maxSurge` and `maxUnavailable`
4. Each new pod must pass readiness probe before next step
5. Old ReplicaSet kept for rollback (controlled by `revisionHistoryLimit`)

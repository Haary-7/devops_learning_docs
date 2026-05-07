# Troubleshooting and Debugging Kubernetes

## Troubleshooting Methodology

```
Pod is not working
      │
      ▼
1. What's the pod status?          → kubectl get pods
      │
      ▼
2. What's the detailed status?     → kubectl describe pod <name>
      │
      ▼
3. What are the logs?              → kubectl logs <name>
      │
      ▼
4. Can I exec into the pod?        → kubectl exec -it <name> -- sh
      │
      ▼
5. Check events                    → kubectl get events --sort-by='.lastTimestamp'
      │
      ▼
6. Check node status               → kubectl describe node <node>
```

## Pod Status Troubleshooting

### Pending

Pod cannot be scheduled.

```bash
kubectl describe pod <pod-name>
```

**Common Causes and Fixes:**

| Cause | How to Check | Fix |
|-------|-------------|-----|
| Insufficient resources | `kubectl describe node` | Scale up cluster, reduce requests |
| No matching node | Check node labels | Add labels or fix nodeSelector |
| Taints not tolerated | `kubectl describe node` | Add tolerations |
| PVC not bound | `kubectl get pvc` | Check StorageClass, provisioner |
| Affinity not satisfied | `kubectl describe pod` | Adjust affinity rules |

### ContainerCreating

Container is being pulled or volumes mounted.

```bash
kubectl describe pod <pod-name>
```

**Common Causes:**

| Cause | Fix |
|-------|-----|
| Image pull slow | Check network, use local registry |
| Image not found | Verify image name, tag, registry access |
| ImagePullSecret missing | Add imagePullSecrets to pod spec |
| Volume mount failed | Check volume configuration |
| CSI driver issue | Check CSI driver pods |

### CrashLoopBackOff

Container keeps crashing and restarting.

```bash
# Check logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous   # Previous crashed container

# Check events
kubectl describe pod <pod-name>

# Exec if brief window
kubectl exec -it <pod-name> -- /bin/sh

# Run command immediately
kubectl run debug --image=busybox --restart=Never --rm -it -- sh
```

**Common Causes:**

| Cause | Fix |
|-------|-----|
| Application error | Fix code, check logs |
| Missing environment variable | Add required env vars |
| Missing config/secret | Create ConfigMap/Secret |
| Wrong command/entrypoint | Check command and args |
| OOMKilled | Increase memory limits |
| Liveness probe failing | Fix probe or app |
| Port already in use | Check port conflicts |

### ImagePullBackOff / ErrImagePull

Cannot pull container image.

```bash
kubectl describe pod <pod-name>
```

**Common Causes:**

| Cause | Fix |
|-------|-----|
| Image doesn't exist | Verify image name and tag |
| Typo in image name | Check spelling |
| Private registry, no credentials | Add imagePullSecret |
| Network issues | Check DNS, connectivity |
| Tag not found | Verify tag exists in registry |

```bash
# Test pulling image
docker pull <image>:<tag>

# Create imagePullSecret
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=user \
  --docker-password=pass

# Add to pod spec
spec:
  imagePullSecrets:
  - name: regcred
```

### OOMKilled

Container exceeded memory limit.

```bash
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'
# Output: OOMKilled
```

**Fix:**
```yaml
resources:
  limits:
    memory: "512Mi"   # Increase this
  requests:
    memory: "256Mi"   # Increase this too
```

### Pod never becomes Ready

```bash
# Check readiness probe
kubectl describe pod <pod-name> | grep -A 10 Readiness

# Test the endpoint manually
kubectl exec -it <pod-name> -- curl http://localhost:<port>/ready

# Check if port is correct
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].ports[*].containerPort}'
```

## Service Troubleshooting

### Service not reachable

```bash
# 1. Check Service exists
kubectl get svc <service-name>

# 2. Check endpoints (pods backing the service)
kubectl get endpoints <service-name>

# 3. If no endpoints, check pod labels
kubectl get pods --show-labels

# 4. Verify selector matches pod labels
kubectl get svc <service-name> -o jsonpath='{.spec.selector}'

# 5. Check kube-proxy
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# 6. Test from another pod
kubectl run test-pod --image=busybox --rm -it -- wget -qO- http://<service-name>:<port>
```

### DNS Issues

```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Test DNS resolution from a pod
kubectl run dns-test --image=busybox --rm -it -- nslookup kubernetes.default

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Check CoreDNS config
kubectl get configmap coredns -n kube-system -o yaml

# Check pod DNS config
kubectl exec <pod> -- cat /etc/resolv.conf
```

## Node Troubleshooting

### Node NotReady

```bash
# Check node status
kubectl get nodes
kubectl describe node <node-name>

# Check node conditions
kubectl get node <node-name> -o jsonpath='{.status.conditions}'

# Check kubelet
systemctl status kubelet
journalctl -u kubelet

# Check container runtime
systemctl status containerd
journalctl -u containerd

# Check disk pressure
kubectl describe node <node-name> | grep -A 5 Conditions

# Check node resources
kubectl top node <node-name>
```

### Node Conditions

| Condition | Meaning |
|-----------|---------|
| `Ready=True` | Node is healthy |
| `Ready=False` | Node is unhealthy |
| `Ready=Unknown` | Node not communicating |
| `MemoryPressure` | Low memory |
| `DiskPressure` | Low disk space |
| `PIDPressure` | Too many processes |
| `NetworkUnavailable` | Network misconfigured |

### Eviction

```bash
# Check if pod was evicted
kubectl get pod <pod-name> -o jsonpath='{.status.phase}'
kubectl describe pod <pod-name> | grep -i evict

# Reasons:
# - Node disk pressure
# - Node memory pressure
# - Node PID pressure
```

## Network Troubleshooting

### Pod-to-Pod Communication

```bash
# Get pod IPs
kubectl get pods -o wide

# Test connectivity
kubectl exec <pod1> -- ping <pod2-ip>
kubectl exec <pod1> -- curl <pod2-ip>:<port>

# Check network policies
kubectl get networkpolicies

# Trace route
kubectl exec <pod> -- traceroute <destination>
```

### Pod-to-External

```bash
# Test external connectivity
kubectl exec <pod> -- curl https://www.google.com
kubectl exec <pod> -- nslookup google.com

# Check if egress is blocked
kubectl get networkpolicies
```

### Ingress Troubleshooting

```bash
# Check ingress
kubectl get ingress
kubectl describe ingress <name>

# Check ingress controller
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Test ingress rules
curl -H "Host: myapp.example.com" http://<ingress-ip>

# Check ingress controller service
kubectl get svc -n ingress-nginx
```

## Debug Container (K8s 1.24+)

Ephemeral debug containers for troubleshooting.

```bash
# Add debug container to running pod
kubectl debug -it <pod-name> --image=busybox --target=<container-name>

# Copy a pod and modify for debugging
kubectl debug <pod-name> -it --image=ubuntu --share-processes --copy-to=debug-pod

# Run node-level debugging
kubectl debug node/<node-name> -it --image=ubuntu
```

## kubectl Debugging Commands

```bash
# Get all info about a resource
kubectl get <resource> <name> -o yaml

# Describe with events
kubectl describe <resource> <name>

# Watch in real-time
kubectl get pods -w
kubectl get events -w

# Sort events by time
kubectl get events --sort-by='.lastTimestamp'

# Custom columns
kubectl get pods -o custom-columns="NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName"

# JSONPath queries
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[*].ready}'
kubectl get pod <name> -o jsonpath='{.spec.containers[*].image}'

# Server-side dry run
kubectl apply -f manifest.yaml --dry-run=server

# Client-side dry run
kubectl apply -f manifest.yaml --dry-run=client -o yaml
```

## Common Scenarios and Solutions

### Scenario 1: Deployment Stuck Rolling Update

```bash
# Check rollout status
kubectl rollout status deployment/<name>

# Check new pods
kubectl get pods -l app=<name>

# Check why new pods aren't ready
kubectl describe pod <new-pod-name>

# Fix and continue
kubectl rollout resume deployment/<name>

# Rollback
kubectl rollout undo deployment/<name>
```

### Scenario 2: All Pods on One Node

```bash
# Check pod distribution
kubectl get pods -o wide

# Add pod anti-affinity
# (edit deployment and add anti-affinity rules)

# Force reschedule
kubectl delete pod <pod-name>  # Scheduler will place on different node
```

### Scenario 3: PersistentVolume Stuck in Pending

```bash
# Check PVC status
kubectl get pvc
kubectl describe pvc <name>

# Check available PVs
kubectl get pv

# Check StorageClass
kubectl get storageclass
kubectl describe storageclass <name>

# Check provisioner pods
kubectl get pods -n kube-system -l app=<provisioner>
```

### Scenario 4: High Memory Usage

```bash
# Find memory-heavy pods
kubectl top pods --sort-by=memory

# Check pod memory limits
kubectl get pods -o custom-columns="NAME:.metadata.name,MEM-LIMIT:.spec.containers[*].resources.limits.memory"

# Check node memory
kubectl top nodes

# Evict if needed
kubectl delete pod <memory-heavy-pod>
```

### Scenario 5: Certificate Expired

```bash
# Check certificate expiry
kubectl get secret <tls-secret> -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates

# Renew with cert-manager
kubectl get certificaterequest
kubectl describe certificate <name>

# Manual renewal
certbot renew
kubectl create secret tls <name> --cert=cert.pem --key=key.pem --dry-run=client -o yaml | kubectl apply -f -
```

## Monitoring and Observability

### Metrics Server

```bash
# Install
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify
kubectl top pods
kubectl top nodes
```

### Prometheus Stack (Helm)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

### k9s (Terminal UI)

```bash
# Install
brew install k9s

# Run
k9s
```

## Emergency Procedures

### Access cluster without kubectl

```bash
# Direct API access
curl -k https://<api-server>:6443/api/v1/pods \
  -H "Authorization: Bearer <token>"

# Use kubectl proxy
kubectl proxy
curl http://localhost:8001/api/v1/namespaces/default/pods
```

### etcd Backup and Restore

```bash
# Backup
ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Restore
ETCDCTL_API=3 etcdctl snapshot restore /tmp/etcd-backup.db \
  --data-dir=/var/lib/etcd-new
```

## Interview Questions and Answers

### Q: A pod is in CrashLoopBackOff. How do you troubleshoot?

1. `kubectl describe pod` - Check events for errors
2. `kubectl logs <pod>` - Check application logs
3. `kubectl logs <pod> --previous` - Check logs from crashed container
4. Check for OOMKilled in lastState
5. Check resource limits
6. Check liveness probe configuration
7. Try running the container command manually

### Q: How do you debug a networking issue in Kubernetes?

1. Check if Service has endpoints: `kubectl get endpoints`
2. Verify pod labels match service selector
3. Test pod-to-pod connectivity
4. Check NetworkPolicies
5. Check CoreDNS for DNS issues
6. Check kube-proxy logs
7. Use `kubectl debug` to add troubleshooting container

### Q: What tools do you use to monitor a Kubernetes cluster?

- **Metrics Server** - Resource usage
- **Prometheus + Grafana** - Metrics and dashboards
- **ELK/Loki** - Log aggregation
- **Jaeger/Zipkin** - Distributed tracing
- **k9s** - Terminal UI
- **kubectl top** - Quick resource checks

### Q: How do you handle a node failure?

1. Check node status: `kubectl get nodes`
2. Describe the node: `kubectl describe node <name>`
3. If permanently failed, drain and replace:
   ```bash
   kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
   kubectl delete node <node>
   ```
4. New node auto-registers with cluster
5. Pods are rescheduled automatically

### Q: What is the difference between kubectl describe and kubectl get -o yaml?

- `kubectl describe` - Human-readable format with events and conditions
- `kubectl get -o yaml` - Raw API object, complete spec and status, machine-readable

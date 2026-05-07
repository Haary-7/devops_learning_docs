# Kubernetes Comprehensive Cheat Sheet

## kubectl Configuration

```bash
# Context management
kubectl config get-contexts
kubectl config use-context <context>
kubectl config current-context
kubectl config set-context --current --namespace=<ns>

# View config
kubectl config view
kubectl config view --minify

# Cluster info
kubectl cluster-info
kubectl api-resources
kubectl api-versions
```

## Resource Management

### Create Resources

```bash
# From file
kubectl apply -f file.yaml
kubectl apply -f ./directory/
kubectl apply -f https://url/to/manifest.yaml

# Imperative
kubectl create deployment nginx --image=nginx
kubectl create service clusterip nginx --tcp=80:80
kubectl create configmap my-config --from-literal=key=value
kubectl create secret generic my-secret --from-literal=password=secret
kubectl create namespace production

# Dry run
kubectl apply -f file.yaml --dry-run=client
kubectl apply -f file.yaml --dry-run=server -o yaml

# Generate YAML
kubectl run nginx --image=nginx --dry-run=client -o yaml
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml
```

### Get Resources

```bash
# Basic
kubectl get pods
kubectl get services
kubectl get deployments
kubectl get nodes
kubectl get namespaces
kubectl get all

# Detailed
kubectl get pods -o wide
kubectl get pods -o yaml
kubectl get pods -o json
kubectl get pods -o custom-columns="NAME:.metadata.name,IP:.status.podIP"

# Filtering
kubectl get pods -l app=nginx
kubectl get pods -n kube-system
kubectl get pods --field-selector=status.phase=Running

# Watch
kubectl get pods -w
kubectl get events -w

# Sort
kubectl get pods --sort-by='.metadata.creationTimestamp'
kubectl get pods --sort-by='.status.containerStatuses[0].restartCount'
kubectl get events --sort-by='.lastTimestamp'
```

### Describe and Inspect

```bash
# Describe
kubectl describe pod <name>
kubectl describe node <name>
kubectl describe svc <name>

# Inspect
kubectl get pod <name> -o jsonpath='{.spec.containers[*].image}'
kubectl get pod <name> -o jsonpath='{.status.podIP}'
kubectl get svc <name> -o jsonpath='{.spec.clusterIP}'

# Labels
kubectl get pods --show-labels
kubectl label pod <name> key=value
kubectl label pod <name> key-
```

### Edit and Update

```bash
# Edit live
kubectl edit deployment/nginx
kubectl edit svc/nginx

# Update image
kubectl set image deployment/nginx nginx=nginx:1.26
kubectl set image deploy/nginx nginx=nginx:1.26 --record

# Scale
kubectl scale deployment/nginx --replicas=5
kubectl scale deployment/nginx --replicas=5 --record

# Annotate
kubectl annotate pod <name> description='my pod'
kubectl annotate pod <name> description-
```

### Delete Resources

```bash
# From file
kubectl delete -f file.yaml

# By name
kubectl delete pod <name>
kubectl delete deployment <name>
kubectl delete svc <name>
kubectl delete namespace <name>

# By label
kubectl delete pods -l app=nginx

# Force delete
kubectl delete pod <name> --grace-period=0 --force

# Delete all in namespace
kubectl delete all --all -n <namespace>
```

## Pods

```bash
# Run pod
kubectl run nginx --image=nginx
kubectl run nginx --image=nginx --port=80 --labels="app=nginx"

# Logs
kubectl logs <pod>
kubectl logs -f <pod>
kubectl logs --tail=100 <pod>
kubectl logs <pod> -c <container>
kubectl logs <pod> --previous

# Exec
kubectl exec -it <pod> -- /bin/bash
kubectl exec -it <pod> -- /bin/sh
kubectl exec <pod> -- ls /app

# Copy
kubectl cp <pod>:/path/to/file ./local-file
kubectl cp ./local-file <pod>:/path/to/dest

# Port forward
kubectl port-forward <pod> 8080:80
kubectl port-forward svc/<name> 8080:80

# Top
kubectl top pods
kubectl top pods -c
kubectl top pods --sort-by=memory
```

## Deployments

```bash
# Create
kubectl create deployment nginx --image=nginx --replicas=3

# Rollout
kubectl rollout status deployment/nginx
kubectl rollout history deployment/nginx
kubectl rollout history deployment/nginx --revision=2
kubectl rollout undo deployment/nginx
kubectl rollout undo deployment/nginx --to-revision=2
kubectl rollout pause deployment/nginx
kubectl rollout resume deployment/nginx

# Status
kubectl get deployments
kubectl describe deployment/nginx
```

## StatefulSet

```bash
kubectl create -f statefulset.yaml
kubectl get statefulset
kubectl describe statefulset <name>

# Scale
kubectl scale statefulset <name> --replicas=5

# Delete pod (by ordinal)
kubectl delete pod <name>-0
```

## Services

```bash
# Create
kubectl expose deployment/nginx --port=80 --target-port=8080 --type=ClusterIP
kubectl expose deployment/nginx --port=80 --type=NodePort
kubectl expose deployment/nginx --port=80 --type=LoadBalancer

# Get
kubectl get svc
kubectl get endpoints <svc>
kubectl describe svc <name>

# External IP
kubectl get svc -o jsonpath='{.items[*].status.loadBalancer.ingress[*].ip}'
```

## ConfigMaps and Secrets

```bash
# ConfigMap
kubectl create configmap <name> --from-literal=key=value
kubectl create configmap <name> --from-file=config.txt
kubectl create configmap <name> --from-file=./config-dir/
kubectl create configmap <name> --from-env-file=.env
kubectl get configmap <name> -o yaml

# Secret
kubectl create secret generic <name> --from-literal=key=value
kubectl create secret generic <name> --from-file=key=./file
kubectl create secret tls <name> --cert=cert.pem --key=key.pem
kubectl create secret docker-registry <name> --docker-server=... --docker-username=... --docker-password=...
kubectl get secret <name> -o jsonpath='{.data.key}' | base64 -d
```

## Storage

```bash
# PV
kubectl get pv
kubectl describe pv <name>

# PVC
kubectl get pvc
kubectl describe pvc <name>

# StorageClass
kubectl get storageclass
kubectl describe storageclass <name>

# Volumes
kubectl get volumes
```

## RBAC

```bash
# ServiceAccounts
kubectl get serviceaccounts
kubectl describe sa <name>

# Roles
kubectl get roles -n <namespace>
kubectl get clusterroles
kubectl describe role <name> -n <namespace>
kubectl describe clusterrole <name>

# Bindings
kubectl get rolebindings -n <namespace>
kubectl get clusterrolebindings
kubectl describe rolebinding <name> -n <namespace>

# Auth check
kubectl auth can-i create pods
kubectl auth can-i create pods --as=system:serviceaccount:default:my-sa
kubectl auth can-i --list
```

## Scheduling

```bash
# Node management
kubectl get nodes
kubectl describe node <name>
kubectl label node <name> key=value
kubectl taint node <name> key=value:NoSchedule
kubectl taint node <name> key=value:NoSchedule-

# Cordon/Uncordon/Drain
kubectl cordon <node>
kubectl uncordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

## Namespaces

```bash
kubectl get namespaces
kubectl create namespace <name>
kubectl delete namespace <name>
kubectl describe namespace <name>

# Resource quota
kubectl get resourcequota -n <name>
kubectl describe resourcequota <name> -n <name>

# Limit range
kubectl get limitrange -n <name>
```

## Troubleshooting

```bash
# Events
kubectl get events --sort-by='.lastTimestamp'
kubectl get events -n <namespace>

# Node conditions
kubectl describe node <name> | grep -A 5 Conditions

# Check kubelet
systemctl status kubelet
journalctl -u kubelet -f

# Check API server
kubectl get --raw=/healthz

# DNS test
kubectl run dns-test --image=busybox --rm -it -- nslookup kubernetes.default

# Network test
kubectl run test --image=busybox --rm -it -- wget -qO- http://<svc>:<port>

# Debug container (K8s 1.24+)
kubectl debug -it <pod> --image=busybox --target=<container>

# Node debug
kubectl debug node/<node> -it --image=ubuntu
```

## Helm

```bash
# Repo
helm repo add <name> <url>
helm repo update
helm repo list
helm repo remove <name>

# Search
helm search repo <keyword>
helm search hub <keyword>

# Install
helm install <release> <chart>
helm install <release> <chart> -f values.yaml
helm install <release> <chart> --set key=value
helm install <release> <chart> --namespace <ns> --create-namespace
helm install <release> <chart> --dry-run --debug

# Manage
helm list
helm list -A
helm status <release>
helm history <release>

# Upgrade/Rollback
helm upgrade <release> <chart>
helm upgrade <release> <chart> --atomic
helm rollback <release>
helm rollback <release> <revision>

# Uninstall
helm uninstall <release>

# Template
helm template <release> <chart>
helm template <release> <chart> -f values.yaml

# Chart
helm create <chart-name>
helm lint <chart>
helm package <chart>
helm dependency update <chart>
```

## JSONPath Queries

```bash
# Pod IPs
kubectl get pods -o jsonpath='{.items[*].status.podIP}'

# Container images
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'

# Node names
kubectl get nodes -o jsonpath='{.items[*].metadata.name}'

# All pod names and statuses
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# Service external IPs
kubectl get svc -o jsonpath='{.items[*].status.loadBalancer.ingress[*].ip}'

# ConfigMap keys
kubectl get configmap <name> -o jsonpath='{.data}'
```

## Common YAML Patterns

### Basic Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
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
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

### Complete Pod Spec

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: complete-pod
  labels:
    app: myapp
    env: production
spec:
  restartPolicy: Always
  serviceAccountName: my-sa
  
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  
  containers:
  - name: app
    image: myapp:1.0
    imagePullPolicy: IfNotPresent
    ports:
    - containerPort: 8080
      name: http
    env:
    - name: APP_ENV
      value: "production"
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: db-host
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    resources:
      requests:
        cpu: 250m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
    volumeMounts:
    - name: data
      mountPath: /data
    - name: config
      mountPath: /etc/config
      readOnly: true
  
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
  - name: config
    configMap:
      name: app-config
```

## Quick Reference

### Port Numbers

| Resource | Default Port |
|----------|-------------|
| kube-apiserver | 6443 |
| etcd | 2379, 2380 |
| kubelet | 10250 |
| kube-proxy | 10256 |
| CoreDNS | 53 |
| NodePort range | 30000-32767 |

### Default Namespaces

| Namespace | Purpose |
|-----------|---------|
| default | User resources |
| kube-system | System components |
| kube-public | Publicly readable |
| kube-node-lease | Node heartbeats |

### Resource Shortcuts

```
po    - pods
svc   - services
deploy - deployments
rs    - replicasets
sts   - statefulsets
ds    - daemonsets
ns    - namespaces
cm    - configmaps
sa    - serviceaccounts
pv    - persistentvolumes
pvc   - persistentvolumeclaims
sc    - storageclasses
ing   - ingress
hpa   - horizontalpodautoscalers
cj    - cronjobs
```

### QoS Classes

| Class | Requirement |
|-------|------------|
| Guaranteed | requests == limits for all containers |
| Burstable | At least one request or limit set |
| BestEffort | No requests or limits set |

### Priority

```
System Node Critical    (highest)
System Cluster Critical
User-defined (1000000+)
Default (0)
```

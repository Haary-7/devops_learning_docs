# Container Cheat Sheet

## Docker Images

```bash
# List images
docker images
docker image ls

# Pull an image
docker pull <image>:<tag>

# Search images on Docker Hub
docker search <keyword>

# Build an image
docker build -t <name>:<tag> <context>

# Tag an image
docker tag <source>:<tag> <target>:<tag>

# Remove an image
docker rmi <image>
docker image rm <image>

# Remove unused images
docker image prune
docker image prune -a  # Remove all unused, not just dangling

# Save/Load image as tar
docker save -o image.tar <image>:<tag>
docker load -i image.tar

# Inspect image
docker inspect <image>

# View image history (layers)
docker history <image>

# Push to registry
docker push <registry>/<image>:<tag>

# Login to registry
docker login
docker login <registry>
```

## Docker Containers

```bash
# Run container
docker run [OPTIONS] <image> [COMMAND]

# Common run options
#   -d              Detached mode
#   -p <host>:<ctr> Port mapping
#   -v <host>:<ctr> Volume mount
#   -e KEY=value    Environment variable
#   --name <name>   Container name
#   --rm            Auto-remove on exit
#   -it             Interactive terminal
#   --network <net> Network to use
#   --restart always  Restart policy

# List containers
docker ps           # Running
docker ps -a        # All
docker ps -s        # With sizes

# Start/Stop/Restart
docker start <container>
docker stop <container>
docker restart <container>

# Remove container
docker rm <container>
docker rm -f <container>  # Force remove running container

# Remove all stopped containers
docker container prune

# Logs
docker logs <container>
docker logs -f <container>
docker logs --tail 100 <container>

# Exec
docker exec -it <container> /bin/bash
docker exec -it <container> /bin/sh

# Inspect
docker inspect <container>

# Stats
docker stats
docker stats <container>

# Copy files
docker cp <container>:<src> <host-dest>
docker cp <host-src> <container>:<dest>

# View changes to container filesystem
docker diff <container>

# Pause/Unpause
docker pause <container>
docker unpause <container>
```

## Docker Compose

```bash
# Start services
docker compose up
docker compose up -d       # Detached
docker compose up --build  # Rebuild images

# Stop and remove
docker compose down
docker compose down -v     # Remove volumes too

# View status
docker compose ps
docker compose ps -a

# Logs
docker compose logs
docker compose logs -f <service>

# Build
docker compose build

# Execute command in service
docker compose exec <service> <command>

# Run one-off command
docker compose run <service> <command>

# Scale
docker compose up -d --scale <service>=<count>
```

## Docker Volumes

```bash
# List volumes
docker volume ls

# Create volume
docker volume create <name>

# Inspect volume
docker volume inspect <name>

# Remove volume
docker volume rm <name>

# Remove unused volumes
docker volume prune
```

## Docker Networks

```bash
# List networks
docker network ls

# Create network
docker network create <name>
docker network create -d bridge <name>

# Inspect network
docker network inspect <name>

# Connect container to network
docker network connect <network> <container>

# Disconnect container
docker network disconnect <network> <container>

# Remove network
docker network rm <name>

# Remove unused networks
docker network prune
```

## Docker Swarm

```bash
# Initialize
docker swarm init

# Join swarm
docker swarm join --token <token> <manager>:2377
docker swarm join-token worker   # Get worker join token
docker swarm join-token manager  # Get manager join token

# Nodes
docker node ls
docker node inspect <node>
docker node promote <node>       # Promote to manager
docker node demote <node>        # Demote to worker
docker node rm <node>            # Remove node

# Services
docker service create --name <name> <image>
docker service ls
docker service ps <service>
docker service inspect <service>
docker service logs <service>
docker service scale <service>=<count>
docker service update <options> <service>
docker service rm <service>

# Stacks
docker stack deploy -c <compose.yml> <stack-name>
docker stack ls
docker stack ps <stack-name>
docker stack services <stack-name>
docker stack rm <stack-name>
```

## Kubernetes (kubectl)

```bash
# Configuration
kubectl config get-contexts
kubectl config use-context <context>
kubectl config current-context

# Resources
kubectl get pods
kubectl get services
kubectl get deployments
kubectl get nodes
kubectl get namespaces
kubectl get all -n <namespace>
kubectl get <resource> -o wide
kubectl get <resource> -o yaml

# Create/Update/Delete
kubectl apply -f <file.yaml>
kubectl apply -f <directory>/
kubectl create -f <file.yaml>
kubectl delete -f <file.yaml>
kubectl delete pod <name>
kubectl delete deployment <name>

# Debugging
kubectl describe pod <name>
kubectl describe node <name>
kubectl logs <pod>
kubectl logs -f <pod>
kubectl logs <pod> -c <container>   # Multi-container pod
kubectl exec -it <pod> -- <command>
kubectl cp <pod>:<path> <local-path>

# Scaling
kubectl scale deployment/<name> --replicas=<count>

# Rollouts
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>

# Port forwarding
kubectl port-forward <pod> <local-port>:<pod-port>

# Namespaces
kubectl create namespace <name>
kubectl get pods -n <namespace>
kubectl config set-context --current --namespace=<name>
```

## Kubernetes Resource Types (Shortcuts)

```
pods          -> po
services      -> svc
deployments   -> deploy
replicasets   -> rs
statefulsets  -> sts
daemonsets    -> ds
namespaces    -> ns
configmaps    -> cm
secrets       -> secret
ingress       -> ing
persistentvolumeclaims -> pvc
jobs          -> job
cronjobs      -> cj
```

## Environment Variables

```bash
# Pass to container
docker run -e KEY=value <image>
docker run -e KEY=value -e KEY2=value2 <image>

# From file
docker run --env-file .env <image>

# Docker Compose
services:
  app:
    environment:
      - DB_HOST=db
      - DB_PORT=5432
    env_file:
      - .env.production
```

## Common Patterns

### Cleanup everything
```bash
docker system prune -a --volumes
```

### Inspect disk usage
```bash
docker system df
docker system df -v
```

### Run with restart policy
```bash
docker run -d --restart unless-stopped <image>
docker run -d --restart always <image>
docker run -d --restart on-failure:5 <image>
```

### Read-only container
```bash
docker run --read-only --tmpfs /tmp <image>
```

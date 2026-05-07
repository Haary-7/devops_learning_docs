# Containers

## What is a Container?

A container is a running instance of a container image. It packages an application and its dependencies together, isolating it from the host system and other containers.

## Containers vs Virtual Machines

| Feature | Containers | Virtual Machines |
|---------|------------|------------------|
| OS | Shares host OS kernel | Full guest OS per VM |
| Size | Megabytes | Gigabytes |
| Boot time | Seconds | Minutes |
| Overhead | Minimal | Significant |
| Isolation | Process-level | Hardware-level |
| Density | High (100s per host) | Low (10s per host) |

## Container Architecture

```
┌─────────────────────────────────────┐
│           Applications              │
├─────────────────────────────────────┤
│     Container Engine (Docker)       │
├─────────────────────────────────────┤
│         Host Operating System       │
├─────────────────────────────────────┤
│        Hardware (Server/Cloud)      │
└─────────────────────────────────────┘
```

## Running Containers

### Basic Commands

```bash
# Run a container
docker run nginx

# Run in detached mode (background)
docker run -d nginx

# Run with port mapping
docker run -d -p 8080:80 nginx

# Run with a name
docker run -d --name my-web nginx

# Run with environment variables
docker run -d -e DB_HOST=localhost -e DB_PORT=5432 myapp

# Run with volume mount
docker run -d -v /host/path:/container/path myapp

# Run interactively
docker run -it ubuntu bash

# Run with resource limits
docker run -d --memory=512m --cpus=1.5 myapp

# Run and remove when stopped
docker run --rm alpine echo "Hello"
```

### Container Lifecycle

```
Created → Running → Paused/Stopped → Removed
                ↘               ↗
                  Exited (with code)
```

## Managing Containers

### Listing Containers

```bash
# Running containers
docker ps

# All containers (including stopped)
docker ps -a

# Filter containers
docker ps --filter "status=running"
docker ps --filter "name=web"

# Show container sizes
docker ps -s

# Custom format
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

### Container Operations

```bash
# Stop a container (graceful)
docker stop <container>

# Kill a container (immediate)
docker kill <container>

# Start a stopped container
docker start <container>

# Restart a container
docker restart <container>

# Pause/Unpause
docker pause <container>
docker unpause <container>

# Remove a container
docker rm <container>

# Remove all stopped containers
docker container prune
```

### Inspecting Containers

```bash
# View container logs
docker logs <container>
docker logs -f <container>         # Follow logs
docker logs --tail 100 <container> # Last 100 lines
docker logs --since 10m <container># Last 10 minutes

# Inspect container details
docker inspect <container>

# View resource usage
docker stats
docker stats <container>

# View processes inside container
docker top <container>
docker exec <container> ps aux
```

### Executing Commands in Containers

```bash
# Run a command in a running container
docker exec <container> ls /app

# Open a shell inside a container
docker exec -it <container> /bin/bash
docker exec -it <container> /bin/sh

# Run as specific user
docker exec -u root <container> whoami
```

## Networking

### Network Types

| Type | Description |
|------|-------------|
| `bridge` | Default network for containers |
| `host` | Container shares host's network namespace |
| `none` | No networking |
| `overlay` | Multi-host networking (Swarm/Kubernetes) |
| `macvlan` | Assign MAC address to containers |

### Network Commands

```bash
# List networks
docker network ls

# Create a network
docker network create my-network

# Run container on a specific network
docker run -d --network my-network nginx

# Connect a running container to a network
docker network connect my-network <container>

# Inspect network
docker network inspect my-network

# Remove network
docker network rm my-network
```

## Storage and Volumes

### Volume Types

| Type | Description | Use Case |
|------|-------------|----------|
| **Named Volume** | Managed by Docker, stored in Docker directory | Persistent data |
| **Bind Mount** | Maps host file/directory to container | Development, config files |
| **tmpfs** | In-memory storage | Temporary data, secrets |

### Volume Commands

```bash
# List volumes
docker volume ls

# Create a volume
docker volume create my-data

# Run with a named volume
docker run -d -v my-data:/var/lib/mysql mysql

# Run with a bind mount
docker run -d -v /host/path:/container/path nginx

# Run with read-only bind mount
docker run -d -v /host/config:/container/config:ro nginx

# Inspect a volume
docker volume inspect my-data

# Remove unused volumes
docker volume prune
```

## Docker Compose

Docker Compose defines and runs multi-container applications using a YAML file.

### Example docker-compose.yml

```yaml
version: "3.8"

services:
  web:
    build: .
    ports:
      - "8080:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:secret@db:5432/myapp
    depends_on:
      - db
    volumes:
      - ./app:/app
    networks:
      - app-network

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=myapp
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - app-network

volumes:
  postgres-data:

networks:
  app-network:
    driver: bridge
```

### Compose Commands

```bash
# Start services
docker compose up
docker compose up -d  # Detached mode

# Stop services
docker compose down

# View logs
docker compose logs
docker compose logs -f web

# Build and start
docker compose up --build

# Scale a service
docker compose up -d --scale web=3

# Run a one-off command
docker compose run web python manage.py migrate
```

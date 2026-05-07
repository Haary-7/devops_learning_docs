# Hands-On Container Exercises

## Prerequisites

- Docker installed (`docker --version`)
- Docker Compose installed (`docker compose version`)
- Basic command line knowledge

---

## Exercise 1: Your First Container

**Goal:** Run an Nginx web server container.

```bash
# 1. Pull the Nginx image
docker pull nginx:alpine

# 2. Run a container in detached mode with port mapping
docker run -d --name my-nginx -p 8080:80 nginx:alpine

# 3. Verify it's running
docker ps

# 4. Open your browser and visit http://localhost:8080

# 5. View logs
docker logs my-nginx

# 6. Stop and remove the container
docker stop my-nginx
docker rm my-nginx
```

**Check:** You should see the Nginx welcome page at http://localhost:8080

---

## Exercise 2: Build a Custom Image

**Goal:** Create a Dockerfile and build a custom image.

### Step 1: Create project structure

```bash
mkdir -p exercise-2 && cd exercise-2
```

### Step 2: Create an HTML file

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head><title>My Docker App</title></head>
<body>
  <h1>Hello from my custom Docker image!</h1>
</body>
</html>
```

### Step 3: Create a Dockerfile

```dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### Step 4: Build and run

```bash
# Build the image
docker build -t my-custom-nginx:1.0 .

# Run the container
docker run -d --name custom-nginx -p 8081:80 my-custom-nginx:1.0

# Test it
curl http://localhost:8081

# Clean up
docker stop custom-nginx
docker rm custom-nginx
```

---

## Exercise 3: Multi-Stage Build

**Goal:** Create a small production image using multi-stage builds.

```bash
mkdir -p exercise-3 && cd exercise-3
```

### Create a Go application

```go
// main.go
package main

import (
	"fmt"
	"net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello from a multi-stage build!")
}

func main() {
	http.HandleFunc("/", handler)
	fmt.Println("Server running on :8080")
	http.ListenAndServe(":8080", nil)
}
```

### Create a multi-stage Dockerfile

```dockerfile
# Stage 1: Build
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY main.go .
RUN CGO_ENABLED=0 GOOS=linux go build -o server .

# Stage 2: Run
FROM alpine:latest

RUN apk --no-cache add ca-certificates
WORKDIR /root/

COPY --from=builder /app/server .

EXPOSE 8080

CMD ["./server"]
```

### Build and run

```bash
docker build -t go-app:latest .

# Compare sizes
docker images | grep go-app

docker run -d --name go-server -p 8082:8080 go-app:latest

curl http://localhost:8082

docker stop go-server && docker rm go-server
```

---

## Exercise 4: Docker Compose - Full Stack App

**Goal:** Run a web app with a database using Docker Compose.

```bash
mkdir -p exercise-4 && cd exercise-4
```

### Create docker-compose.yml

```yaml
version: "3.8"

services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
    depends_on:
      - redis
    networks:
      - frontend

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - frontend

volumes:
  redis-data:

networks:
  frontend:
    driver: bridge
```

### Create HTML content

```bash
mkdir html
cat > html/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>Compose App</title></head>
<body>
  <h1>Docker Compose App</h1>
  <p>Nginx + Redis running together!</p>
</body>
</html>
EOF
```

### Run the stack

```bash
# Start services
docker compose up -d

# Check status
docker compose ps

# View logs
docker compose logs

# Test Nginx
curl http://localhost:8080

# Test Redis
docker exec -it exercise-4-redis-1 redis-cli ping

# Stop and remove (keep volumes)
docker compose down

# Stop and remove (including volumes)
docker compose down -v
```

---

## Exercise 5: Docker Networking

**Goal:** Create custom networks and connect containers.

```bash
# 1. Create a custom bridge network
docker network create my-network

# 2. Run containers on the network
docker run -d --name app1 --network my-network nginx:alpine
docker run -d --name app2 --network my-network nginx:alpine

# 3. Verify they can communicate
docker exec app1 ping -c 2 app2

# 4. Inspect the network
docker network inspect my-network

# 5. Connect a new container to the network
docker run -d --name app3 alpine sleep 3600
docker network connect my-network app3

# 6. Verify connectivity
docker exec app3 ping -c 2 app1

# 7. Clean up
docker stop app1 app2 app3
docker rm app1 app2 app3
docker network rm my-network
```

---

## Exercise 6: Volumes and Data Persistence

**Goal:** Understand Docker volumes and data persistence.

```bash
# 1. Create a named volume
docker volume create my-data

# 2. Run a container that writes to the volume
docker run -d --name writer -v my-data:/data alpine \
  sh -c "while true; do echo $(date) >> /data/log.txt; sleep 5; done"

# 3. Verify data is being written
docker exec writer cat /data/log.txt

# 4. Stop and remove the container
docker stop writer
docker rm writer

# 5. Run a new container with the same volume
docker run --name reader -v my-data:/data alpine cat /data/log.txt

# 6. Data persists! Clean up
docker rm reader
docker volume rm my-data
```

### Bind Mount Exercise

```bash
# 1. Create a directory on the host
mkdir -p /tmp/docker-data
echo "Host file content" > /tmp/docker-data/host.txt

# 2. Mount it into a container
docker run --rm -v /tmp/docker-data:/mnt alpine cat /mnt/host.txt

# 3. Write from the container
docker run --rm -v /tmp/docker-data:/mnt alpine sh -c "echo 'From container' > /mnt/container.txt"

# 4. Verify on host
cat /tmp/docker-data/container.txt
```

---

## Exercise 7: Environment Variables and Secrets

```bash
# 1. Run with environment variables
docker run --rm -e APP_ENV=production -e APP_PORT=3000 \
  alpine sh -c 'echo "App running in $APP_ENV on port $APP_PORT"'

# 2. Using env files
cat > .env << EOF
DB_HOST=localhost
DB_PORT=5432
DB_USER=admin
DB_PASS=secret123
EOF

docker run --rm --env-file .env alpine env | grep DB

# 3. Using Docker secrets (Swarm mode)
echo "supersecret" | docker secret create db-pass -

# Verify
docker secret ls

# Clean up
docker secret rm db-pass
rm .env
```

---

## Exercise 8: Resource Limits

```bash
# 1. Run with memory and CPU limits
docker run -d --name limited \
  --memory=256m \
  --memory-reservation=128m \
  --cpus=0.5 \
  nginx:alpine

# 2. Verify limits
docker inspect limited --format='{{.HostConfig.Memory}}'
docker inspect limited --format='{{.HostConfig.NanoCpus}}'

# 3. View resource usage
docker stats limited

# 4. Clean up
docker stop limited && docker rm limited
```

---

## Exercise 9: Health Checks

### Create an application with health check

```bash
mkdir -p exercise-9 && cd exercise-9
```

```dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN pip install flask

COPY app.py .

EXPOSE 5000

HEALTHCHECK --interval=10s --timeout=3s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:5000/health')" || exit 1

CMD ["python", "app.py"]
```

```python
# app.py
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/')
def index():
    return jsonify({"message": "Hello World"})

@app.route('/health')
def health():
    return jsonify({"status": "healthy"}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

```bash
# Build and run
docker build -t healthcheck-app .
docker run -d --name hc-app -p 5000:5000 healthcheck-app

# Wait a moment, then check health
docker inspect hc-app --format='{{.State.Health.Status}}'

# Clean up
docker stop hc-app && docker rm hc-app
docker rmi healthcheck-app
```

---

## Exercise 10: Container Security Best Practices

```bash
# 1. Run as non-root user
docker run --rm --user 1000:1000 alpine whoami

# 2. Run with read-only filesystem
docker run -d --name ro-app \
  --read-only \
  --tmpfs /tmp \
  nginx:alpine

# 3. Drop capabilities
docker run -d --name secure-app \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  nginx:alpine

# 4. No new privileges
docker run --rm --security-opt=no-new-privileges alpine echo "Secure!"

# 5. Clean up
docker stop ro-app secure-app 2>/dev/null
docker rm ro-app secure-app 2>/dev/null
```

---

## Exercise 11: Export and Import Containers

```bash
# 1. Run and customize a container
docker run -d --name custom alpine sleep 3600
docker exec custom sh -c "echo 'Customized!' > /custom.txt"

# 2. Export container filesystem
docker export custom > container.tar

# 3. Import as a new image
cat container.tar | docker import - my-exported:latest

# 4. Run the imported image
docker run --rm my-exported:latest cat /custom.txt

# 5. Clean up
docker stop custom && docker rm custom
docker rmi my-exported:latest
rm container.tar
```

---

## Challenge Projects

### Project 1: Three-Tier Application

Build a docker-compose setup with:
- Frontend (React/Vue static site on Nginx)
- Backend (Node.js/Python API)
- Database (PostgreSQL/MySQL)

Requirements:
- All services on a custom network
- Persistent database volume
- Environment variables for configuration
- Health checks for all services

### Project 2: CI/CD Pipeline Simulation

Create a Docker-based build pipeline:
1. Build stage image with code
2. Test stage image that runs tests
3. Production stage with optimized image

### Project 3: Monitoring Stack

Set up a monitoring stack with:
- Prometheus (metrics collection)
- Grafana (visualization)
- Application that exposes metrics

---

## Cleanup

After completing all exercises:

```bash
# Stop all running containers
docker stop $(docker ps -q) 2>/dev/null

# Remove all containers
docker rm $(docker ps -aq) 2>/dev/null

# Remove all images
docker rmi $(docker images -q) 2>/dev/null

# Remove all volumes
docker volume prune -f

# Remove all networks
docker network prune -f

# System cleanup
docker system prune -a --volumes -f
```

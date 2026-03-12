# Docker Configuration

## Overview

PrimeFlow uses Docker Compose for local development with 6 services:
- nginx (reverse proxy)
- gateway (API gateway)
- users (users service)
- tenant (tenant service)
- webapp (web application)
- website (marketing site)

---

## Docker Compose Configuration

### File Location
`docker-compose.yml` (project root)

### Full Configuration

```yaml
version: '3.8'

services:
  # Nginx Reverse Proxy
  nginx:
    image: nginx:alpine
    container_name: primeflow-nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - gateway
      - webapp
      - website
    networks:
      - primeflow-network

  # API Gateway Service
  gateway:
    build:
      context: ./go-gateway-service
      dockerfile: Dockerfile.dev
    container_name: primeflow-gateway
    environment:
      - PORT=8080
      - JWT_SIGNING_KEY=your-secret-key-change-in-production
      - JWT_EXPIRATION=168h
      - USER_SERVICE_URL=http://users:8081
      - TENANT_SERVICE_URL=http://tenant:8082
      - PUBLIC_ROUTES=/users/v1/login,/tenant/v1/signup,/v1/health,/tenant/v1/health,/users/v1/health
      - LOG_LEVEL=info
    networks:
      - primeflow-network
    depends_on:
      - users
      - tenant

  # Users Service
  users:
    build:
      context: ./go-users-service
      dockerfile: Dockerfile.dev
    container_name: primeflow-users
    environment:
      - PORT=8081
      - DB_HOST=trolley.proxy.rlwy.net
      - DB_PORT=11306
      - DB_USERNAME=postgres
      - DB_PASSWORD=clBcJPLpujzLWqRfXejgqvClSdJWefYn
      - JWT_SIGNING_KEY=your-secret-key-change-in-production
      - JWT_EXPIRATION=168h
      - LOG_LEVEL=info
    networks:
      - primeflow-network

  # Tenant Service
  tenant:
    build:
      context: ./go-tenant-service
      dockerfile: Dockerfile.dev
    container_name: primeflow-tenant
    environment:
      - PORT=8082
      - DB_HOST=trolley.proxy.rlwy.net
      - DB_PORT=11306
      - DB_USERNAME=postgres
      - DB_PASSWORD=clBcJPLpujzLWqRfXejgqvClSdJWefYn
      - JWT_SIGNING_KEY=your-secret-key-change-in-production
      - JWT_EXPIRATION=168h
      - LOG_LEVEL=info
    networks:
      - primeflow-network

  # WebApp Frontend
  webapp:
    build:
      context: ./ts-webapp-frontend
      dockerfile: Dockerfile.dev
    container_name: primeflow-webapp
    volumes:
      - ./ts-webapp-frontend:/app
      - /app/node_modules
    networks:
      - primeflow-network

  # Website Frontend
  website:
    build:
      context: ./ts-website-frontend
      dockerfile: Dockerfile.dev
    container_name: primeflow-website
    volumes:
      - ./ts-website-frontend:/app
      - /app/node_modules
    networks:
      - primeflow-network

networks:
  primeflow-network:
    driver: bridge
```

---

## Nginx Configuration

### File Location
`nginx.conf` (project root)

### Configuration

```nginx
events {
    worker_connections 1024;
}

http {
    # Upstream definitions
    upstream gateway {
        server gateway:8080;
    }

    upstream webapp {
        server webapp:4200;
    }

    upstream website {
        server website:4200;
    }

    # Website (localhost)
    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://website;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }
    }

    # WebApp (app.localhost)
    server {
        listen 80;
        server_name app.localhost;

        location / {
            proxy_pass http://webapp;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }
    }

    # API Gateway (api.localhost)
    server {
        listen 80;
        server_name api.localhost;

        # CORS Configuration
        add_header 'Access-Control-Allow-Origin' '*' always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
        add_header 'Access-Control-Allow-Headers' 'Content-Type, Authorization, X-Tenant-Slug' always;
        add_header 'Access-Control-Max-Age' 1728000 always;

        # Handle preflight requests
        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin' '*';
            add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'Content-Type, Authorization, X-Tenant-Slug';
            add_header 'Access-Control-Max-Age' 1728000;
            add_header 'Content-Type' 'text/plain; charset=utf-8';
            add_header 'Content-Length' 0;
            return 204;
        }

        location / {
            proxy_pass http://gateway;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

---

## Dockerfiles

### Go Services Dockerfile

**Location:** `go-*-service/Dockerfile.dev`

```dockerfile
# Build stage
FROM golang:1.24-alpine AS builder

# Install build dependencies
RUN apk add --no-cache git

# Set working directory
WORKDIR /build

# Copy go-pkg-lib (shared library)
COPY ../go-pkg-lib /build/go-pkg-lib

# Copy service code
COPY go-*-service /build/go-*-service

# Set working directory to service
WORKDIR /build/go-*-service

# Download dependencies
RUN GOTOOLCHAIN=auto go mod download

# Build the application
RUN GOTOOLCHAIN=auto CGO_ENABLED=0 GOOS=linux go build -o service ./cmd/*/

# Runtime stage
FROM alpine:latest

# Install ca-certificates for HTTPS
RUN apk --no-cache add ca-certificates

# Set working directory
WORKDIR /root/

# Copy binary from builder
COPY --from=builder /build/go-*-service/service .

# Copy migrations (if applicable)
COPY --from=builder /build/go-*-service/migrations ./migrations

# Expose port
EXPOSE 8080

# Run the application
CMD ["./service"]
```

### Angular Frontend Dockerfile

**Location:** `ts-*-frontend/Dockerfile.dev`

```dockerfile
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci

# Copy source code
COPY . .

# Expose port
EXPOSE 4200

# Start development server
CMD ["npm", "start", "--", "--host", "0.0.0.0", "--poll", "2000"]
```

---

## Docker Commands

### Starting Services

```bash
# Start all services
docker-compose up

# Start in detached mode
docker-compose up -d

# Rebuild and start
docker-compose up --build

# Start specific service
docker-compose up gateway
```

### Stopping Services

```bash
# Stop all services
docker-compose down

# Stop and remove volumes
docker-compose down -v

# Stop specific service
docker-compose stop gateway
```

### Viewing Logs

```bash
# All services
docker-compose logs

# Specific service
docker-compose logs gateway

# Follow logs
docker-compose logs -f users

# Last N lines
docker-compose logs --tail=50 tenant
```

### Rebuilding Services

```bash
# Rebuild all
docker-compose build

# Rebuild specific service
docker-compose build gateway

# Rebuild without cache
docker-compose build --no-cache gateway
```

### Inspecting Services

```bash
# List running containers
docker-compose ps

# Inspect container
docker inspect primeflow-gateway

# Enter container shell
docker exec -it primeflow-gateway sh

# View container logs
docker logs primeflow-gateway
```

---

## Volume Management

### Named Volumes

Currently not using named volumes. Using bind mounts for development:

```yaml
volumes:
  - ./ts-webapp-frontend:/app
  - /app/node_modules  # Anonymous volume for node_modules
```

### Volume Commands

```bash
# List volumes
docker volume ls

# Remove unused volumes
docker volume prune

# Remove specific volume
docker volume rm <volume-name>
```

---

## Network Configuration

### Network Details

```yaml
networks:
  primeflow-network:
    driver: bridge
```

### Network Commands

```bash
# List networks
docker network ls

# Inspect network
docker network inspect primeflowsolutions_primeflow-network

# Connect container to network
docker network connect primeflow-network <container>

# Disconnect container from network
docker network disconnect primeflow-network <container>
```

---

## Environment Variables

### Development vs Production

**Development:**
```yaml
environment:
  - DB_HOST=trolley.proxy.rlwy.net
  - JWT_SIGNING_KEY=dev-secret-key
  - LOG_LEVEL=debug
```

**Production:**
```yaml
environment:
  - DB_HOST=${DB_HOST}
  - JWT_SIGNING_KEY=${JWT_SIGNING_KEY}
  - LOG_LEVEL=info
```

### Using .env File

Create `.env` file in project root:

```bash
DB_HOST=trolley.proxy.rlwy.net
DB_PORT=11306
DB_USERNAME=postgres
DB_PASSWORD=your-password
JWT_SIGNING_KEY=your-secret-key
```

Reference in docker-compose.yml:

```yaml
environment:
  - DB_HOST=${DB_HOST}
  - DB_PASSWORD=${DB_PASSWORD}
```

---

## Health Checks

### Docker Health Check

Add to service definition:

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/v1/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

### Checking Health

```bash
# View health status
docker-compose ps

# Inspect health
docker inspect --format='{{.State.Health.Status}}' primeflow-gateway
```

---

## Resource Limits

### CPU and Memory Limits

```yaml
services:
  gateway:
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
```

---

## Multi-Stage Builds

### Benefits
- Smaller final images
- Faster builds (caching)
- Separate build and runtime dependencies

### Example

```dockerfile
# Build stage
FROM golang:1.24-alpine AS builder
WORKDIR /build
COPY . .
RUN go build -o app

# Runtime stage
FROM alpine:latest
COPY --from=builder /build/app .
CMD ["./app"]
```

---

## Docker Compose Profiles

### Define Profiles

```yaml
services:
  gateway:
    profiles: ["backend"]
  
  webapp:
    profiles: ["frontend"]
```

### Use Profiles

```bash
# Start only backend services
docker-compose --profile backend up

# Start only frontend services
docker-compose --profile frontend up

# Start all services
docker-compose --profile backend --profile frontend up
```

---

## Troubleshooting

### Port Already in Use

```bash
# Find process using port
lsof -i :80

# Kill process
kill -9 <PID>

# Or change port in docker-compose.yml
ports:
  - "8080:80"
```

### Container Won't Start

```bash
# Check logs
docker-compose logs <service>

# Check container status
docker-compose ps

# Inspect container
docker inspect <container>
```

### Build Failures

```bash
# Clean build cache
docker builder prune

# Remove all images
docker-compose down --rmi all

# Rebuild from scratch
docker-compose build --no-cache
```

### Network Issues

```bash
# Recreate network
docker-compose down
docker network prune
docker-compose up
```

### Database Connection Issues

```bash
# Check if database is accessible
docker exec -it primeflow-gateway sh
ping trolley.proxy.rlwy.net

# Check environment variables
docker exec primeflow-gateway env | grep DB_
```

---

## Best Practices

### Development
- Use bind mounts for hot reload
- Use `.dockerignore` to exclude files
- Keep images small
- Use multi-stage builds
- Tag images properly

### Production
- Use specific image versions (not `latest`)
- Set resource limits
- Use health checks
- Enable logging
- Use secrets management
- Enable SSL/TLS

### Security
- Don't commit secrets to git
- Use environment variables
- Scan images for vulnerabilities
- Run as non-root user
- Keep images updated

---

## Docker Ignore

### .dockerignore File

```
# Git
.git
.gitignore

# Node
node_modules
npm-debug.log

# Go
*.exe
*.test
*.out

# IDE
.vscode
.idea

# OS
.DS_Store
Thumbs.db

# Logs
*.log

# Environment
.env
.env.local
```

---

## Production Deployment

### Build Production Images

```bash
# Build all services
docker-compose -f docker-compose.prod.yml build

# Tag images
docker tag primeflow-gateway:latest registry.example.com/primeflow-gateway:v1.0.0

# Push to registry
docker push registry.example.com/primeflow-gateway:v1.0.0
```

### Production Compose File

```yaml
version: '3.8'

services:
  gateway:
    image: registry.example.com/primeflow-gateway:v1.0.0
    restart: always
    environment:
      - JWT_SIGNING_KEY=${JWT_SIGNING_KEY}
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1'
          memory: 1G
```

---

## Monitoring

### Container Stats

```bash
# Real-time stats
docker stats

# Specific container
docker stats primeflow-gateway
```

### Logs

```bash
# Follow logs
docker-compose logs -f

# Export logs
docker-compose logs > logs.txt
```

### Metrics

Consider adding:
- Prometheus for metrics
- Grafana for visualization
- Loki for log aggregation

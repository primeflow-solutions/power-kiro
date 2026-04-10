---
inclusion: auto
---

# Docker Compose - Criação do Zero

## Passo 1: Criar docker-compose.yml

Crie o arquivo `docker-compose.yml` na raiz do projeto:

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

  # API Gateway
  gateway:
    build:
      context: .
      dockerfile: go-gateway-service/Dockerfile.dev
    container_name: primeflow-gateway
    environment:
      - PORT=8080
      - DB_HOST=your-db-host
      - DB_PORT=5432
      - DB_USERNAME=postgres
      - DB_PASSWORD=your-password
      - JWT_SIGNING_KEY=your-secret-key
      - TENANT_SERVICE_URL=http://tenant:8082
      - USER_SERVICE_URL=http://users:8081
      - ORGANIZATIONS_SERVICE_URL=http://organizations:8083
      - PUBLIC_ROUTES=/users/v1/login,/tenant/v1/signup,/tenant/v1/tenants,/health
    volumes:
      - ./go-gateway-service:/app
    networks:
      - primeflow-network
    restart: unless-stopped

  # Webapp (Angular/TypeScript)
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
    restart: unless-stopped

  # Website (Angular/TypeScript)
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
    restart: unless-stopped

networks:
  primeflow-network:
    driver: bridge
```

## Passo 2: Criar nginx.conf

Crie o arquivo `nginx.conf` na raiz do projeto:

```nginx
events {
    worker_connections 1024;
}

http {
    # API Gateway - api.localhost
    server {
        listen 80;
        server_name api.localhost;

        location / {
            proxy_pass http://gateway:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    # Webapp - app.localhost
    server {
        listen 80;
        server_name app.localhost;

        location / {
            proxy_pass http://webapp:4200;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # WebSocket support for dev server
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }

    # Website - localhost
    server {
        listen 80 default_server;
        server_name localhost;

        location / {
            proxy_pass http://website:4200;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # WebSocket support for dev server
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }
}
```

## Passo 3: Criar .dockerignore

Crie o arquivo `.dockerignore` na raiz do projeto:

```
# Git
.git
.gitignore

# Docker
docker-compose.yml
Dockerfile*
.dockerignore

# IDEs
.vscode
.idea
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Logs
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Dependencies (serão instaladas no container)
node_modules
vendor

# Build outputs
dist
build
*.exe
*.dll
*.so
*.dylib

# Test coverage
coverage
*.coverprofile

# Environment
.env
.env.local
.env.*.local
```

## Passo 4: Testar

```bash
# Iniciar todos os serviços
docker-compose up -d

# Ver logs
docker-compose logs -f

# Acessar
# - Website: http://localhost
# - Webapp: http://app.localhost
# - API: http://api.localhost
```

## Estrutura de Pastas Esperada

```
project-root/
├── docker-compose.yml
├── nginx.conf
├── .dockerignore
├── go-gateway-service/
│   ├── Dockerfile.dev
│   ├── cmd/
│   ├── internal/
│   └── go.mod
├── ts-webapp-frontend/
│   ├── Dockerfile.dev
│   ├── src/
│   └── package.json
└── ts-website-frontend/
    ├── Dockerfile.dev
    ├── src/
    └── package.json
```

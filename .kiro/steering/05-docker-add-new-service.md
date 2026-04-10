---
inclusion: auto
---

# Docker - Adicionar Novo Serviço

## Template para Novo Serviço Backend (Go)

### 1. Criar Dockerfile.dev

Crie `<service-name>/Dockerfile.dev`:

```dockerfile
FROM golang:1.24-alpine

# Install git and air for hot reload
RUN apk add --no-cache git
RUN go install github.com/air-verse/air@latest

# Set GOPRIVATE for private repos (se necessário)
ENV GOPRIVATE=github.com/primeflow-solutions/*

WORKDIR /app

# Copy go.mod and go.sum
COPY go.mod go.sum ./

# Download dependencies
RUN go mod download

# Copy source code
COPY . .

# Expose port
EXPOSE <PORT>

# Run with air for hot reload
CMD ["air", "-c", ".air.toml"]
```

### 2. Criar .air.toml

Crie `<service-name>/.air.toml`:

```toml
root = "."
testdata_dir = "testdata"
tmp_dir = "tmp"

[build]
  args_bin = []
  bin = "./tmp/main"
  cmd = "go build -o ./tmp/main ./cmd/<service-name>"
  delay = 1000
  exclude_dir = ["assets", "tmp", "vendor", "testdata"]
  exclude_file = []
  exclude_regex = ["_test.go"]
  exclude_unchanged = false
  follow_symlink = false
  full_bin = ""
  include_dir = []
  include_ext = ["go", "tpl", "tmpl", "html"]
  include_file = []
  kill_delay = "0s"
  log = "build-errors.log"
  poll = false
  poll_interval = 0
  rerun = false
  rerun_delay = 500
  send_interrupt = false
  stop_on_error = false

[color]
  app = ""
  build = "yellow"
  main = "magenta"
  runner = "green"
  watcher = "cyan"

[log]
  main_only = false
  time = false

[misc]
  clean_on_exit = false

[screen]
  clear_on_rebuild = false
  keep_scroll = true
```

### 3. Adicionar ao docker-compose.yml

Adicione o serviço em `docker-compose.yml`:

```yaml
services:
  # ... outros serviços ...

  # Novo Serviço
  <service-name>:
    build:
      context: .
      dockerfile: go-<service-name>-service/Dockerfile.dev
    container_name: primeflow-<service-name>
    environment:
      - PORT=<PORT>
      - DB_HOST=trolley.proxy.rlwy.net
      - DB_PORT=11306
      - DB_USERNAME=postgres
      - DB_PASSWORD=clBcJPLpujzLWqRfXejgqvClSdJWefYn
      - JWT_SIGNING_KEY=your-secret-key
    volumes:
      - ./go-<service-name>-service:/app
    networks:
      - primeflow-network
    restart: unless-stopped
```

### 4. Atualizar Gateway (se necessário)

Se o serviço precisa ser acessado via gateway, adicione em `docker-compose.yml` no serviço `gateway`:

```yaml
gateway:
  environment:
    - <SERVICE_NAME>_SERVICE_URL=http://<service-name>:<PORT>
```

### 5. Atualizar nginx.conf (se necessário)

Se o serviço precisa de acesso direto via nginx:

```nginx
http {
    # ... outros servers ...

    # Novo Serviço - <service-name>.localhost
    server {
        listen 80;
        server_name <service-name>.localhost;

        location / {
            proxy_pass http://<service-name>:<PORT>;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

### 6. Iniciar o Serviço

```bash
# Build e start do novo serviço
docker-compose up -d --build <service-name>

# Ver logs
docker-compose logs -f <service-name>
```

## Template para Novo Serviço Frontend (Angular/React)

### 1. Criar Dockerfile.dev

Crie `<frontend-name>/Dockerfile.dev`:

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies (ignore postinstall scripts)
RUN npm install --ignore-scripts

# Copy source code
COPY . .

# Expose port
EXPOSE <PORT>

# Start development server
CMD ["npm", "start", "--", "--host", "0.0.0.0", "--poll", "2000"]
```

### 2. Adicionar ao docker-compose.yml

```yaml
services:
  # ... outros serviços ...

  # Novo Frontend
  <frontend-name>:
    build:
      context: ./<frontend-name>
      dockerfile: Dockerfile.dev
    container_name: primeflow-<frontend-name>
    volumes:
      - ./<frontend-name>:/app
      - /app/node_modules
    networks:
      - primeflow-network
    restart: unless-stopped
```

### 3. Adicionar ao nginx.conf

```nginx
http {
    # ... outros servers ...

    # Novo Frontend - <frontend-name>.localhost
    server {
        listen 80;
        server_name <frontend-name>.localhost;

        location / {
            proxy_pass http://<frontend-name>:<PORT>;
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

### 4. Atualizar nginx depends_on

No `docker-compose.yml`, adicione o novo frontend nas dependências do nginx:

```yaml
nginx:
  depends_on:
    - gateway
    - webapp
    - website
    - <frontend-name>  # Adicionar aqui
```

## Exemplo Completo: Organizations Service

### Dockerfile.dev
```dockerfile
FROM golang:1.24-alpine

RUN apk add --no-cache git
RUN go install github.com/air-verse/air@latest

ENV GOPRIVATE=github.com/primeflow-solutions/*

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

EXPOSE 8083

CMD ["air", "-c", ".air.toml"]
```

### docker-compose.yml
```yaml
  organizations:
    build:
      context: .
      dockerfile: go-organizations-service/Dockerfile.dev
    container_name: primeflow-organizations
    environment:
      - PORT=8083
      - DB_HOST=trolley.proxy.rlwy.net
      - DB_PORT=11306
      - DB_USERNAME=postgres
      - DB_PASSWORD=clBcJPLpujzLWqRfXejgqvClSdJWefYn
      - JWT_SIGNING_KEY=your-secret-key
    volumes:
      - ./go-organizations-service:/app
    networks:
      - primeflow-network
    restart: unless-stopped
```

### Gateway environment
```yaml
gateway:
  environment:
    - ORGANIZATIONS_SERVICE_URL=http://organizations:8083
```

## Checklist

- [ ] Criar Dockerfile.dev no serviço
- [ ] Criar .air.toml (se Go)
- [ ] Adicionar serviço ao docker-compose.yml
- [ ] Configurar variáveis de ambiente
- [ ] Adicionar volumes para hot reload
- [ ] Atualizar gateway (se necessário)
- [ ] Atualizar nginx.conf (se acesso direto)
- [ ] Atualizar nginx depends_on (se frontend)
- [ ] Testar: `docker-compose up -d --build <service-name>`
- [ ] Verificar logs: `docker-compose logs -f <service-name>`
- [ ] Testar acesso via URL configurada

## Portas Padrão

- **8080**: Gateway
- **8081**: Users Service
- **8082**: Tenant Service
- **8083**: Organizations Service
- **8084**: Próximo serviço backend
- **4200**: Frontends Angular (interno)
- **3000**: Frontends React (interno)

## Troubleshooting

### Serviço não compila
```bash
# Entrar no container
docker-compose exec <service-name> sh

# Verificar dependências
go mod tidy
npm install
```

### Hot reload não funciona
- Verificar se o volume está mapeado corretamente
- Verificar se .air.toml está configurado
- Verificar se --poll está habilitado (frontends)

### Porta em conflito
- Mudar a porta no Dockerfile.dev
- Mudar a porta no docker-compose.yml
- Atualizar referências no gateway/nginx

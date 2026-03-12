# Local Development Setup

## Prerequisites

### Required Software
- **Docker Desktop**: Latest version
- **Docker Compose**: v2.0+
- **Git**: For version control
- **Code Editor**: VS Code recommended

### Optional Tools
- **Go**: 1.24+ (for local Go development)
- **Node.js**: 18+ (for local Angular development)
- **PostgreSQL Client**: For database inspection

## Quick Start (5 minutes)

### 1. Clone the Repository

```bash
git clone <repository-url>
cd PrimeFlowSolutions
```

### 2. Start All Services

```bash
docker-compose up --build
```

This will start:
- Nginx (reverse proxy)
- Gateway Service
- Users Service
- Tenant Service
- WebApp Frontend
- Website Frontend

### 3. Access the Applications

- **Website**: http://localhost
- **Web App**: http://app.localhost
- **API Gateway**: http://api.localhost

### 4. Create Your First Tenant

1. Go to http://localhost (website)
2. Click "Teste grátis" or "Começar Agora"
3. Fill in the signup form:
   - Company Name: Your Company
   - Full Name: Your Name
   - Email: admin@yourcompany.com
   - Password: (min 8 chars with uppercase, lowercase, numbers)
4. Click "Criar Conta"
5. You'll be automatically logged in and redirected to the dashboard

## Detailed Setup

### Environment Variables

Each service has an `.env.example` file. Copy and customize as needed:

```bash
# Gateway Service
cd go-gateway-service
cp .env.example .env

# Users Service
cd go-users-service
cp .env.example .env

# Tenant Service
cd go-tenant-service
cp .env.example .env
```

### Database Configuration

The project uses Railway PostgreSQL. Connection details are in `docker-compose.yml`:

```yaml
DB_HOST: trolley.proxy.rlwy.net
DB_PORT: 11306
DB_USERNAME: postgres
DB_PASSWORD: clBcJPLpujzLWqRfXejgqvClSdJWefYn
```

**Note**: These are development credentials. Change for production!

### Docker Compose Configuration

The `docker-compose.yml` defines all services:

```yaml
services:
  nginx:       # Reverse proxy (port 80)
  gateway:     # API Gateway (internal port 8080)
  users:       # Users Service (internal port 8081)
  tenant:      # Tenant Service (internal port 8082)
  webapp:      # Web App Frontend (internal port 4200)
  website:     # Website Frontend (internal port 4200)
```

### Nginx Configuration

Nginx routes requests based on subdomain:

- `localhost` → Website Frontend
- `app.localhost` → WebApp Frontend
- `api.localhost` → Gateway Service

Configuration file: `nginx.conf`

## Development Workflow

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
```

### Viewing Logs

```bash
# All services
docker-compose logs

# Specific service
docker-compose logs gateway

# Follow logs
docker-compose logs -f users

# Last 50 lines
docker-compose logs --tail=50 tenant
```

### Rebuilding Services

```bash
# Rebuild all
docker-compose build

# Rebuild specific service
docker-compose build gateway

# Rebuild without cache
docker-compose build --no-cache
```

## Local Development (Without Docker)

### Backend Services (Go)

Each Go service can run locally:

```bash
# Gateway Service
cd go-gateway-service
go mod download
go run cmd/gateway/main.go

# Users Service
cd go-users-service
go mod download
go run cmd/usuario/main.go

# Tenant Service
cd go-tenant-service
go mod download
go run cmd/tenant/main.go
```

### Frontend Applications (Angular)

```bash
# WebApp
cd ts-webapp-frontend
npm install
npm start
# Access at http://localhost:4200

# Website
cd ts-website-frontend
npm install
npm start
# Access at http://localhost:4201
```

## Troubleshooting

### Port Already in Use

```bash
# Check what's using port 80
lsof -i :80

# Kill process
kill -9 <PID>
```

### Docker Build Fails

```bash
# Clean Docker cache
docker system prune -a

# Remove all containers and images
docker-compose down
docker system prune -a --volumes
```

### Database Connection Issues

1. Check Railway database is accessible
2. Verify credentials in docker-compose.yml
3. Check firewall/network settings

### Frontend Not Loading

```bash
# Check if container is running
docker-compose ps

# Check logs
docker-compose logs webapp

# Rebuild frontend
docker-compose build webapp
docker-compose up webapp
```

### CORS Errors

1. Check nginx.conf has CORS headers
2. Verify API URL in frontend environment files
3. Restart nginx: `docker-compose restart nginx`

## Hot Reload

### Backend (Go)
- Changes require rebuild: `docker-compose up --build <service>`
- Or use Air for hot reload (not configured by default)

### Frontend (Angular)
- Hot reload is enabled by default
- Changes reflect automatically in browser
- Volume mounted: `./ts-webapp-frontend:/app`

## Database Migrations

### Automatic Migrations

Migrations run automatically on service startup:
- **Tenant Service**: Runs shared database migrations
- **Tenant Service**: Runs tenant database migrations on signup

### Manual Migration Check

```bash
# Connect to database
docker exec -it primeflow-tenant sh

# Check migrations table
# (requires psql client in container)
```

## Testing the Setup

### 1. Health Checks

```bash
# Gateway
curl http://api.localhost/v1/health

# Users Service
curl http://api.localhost/users/v1/health

# Tenant Service
curl http://api.localhost/tenant/v1/health
```

### 2. Signup Flow

```bash
curl -X POST http://api.localhost/tenant/v1/signup \
  -H "Content-Type: application/json" \
  -d '{
    "company_name": "Test Company",
    "email": "admin@test.com",
    "domain": "test.com",
    "admin_name": "Admin User",
    "password": "Test1234"
  }'
```

### 3. Login Flow

```bash
curl -X POST http://api.localhost/users/v1/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@test.com",
    "password": "Test1234"
  }'
```

## Development Tips

### 1. Use Docker Logs

Always check logs when debugging:
```bash
docker-compose logs -f <service-name>
```

### 2. Inspect Containers

```bash
# Enter container shell
docker exec -it primeflow-gateway sh

# Check environment variables
docker exec primeflow-gateway env
```

### 3. Database Inspection

Use a PostgreSQL client to inspect databases:
- Host: trolley.proxy.rlwy.net
- Port: 11306
- User: postgres
- Password: clBcJPLpujzLWqRfXejgqvClSdJWefYn

### 4. Network Debugging

```bash
# Check container network
docker network ls
docker network inspect primeflowsolutions_primeflow-network
```

## Next Steps

After setup:
1. Read [API Documentation](03-api-documentation.md)
2. Understand [Database Schema](04-database-schema.md)
3. Learn [Frontend Structure](05-frontend-structure.md)
4. Review [Backend Services](06-backend-services.md)

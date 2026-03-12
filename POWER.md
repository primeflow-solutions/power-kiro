---
name: PrimeFlow Documentation
description: Complete documentation for PrimeFlow multi-tenant SaaS platform - architecture, APIs, database schema, deployment guides, and troubleshooting
version: 1.0.0
author: PrimeFlow Solutions Team
keywords:
  - primeflow
  - multi-tenant
  - saas
  - microservices
  - go
  - golang
  - angular
  - docker
  - postgresql
  - jwt
  - authentication
  - api-gateway
  - tenant-management
  - user-management
  - database-migrations
  - architecture
  - documentation
  - setup
  - deployment
  - troubleshooting
---

# 📚 PrimeFlow Solutions Documentation Power

> Your complete guide to the PrimeFlow multi-tenant SaaS platform

## What This Power Provides

This Power is your **single source of truth** for everything PrimeFlow - from architecture decisions to API contracts, from local setup to production deployment.

### 🎯 Core Documentation

**Architecture & Design**
- Multi-tenant isolation strategy (physical + logical)
- Microservices architecture with Clean Architecture patterns
- JWT authentication flow and gateway routing
- Database schema design and migration strategy

**Backend Services (Go 1.22+)**
- `go-gateway-service` - API Gateway with JWT validation
- `go-users-service` - User authentication and management
- `go-tenant-service` - Tenant provisioning and database creation
- `go-pkg-lib` - Shared library (database, middleware, JWT, migrations)

**Frontend Applications (Angular 18+)**
- `ts-webapp-frontend` - Multi-tenant web app (PrimeNG Freya)
- `ts-website-frontend` - Marketing landing page

**Infrastructure & DevOps**
- Docker Compose local development setup
- Nginx reverse proxy with subdomain routing
- PostgreSQL multi-tenant database configuration
- Railway production deployment

### 📖 Documentation Files

| File | What You'll Learn |
|------|-------------------|
| `01-architecture-overview.md` | System design, multi-tenant strategy, technology stack |
| `02-local-development-setup.md` | Getting started, Docker setup, first tenant creation |
| `03-api-documentation.md` | All API endpoints with request/response examples |
| `04-database-schema.md` | Complete schema, migrations, multi-tenant structure |
| `05-frontend-structure.md` | Angular apps, components, routing, services |
| `06-backend-services.md` | Go services deep dive, Clean Architecture implementation |
| `07-docker-configuration.md` | Docker Compose, networking, volumes, environment vars |
| `08-troubleshooting-guide.md` | Common issues, debugging tips, solutions |
| `09-deployment-production.md` | Production deployment, security, monitoring, scaling |

## When to Use This Power

### 🚀 Getting Started
- **New to the project?** → Start with `02-local-development-setup.md`
- **Want to understand the architecture?** → Read `01-architecture-overview.md`
- **Need to call an API?** → Check `03-api-documentation.md`

### 🔧 Development
- **Building a new feature?** → Reference `06-backend-services.md` or `05-frontend-structure.md`
- **Working with the database?** → See `04-database-schema.md`
- **Debugging an issue?** → Try `08-troubleshooting-guide.md`

### 🚢 Deployment
- **Setting up Docker?** → Follow `07-docker-configuration.md`
- **Deploying to production?** → Use `09-deployment-production.md`

## Quick Reference

### Multi-Tenant Architecture
```
Shared DB (tenant metadata) → Tenant DB (empresa_a) → Service Schemas (users, tenant)
                            → Tenant DB (empresa_b) → Service Schemas (users, tenant)
```

### Authentication Flow
```
Client → Gateway (validate JWT) → Service (X-Tenant-ID + X-User-ID headers)
```

### API URL Structure
```
/{service}/{version}/{endpoint}
Example: /users/v1/users
```

### Local URLs
- API Gateway: `http://api.localhost`
- Web App: `http://app.localhost`
- Website: `http://localhost`

## Technology Stack

**Backend**: Go 1.22+, PostgreSQL, pgx/v5, JWT, Clean Architecture  
**Frontend**: Angular 18+, PrimeNG 17+, RxJS, TypeScript  
**Infrastructure**: Docker, Nginx, Railway

## Need Help?

1. Check the relevant documentation file above
2. Review `08-troubleshooting-guide.md` for common issues
3. Verify Docker services: `docker-compose ps`
4. Check logs: `docker-compose logs -f [service]`

---

**See README.md for detailed documentation structure and usage guide**

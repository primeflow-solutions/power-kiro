# 📚 PrimeFlow Solutions Documentation Power

> Comprehensive documentation for the PrimeFlow multi-tenant SaaS platform

## Overview

This Power provides complete, centralized documentation for the entire PrimeFlow Solutions ecosystem - a multi-tenant B2B SaaS platform built with Go microservices and Angular frontends.

## What's Inside

### 🏗️ Architecture & Design
- **Multi-tenant Strategy**: Physical database isolation per tenant with logical schema separation per service
- **Microservices Architecture**: Independent Go services with Clean Architecture
- **Authentication Flow**: JWT-based auth with gateway validation and header injection
- **API Gateway**: Dynamic routing with tenant context management

### 🔧 Backend Services (Go 1.22+)

| Service | Purpose | Key Features |
|---------|---------|--------------|
| **go-gateway-service** | API Gateway | JWT validation, dynamic routing, tenant context injection |
| **go-users-service** | User Management | Authentication, CRUD operations, multi-tenant user isolation |
| **go-tenant-service** | Tenant Management | Tenant provisioning, database creation, domain management |
| **go-pkg-lib** | Shared Library | Database manager, middleware, JWT, migrations, connection pooling |

### 🎨 Frontend Applications (Angular 18+)

| Application | Purpose | Template |
|-------------|---------|----------|
| **ts-webapp-frontend** | Multi-tenant Web App | PrimeNG Freya |
| **ts-website-frontend** | Marketing Landing Page | Custom |

### 🐳 Infrastructure

- **Docker Compose**: Complete local development environment
- **Nginx**: Reverse proxy with subdomain routing (`api.localhost`, `app.localhost`)
- **PostgreSQL**: Shared database + per-tenant databases
- **Railway**: Production deployment configuration

## Documentation Structure

```
📁 power-kiro/
├── 📄 POWER.md                          # Power metadata and keywords
├── 📄 README.md                         # This file
├── 📄 01-architecture-overview.md       # System architecture and design decisions
├── 📄 02-local-development-setup.md     # Getting started guide
├── 📄 03-api-documentation.md           # All API endpoints with examples
├── 📄 04-database-schema.md             # Complete database schema and migrations
├── 📄 05-frontend-structure.md          # Angular app structure and components
├── 📄 06-backend-services.md            # Go services deep dive
├── 📄 07-docker-configuration.md        # Docker setup and networking
├── 📄 08-troubleshooting-guide.md       # Common issues and solutions
└── 📄 09-deployment-production.md       # Production deployment guide
```

## Quick Start

### For New Developers
1. Start with `02-local-development-setup.md` to get your environment running
2. Read `01-architecture-overview.md` to understand the system design
3. Reference `03-api-documentation.md` when working with APIs

### For Backend Development
- `06-backend-services.md` - Service implementation details
- `04-database-schema.md` - Database structure and migrations
- `07-docker-configuration.md` - Local environment setup

### For Frontend Development
- `05-frontend-structure.md` - Angular app architecture
- `03-api-documentation.md` - API contracts and usage

### For DevOps/Deployment
- `07-docker-configuration.md` - Container setup
- `09-deployment-production.md` - Production deployment
- `08-troubleshooting-guide.md` - Common issues

## Key Concepts

### Multi-Tenant Isolation
- **Physical Isolation**: Each tenant has a dedicated PostgreSQL database
- **Logical Isolation**: Each service has its own schema within tenant databases
- **Shared Database**: Global tenant metadata and email domain mappings

### Authentication Flow
```
Client → Gateway (JWT validation) → Service (X-Tenant-ID + X-User-ID headers)
```

### Database Naming
- Shared: `shared` database
- Tenant: `{tenant_slug}` database (e.g., `empresa_a`)
- Service Schema: `{service_name}` schema (e.g., `users`, `tenant`)

### API Structure
```
/{service}/{version}/{endpoint}
Example: /users/v1/users
```

## When to Use This Power

✅ **Use when you need to:**
- Set up the development environment
- Understand the multi-tenant architecture
- Find API endpoint documentation
- Debug tenant isolation issues
- Learn the database schema
- Configure Docker services
- Deploy to production
- Troubleshoot common problems

## Technology Stack

### Backend
- **Language**: Go 1.22+
- **Router**: Native `net/http` with `http.ServeMux`
- **Database**: PostgreSQL with `pgx/v5`
- **Auth**: JWT with `golang-jwt/jwt/v5`
- **Migrations**: Custom migration engine

### Frontend
- **Framework**: Angular 18+
- **UI Library**: PrimeNG 17+ (Freya template)
- **State Management**: RxJS + Services
- **HTTP**: Angular HttpClient with interceptors

### Infrastructure
- **Containers**: Docker + Docker Compose
- **Reverse Proxy**: Nginx
- **Database**: PostgreSQL 16+
- **Deployment**: Railway (production)

## Contributing

When updating documentation:
1. Keep examples practical and tested
2. Update all related sections when making changes
3. Maintain consistent formatting
4. Include code examples where helpful
5. Update this README if adding new documentation files

## Support

For questions or issues:
- Check `08-troubleshooting-guide.md` first
- Review relevant documentation sections
- Check Docker logs: `docker-compose logs -f [service]`
- Verify database connections and migrations

---

**Last Updated**: March 2026  
**Version**: 1.0.0  
**Maintained by**: PrimeFlow Solutions Team

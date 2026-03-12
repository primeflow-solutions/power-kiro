# Architecture Overview

## System Architecture

PrimeFlow Solutions is a multi-tenant SaaS platform built with a microservices architecture.

```
┌─────────────────────────────────────────────────────────────┐
│                         Internet                             │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │   Nginx (Port 80)    │
              │  Reverse Proxy       │
              └──────────┬───────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
   localhost      api.localhost    app.localhost
   (Website)      (API Gateway)    (Web App)
         │               │               │
         ▼               ▼               ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Website    │  │   Gateway   │  │   WebApp    │
│  Frontend   │  │   Service   │  │  Frontend   │
│  (Angular)  │  │    (Go)     │  │  (Angular)  │
└─────────────┘  └──────┬──────┘  └─────────────┘
                        │
                        │ JWT Validation
                        │ Tenant Routing
                        │
         ┌──────────────┼──────────────┐
         │                             │
         ▼                             ▼
┌─────────────────┐          ┌─────────────────┐
│  Users Service  │          │ Tenant Service  │
│      (Go)       │          │      (Go)       │
└────────┬────────┘          └────────┬────────┘
         │                            │
         └────────────┬───────────────┘
                      │
                      ▼
         ┌────────────────────────┐
         │   PostgreSQL (Railway) │
         │                        │
         │  ┌──────────────────┐  │
         │  │  shared database │  │
         │  │  - tenant schema │  │
         │  └──────────────────┘  │
         │                        │
         │  ┌──────────────────┐  │
         │  │ tenant databases │  │
         │  │ - users schema   │  │
         │  └──────────────────┘  │
         └────────────────────────┘
```

## Multi-Tenant Strategy

### Database-per-Tenant Approach

Each tenant gets its own PostgreSQL database:
- **Shared Database**: Stores tenant metadata, email domains
- **Tenant Databases**: One database per tenant with user data

### Tenant Identification Flow

1. User enters email on login
2. Extract domain from email (e.g., `user@company.com` → `company.com`)
3. Lookup tenant in shared database by email domain
4. Connect to tenant's database
5. Authenticate user in tenant database
6. Generate JWT with `tenant_id` and `tenant_slug`

### Request Flow with Multi-Tenancy

```
1. Frontend → API Gateway
   Headers: Authorization: Bearer <JWT>

2. API Gateway validates JWT
   - Extracts: user_id, tenant_id, tenant_slug
   - Injects headers: X-Tenant-ID, X-Tenant-Slug, X-User-ID

3. Backend Service receives request
   - Middleware reads X-Tenant-Slug
   - Connects to tenant database
   - Executes operation in tenant context

4. Response flows back to frontend
```

## Technology Stack

### Backend
- **Language**: Go 1.24
- **Framework**: Standard library (net/http)
- **ORM**: GORM
- **Database**: PostgreSQL
- **Authentication**: JWT (golang-jwt/jwt)
- **Password**: bcrypt

### Frontend
- **Framework**: Angular 19
- **Template**: Freya (PrimeNG)
- **UI Library**: PrimeNG
- **State Management**: Signals
- **HTTP Client**: HttpClient with Interceptors

### Infrastructure
- **Reverse Proxy**: Nginx
- **Container**: Docker & Docker Compose
- **Database Host**: Railway (PostgreSQL)
- **Local Development**: Docker Compose

## Service Responsibilities

### Gateway Service
- JWT validation
- Tenant context injection (X-Tenant-Slug header)
- Service routing and proxying
- CORS handling
- Public route management

### Users Service
- User CRUD operations
- Login authentication
- Password management
- Per-request tenant database connections

### Tenant Service
- Tenant CRUD operations
- Self-service signup
- Database provisioning
- Email domain management
- Admin user creation

### Shared Library (go-pkg-lib)
- Database manager with connection pooling
- JWT token generation and validation
- Migration engine
- Middleware (public/private routes)
- Structured logging

## Security Model

### Authentication
- JWT tokens with 7-day expiration
- Tokens contain: user_id, tenant_id, tenant_slug
- Bcrypt password hashing (cost factor 12)

### Authorization
- Gateway validates JWT on all private routes
- Tenant isolation via database separation
- X-Tenant-Slug header prevents cross-tenant access

### Multi-Tenant Isolation
- Physical database separation per tenant
- Connection pooling per tenant database
- Middleware enforces tenant context
- No shared data between tenants

## Scalability Considerations

### Connection Pooling
- Max 100 open connections per database
- Max 10 idle connections per database
- 1-hour connection lifetime
- 10-minute idle connection timeout

### Horizontal Scaling
- Stateless services (can scale horizontally)
- JWT tokens (no session storage needed)
- Database connection pooling per instance

### Performance Optimizations
- Connection reuse via pooling
- Lazy tenant database connections
- Efficient JWT validation
- Nginx caching for static assets

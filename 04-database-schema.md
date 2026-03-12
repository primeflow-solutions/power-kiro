# Database Schema

## Overview

PrimeFlow uses a **database-per-tenant** strategy with PostgreSQL:
- **1 Shared Database**: Stores tenant metadata
- **N Tenant Databases**: One per tenant, stores user data

## Shared Database (`shared`)

### Schema: `tenant`

#### Table: `tenants`

Stores tenant information.

```sql
CREATE TABLE tenant.tenants (
    id SERIAL PRIMARY KEY,
    slug VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    database_name VARCHAR(255) UNIQUE NOT NULL,
    active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tenants_slug ON tenant.tenants(slug);
CREATE INDEX idx_tenants_active ON tenant.tenants(active);
```

**Columns:**
- `id`: Primary key
- `slug`: URL-friendly identifier (e.g., `acme_corp`)
- `name`: Display name (e.g., `Acme Corporation`)
- `database_name`: PostgreSQL database name (e.g., `acme_corp`)
- `active`: Whether tenant is active
- `created_at`: Creation timestamp
- `updated_at`: Last update timestamp

**Example Data:**
```sql
INSERT INTO tenant.tenants (slug, name, database_name, active)
VALUES ('primeflow_solutions', 'PrimeFlow Solutions', 'primeflow_solutions', true);
```

---

#### Table: `email_domains`

Maps email domains to tenants for login identification.

```sql
CREATE TABLE tenant.email_domains (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL REFERENCES tenant.tenants(id) ON DELETE CASCADE,
    domain VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE(domain)
);

CREATE INDEX idx_email_domains_tenant_id ON tenant.email_domains(tenant_id);
CREATE INDEX idx_email_domains_domain ON tenant.email_domains(domain);
```

**Columns:**
- `id`: Primary key
- `tenant_id`: Foreign key to tenants table
- `domain`: Email domain (e.g., `acme.com`)
- `created_at`: Creation timestamp

**Example Data:**
```sql
INSERT INTO tenant.email_domains (tenant_id, domain)
VALUES (1, 'primeflow.solutions');
```

**Usage:**
When a user logs in with `admin@acme.com`:
1. Extract domain: `acme.com`
2. Query: `SELECT tenant_id FROM tenant.email_domains WHERE domain = 'acme.com'`
3. Get tenant info and connect to tenant database

---

#### Table: `migrations`

Tracks applied migrations for the shared database.

```sql
CREATE TABLE tenant.migrations (
    id SERIAL PRIMARY KEY,
    version VARCHAR(50) UNIQUE NOT NULL,
    applied_at BIGINT
);

CREATE UNIQUE INDEX idx_migrations_version ON tenant.migrations(version);
```

---

## Tenant Databases (e.g., `primeflow_solutions`)

Each tenant gets their own database with this schema:

### Schema: `users`

#### Table: `users`

Stores user information for the tenant.

```sql
CREATE SCHEMA IF NOT EXISTS users;

CREATE TABLE users.users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(255) NOT NULL,
    last_name VARCHAR(255) NOT NULL,
    active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users.users(email);
CREATE INDEX idx_users_active ON users.users(active);
```

**Columns:**
- `id`: Primary key
- `email`: User email (unique within tenant)
- `password_hash`: Bcrypt hashed password (cost factor 12)
- `first_name`: User's first name
- `last_name`: User's last name
- `active`: Whether user account is active
- `created_at`: Creation timestamp
- `updated_at`: Last update timestamp

**Example Data:**
```sql
INSERT INTO users.users (email, password_hash, first_name, last_name, active)
VALUES (
    'admin@primeflow.solutions',
    '$2a$12$...',  -- bcrypt hash
    'Admin',
    'User',
    true
);
```

---

#### Table: `migrations`

Tracks applied migrations for the tenant database.

```sql
CREATE TABLE users.migrations (
    id SERIAL PRIMARY KEY,
    version VARCHAR(50) UNIQUE NOT NULL,
    applied_at BIGINT
);

CREATE UNIQUE INDEX idx_migrations_version ON users.migrations(version);
```

---

## Database Relationships

```
┌─────────────────────────────────────────┐
│         Shared Database (shared)        │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │  tenant.tenants                   │  │
│  │  - id (PK)                        │  │
│  │  - slug                           │  │
│  │  - name                           │  │
│  │  - database_name                  │  │
│  │  - active                         │  │
│  └───────────┬───────────────────────┘  │
│              │                           │
│              │ 1:N                       │
│              ▼                           │
│  ┌───────────────────────────────────┐  │
│  │  tenant.email_domains             │  │
│  │  - id (PK)                        │  │
│  │  - tenant_id (FK)                 │  │
│  │  - domain (UNIQUE)                │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│   Tenant Database (primeflow_solutions) │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │  users.users                      │  │
│  │  - id (PK)                        │  │
│  │  - email (UNIQUE)                 │  │
│  │  - password_hash                  │  │
│  │  - first_name                     │  │
│  │  - last_name                      │  │
│  │  - active                         │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

---

## Connection Pooling Configuration

Each database connection uses these pool settings:

```go
sqlDB.SetMaxIdleConns(10)                    // Max idle connections
sqlDB.SetMaxOpenConns(100)                   // Max open connections
sqlDB.SetConnMaxLifetime(time.Hour)          // Connection lifetime: 1 hour
sqlDB.SetConnMaxIdleTime(10 * time.Minute)   // Idle timeout: 10 minutes
```

**Per Service:**
- Shared DB: 1 connection pool
- Each Tenant DB: 1 connection pool (created on first access)

---

## Migration System

### Migration Files

Migrations are SQL files with naming convention:
```
{version}_{description}.{up|down}.sql
```

**Example:**
```
001_create_tenants.up.sql
001_create_tenants.down.sql
002_create_email_domains.up.sql
002_create_email_domains.down.sql
```

### Migration Locations

**Shared Database Migrations:**
```
go-tenant-service/migrations/shared/
├── 001_create_tenants.up.sql
├── 001_create_tenants.down.sql
├── 002_create_email_domains.up.sql
└── 002_create_email_domains.down.sql
```

**Tenant Database Migrations:**
```
go-tenant-service/migrations/tenant/
├── 001_create_users.up.sql
└── 001_create_users.down.sql
```

### Migration Execution

**Shared Database:**
- Runs on tenant service startup
- Executed by: `go-tenant-service/cmd/tenant/main.go`

**Tenant Database:**
- Runs during tenant provisioning (signup)
- Executed by: `DatabaseProvisioner.runMigrations()`

### Migration Tracking

Migrations are tracked in the `migrations` table:

```sql
SELECT * FROM tenant.migrations;
-- version | applied_at
-- 001     | 1234567890
-- 002     | 1234567891
```

---

## Database Provisioning Flow

When a new tenant signs up:

1. **Create Tenant Record** (shared database)
   ```sql
   INSERT INTO tenant.tenants (slug, name, database_name)
   VALUES ('acme_corp', 'Acme Corp', 'acme_corp');
   ```

2. **Create Email Domain** (shared database)
   ```sql
   INSERT INTO tenant.email_domains (tenant_id, domain)
   VALUES (1, 'acme.com');
   ```

3. **Create Physical Database**
   ```sql
   CREATE DATABASE acme_corp WITH TEMPLATE template0 ENCODING 'UTF8';
   ```

4. **Run Tenant Migrations**
   - Connect to `acme_corp` database
   - Execute migrations from `migrations/tenant/`
   - Creates `users` schema and `users.users` table

5. **Create Admin User**
   ```sql
   INSERT INTO users.users (email, password_hash, first_name, last_name)
   VALUES ('admin@acme.com', '$2a$12$...', 'Admin', 'User');
   ```

---

## Query Examples

### Find Tenant by Email Domain

```sql
SELECT t.*
FROM tenant.tenants t
INNER JOIN tenant.email_domains ed ON ed.tenant_id = t.id
WHERE ed.domain = 'acme.com' AND t.active = true;
```

### List All Users in Tenant

```sql
-- Connect to tenant database first
\c acme_corp

SELECT * FROM users.users
ORDER BY created_at DESC;
```

### Count Users per Tenant

```sql
-- This requires connecting to each tenant database
-- No cross-database queries in PostgreSQL
```

### Check Migration Status

```sql
-- Shared database
SELECT * FROM tenant.migrations ORDER BY version;

-- Tenant database
\c acme_corp
SELECT * FROM users.migrations ORDER BY version;
```

---

## Backup Strategy

### Shared Database
- Critical: Contains all tenant metadata
- Backup frequency: Hourly
- Retention: 30 days

### Tenant Databases
- Important: Contains user data
- Backup frequency: Daily
- Retention: 7 days
- Per-tenant restore capability

---

## Security Considerations

### Password Storage
- Bcrypt with cost factor 12
- Never store plain text passwords
- Password validation: min 8 chars, uppercase, lowercase, numbers

### SQL Injection Prevention
- Use parameterized queries (GORM handles this)
- Never concatenate user input into SQL

### Tenant Isolation
- Physical database separation
- No cross-tenant queries possible
- Middleware enforces tenant context

### Connection Security
- Use SSL in production (`sslmode=require`)
- Rotate database credentials regularly
- Limit database user permissions

---

## Performance Optimization

### Indexes
- Email columns (frequent lookups)
- Active status (filtering)
- Foreign keys (joins)
- Slug columns (URL routing)

### Connection Pooling
- Reuse connections
- Limit max connections
- Close idle connections

### Query Optimization
- Use EXPLAIN ANALYZE for slow queries
- Add indexes for frequent queries
- Avoid N+1 queries

---

## Monitoring

### Key Metrics
- Connection pool usage
- Query execution time
- Database size per tenant
- Active connections
- Failed queries

### Slow Query Log
Enable in PostgreSQL:
```sql
ALTER DATABASE shared SET log_min_duration_statement = 200;
```

---

## Database Maintenance

### Vacuum
Run periodically to reclaim space:
```sql
VACUUM ANALYZE tenant.tenants;
VACUUM ANALYZE users.users;
```

### Reindex
Rebuild indexes if needed:
```sql
REINDEX TABLE tenant.tenants;
REINDEX TABLE users.users;
```

### Statistics Update
```sql
ANALYZE tenant.tenants;
ANALYZE users.users;
```

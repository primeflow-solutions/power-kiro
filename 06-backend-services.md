# Backend Services

## Overview

PrimeFlow backend consists of 4 Go services:
1. **go-gateway-service**: API Gateway
2. **go-users-service**: User management
3. **go-tenant-service**: Tenant management
4. **go-pkg-lib**: Shared library

All services use:
- Go 1.24
- GORM for database access
- Standard library for HTTP
- JWT for authentication

---

## Gateway Service (go-gateway-service)

### Purpose
- API Gateway and reverse proxy
- JWT validation
- Tenant context injection
- CORS handling
- Service routing

### Project Structure

```
go-gateway-service/
├── cmd/
│   └── gateway/
│       └── main.go              # Entry point
├── internal/
│   ├── config/
│   │   └── config.go            # Configuration
│   ├── handler/
│   │   └── health.go            # Health check
│   ├── middleware/
│   │   └── jwt.go               # JWT validation
│   └── router/
│       ├── proxy.go             # Reverse proxy
│       └── registry.go          # Service registry
├── Dockerfile.dev               # Docker configuration
├── .env.example                 # Environment template
├── go.mod                       # Go dependencies
└── go.sum                       # Dependency checksums
```

### Key Components

#### JWT Middleware

**Location:** `internal/middleware/jwt.go`

**Responsibilities:**
- Validate JWT tokens
- Extract claims (user_id, tenant_id, tenant_slug)
- Inject headers (X-Tenant-ID, X-Tenant-Slug, X-User-ID)
- Handle public routes

**Flow:**
```go
func (jm *JWTMiddleware) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Check if route is public
        if jm.isPublicRoute(r.URL.Path) {
            next.ServeHTTP(w, r)
            return
        }

        // Extract and validate token
        token := extractToken(r)
        claims, err := jm.tokenGenerator.ValidateToken(token)
        if err != nil {
            respondError(w, http.StatusUnauthorized, "invalid or expired token")
            return
        }

        // Inject headers
        r.Header.Set("X-Tenant-ID", strconv.Itoa(claims.TenantID))
        r.Header.Set("X-Tenant-Slug", claims.TenantSlug)
        r.Header.Set("X-User-ID", strconv.Itoa(userID))

        next.ServeHTTP(w, r)
    })
}
```

#### Service Registry

**Location:** `internal/router/registry.go`

**Registered Services:**
```go
type ServiceRegistry struct {
    services map[string]string
}

func NewServiceRegistry() *ServiceRegistry {
    return &ServiceRegistry{
        services: map[string]string{
            "users":  os.Getenv("USER_SERVICE_URL"),   // http://users:8081
            "tenant": os.Getenv("TENANT_SERVICE_URL"), // http://tenant:8082
        },
    }
}
```

#### Reverse Proxy

**Location:** `internal/router/proxy.go`

**Routing Logic:**
```go
// /users/v1/login → http://users:8081/v1/login
// /tenant/v1/signup → http://tenant:8082/v1/signup
```

### Environment Variables

```bash
PORT=8080
JWT_SIGNING_KEY=your-secret-key-here
JWT_EXPIRATION=168h  # 7 days
USER_SERVICE_URL=http://users:8081
TENANT_SERVICE_URL=http://tenant:8082
PUBLIC_ROUTES=/users/v1/login,/tenant/v1/signup,/v1/health
LOG_LEVEL=info
```

### Public Routes

Routes that don't require authentication:
- `/users/v1/login`
- `/tenant/v1/signup`
- `/v1/health`
- `/tenant/v1/health`
- `/users/v1/health`

---

## Users Service (go-users-service)

### Purpose
- User CRUD operations
- Login authentication
- Password management
- Multi-tenant user isolation

### Project Structure

```
go-users-service/
├── cmd/
│   └── usuario/
│       └── main.go                      # Entry point
├── internal/
│   ├── application/                     # Use cases
│   │   ├── login_usecase.go
│   │   ├── create_user_usecase.go
│   │   ├── list_users_usecase.go
│   │   ├── get_user_usecase.go
│   │   ├── update_user_usecase.go
│   │   └── delete_user_usecase.go
│   ├── domain/                          # Domain models
│   │   ├── user.go
│   │   ├── tenant.go
│   │   └── repository.go
│   ├── infrastructure/
│   │   └── repository/                  # Data access
│   │       ├── user_repository.go
│   │       └── tenant_repository.go
│   └── presentation/
│       └── http/
│           ├── dto/                     # Data transfer objects
│           │   ├── user_dto.go
│           │   └── login_dto.go
│           ├── handler/                 # HTTP handlers
│           │   ├── user_handler.go
│           │   ├── login_handler.go
│           │   └── health_handler.go
│           └── router/
│               └── router.go            # Route setup
├── migrations/
│   └── tenant/                          # Tenant DB migrations
│       ├── 001_create_schema.up.sql
│       └── 002_create_users.up.sql
├── Dockerfile.dev
├── .env.example
├── go.mod
└── go.sum
```

### Key Components

#### User Domain Model

**Location:** `internal/domain/user.go`

```go
type User struct {
    ID           int       `gorm:"primaryKey"`
    Email        string    `gorm:"uniqueIndex;size:255;not null"`
    PasswordHash string    `gorm:"size:255;not null"`
    FirstName    string    `gorm:"size:255;not null"`
    LastName     string    `gorm:"size:255;not null"`
    Active       bool      `gorm:"not null;default:true"`
    CreatedAt    time.Time `gorm:"not null"`
    UpdatedAt    time.Time `gorm:"not null"`
}

func (User) TableName() string {
    return "users.users"
}

func (u *User) SetPassword(plainPassword string) error {
    hash, err := bcrypt.GenerateFromPassword([]byte(plainPassword), 12)
    if err != nil {
        return err
    }
    u.PasswordHash = string(hash)
    return nil
}

func (u *User) CheckPassword(plainPassword string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(u.PasswordHash), []byte(plainPassword))
    return err == nil
}
```

#### Login Use Case

**Location:** `internal/application/login_usecase.go`

**Flow:**
1. Extract domain from email
2. Find tenant by email domain (shared DB)
3. Connect to tenant database
4. Find user by email (tenant DB)
5. Verify password
6. Generate JWT token with tenant_slug

```go
func (uc *LoginUseCase) Execute(ctx context.Context, email, password string) (*LoginResult, error) {
    // Extract domain
    emailDomain, err := extractDomain(email)
    
    // Find tenant
    tenant, err := uc.tenantRepo.FindByEmailDomain(ctx, emailDomain)
    
    // Connect to tenant DB
    tenantDB, err := uc.dbManager.GetTenantDB(tenant.DatabaseName)
    
    // Find user
    userRepo := repository.NewPostgresUserRepository(tenantDB)
    user, err := userRepo.FindByEmail(ctx, email)
    
    // Verify password
    if !user.CheckPassword(password) {
        return nil, domain.ErrInvalidPassword
    }
    
    // Generate token with tenant_slug
    token, err := uc.tokenGen.GenerateToken(user.ID, tenant.ID, tenant.Slug)
    
    return &LoginResult{
        Token:  token,
        User:   user,
        Tenant: tenant,
    }, nil
}
```

#### User Handler (Per-Request Repositories)

**Location:** `internal/presentation/http/handler/user_handler.go`

**Key Pattern:** Creates repositories per request using tenant database

```go
func (h *UserHandler) List(w http.ResponseWriter, r *http.Request, reqCtx *database.RequestContext) {
    // Create repository with tenant database
    userRepo := repository.NewPostgresUserRepository(reqCtx.TenantDB)
    listUseCase := application.NewListUsersUseCase(userRepo)
    
    users, err := listUseCase.Execute(r.Context())
    // ... handle response
}
```

**Why Per-Request?**
- Each request may be for a different tenant
- `reqCtx.TenantDB` is the correct database for the tenant
- Ensures proper multi-tenant isolation

### Environment Variables

```bash
PORT=8081
DB_HOST=trolley.proxy.rlwy.net
DB_PORT=11306
DB_USERNAME=postgres
DB_PASSWORD=your-password
JWT_SIGNING_KEY=your-secret-key
JWT_EXPIRATION=168h
LOG_LEVEL=info
```

---

## Tenant Service (go-tenant-service)

### Purpose
- Tenant CRUD operations
- Self-service signup
- Database provisioning
- Email domain management

### Project Structure

```
go-tenant-service/
├── cmd/
│   └── tenant/
│       └── main.go
├── internal/
│   ├── application/                     # Use cases
│   │   ├── signup_tenant_usecase.go
│   │   ├── create_tenant_usecase.go
│   │   ├── list_tenants_usecase.go
│   │   ├── get_tenant_usecase.go
│   │   ├── update_tenant_usecase.go
│   │   ├── delete_tenant_usecase.go
│   │   ├── add_domain_usecase.go
│   │   ├── list_domains_usecase.go
│   │   └── remove_domain_usecase.go
│   ├── domain/
│   │   ├── tenant.go
│   │   ├── email_domain.go
│   │   └── repository.go
│   ├── infrastructure/
│   │   ├── database/
│   │   │   └── provisioner.go           # Database provisioning
│   │   └── repository/
│   │       ├── tenant_repository.go
│   │       └── domain_repository.go
│   └── presentation/
│       └── http/
│           ├── dto/
│           │   ├── tenant_dto.go
│           │   ├── signup_dto.go
│           │   ├── domain_dto.go
│           │   └── error_dto.go
│           ├── handler/
│           │   ├── tenant_handler.go
│           │   ├── signup_handler.go
│           │   ├── domain_handler.go
│           │   ├── health_handler.go
│           │   └── swagger_handler.go
│           └── router/
│               └── router.go
├── migrations/
│   ├── shared/                          # Shared DB migrations
│   │   ├── 001_create_tenants.up.sql
│   │   └── 002_create_email_domains.up.sql
│   └── tenant/                          # Tenant DB migrations
│       └── 001_create_users.up.sql
├── docs/
│   └── swagger.json                     # API documentation
├── Dockerfile.dev
├── .env.example
├── go.mod
└── go.sum
```

### Key Components

#### Tenant Domain Model

**Location:** `internal/domain/tenant.go`

```go
type Tenant struct {
    ID           int       `gorm:"primaryKey"`
    Slug         string    `gorm:"uniqueIndex;size:255;not null"`
    Name         string    `gorm:"size:255;not null"`
    DatabaseName string    `gorm:"uniqueIndex;size:255;not null"`
    Active       bool      `gorm:"not null;default:true"`
    CreatedAt    time.Time `gorm:"not null"`
    UpdatedAt    time.Time `gorm:"not null"`
}

func (Tenant) TableName() string {
    return "tenant.tenants"
}
```

#### Database Provisioner

**Location:** `internal/infrastructure/database/provisioner.go`

**Responsibilities:**
- Create physical PostgreSQL database
- Run tenant migrations
- Handle rollback on failure

```go
func (dp *DatabaseProvisioner) ProvisionDatabase(ctx context.Context, databaseName string) error {
    // 1. Create database
    if err := dp.createDatabase(ctx, databaseName); err != nil {
        return fmt.Errorf("failed to create database: %w", err)
    }

    // 2. Run migrations
    if err := dp.runMigrations(ctx, databaseName); err != nil {
        // Rollback: drop database
        _ = dp.dropDatabase(ctx, databaseName)
        return fmt.Errorf("failed to run migrations: %w", err)
    }

    return nil
}

func (dp *DatabaseProvisioner) createDatabase(ctx context.Context, databaseName string) error {
    query := fmt.Sprintf("CREATE DATABASE %s WITH TEMPLATE template0 ENCODING 'UTF8'", databaseName)
    _, err := sqlDB.ExecContext(ctx, query)
    return err
}

func (dp *DatabaseProvisioner) runMigrations(ctx context.Context, databaseName string) error {
    tenantDB, err := dp.dbManager.GetTenantDB(databaseName)
    migrationEngine := database.NewMigrationEngine(tenantDB)
    return migrationEngine.RunMigrationsWithSchema("./migrations/tenant", "users")
}
```

#### Signup Use Case

**Location:** `internal/application/signup_tenant_usecase.go`

**Flow:**
1. Validate input
2. Generate slug from company name
3. Create tenant record
4. Create email domain record
5. Provision database (create + migrate)
6. Connect to tenant database
7. Create admin user
8. Generate JWT token

```go
func (uc *SignupTenantUseCase) Execute(ctx context.Context, input SignupTenantInput) (*SignupTenantOutput, error) {
    // 1. Generate slug
    slug := generateSlug(input.CompanyName)
    
    // 2. Create tenant
    tenant := &domain.Tenant{
        Slug:         slug,
        Name:         input.CompanyName,
        DatabaseName: slug,
        Active:       true,
    }
    if err := uc.tenantRepo.Create(ctx, tenant); err != nil {
        return nil, err
    }
    
    // 3. Create email domain
    emailDomain := &domain.EmailDomain{
        TenantID: tenant.ID,
        Domain:   input.Domain,
    }
    if err := uc.domainRepo.Create(ctx, emailDomain); err != nil {
        return nil, err
    }
    
    // 4. Provision database
    if err := uc.provisioner.ProvisionDatabase(ctx, tenant.DatabaseName); err != nil {
        return nil, err
    }
    
    // 5. Create admin user
    tenantDB, err := uc.dbManager.GetTenantDB(tenant.DatabaseName)
    userRepo := repository.NewPostgresUserRepository(tenantDB)
    
    adminUser := &domain.User{
        Email:     input.Email,
        FirstName: extractFirstName(input.AdminName),
        LastName:  extractLastName(input.AdminName),
        Active:    true,
    }
    adminUser.SetPassword(input.Password)
    
    if err := userRepo.Create(ctx, adminUser); err != nil {
        return nil, err
    }
    
    return &SignupTenantOutput{
        TenantID:     tenant.ID,
        TenantName:   tenant.Name,
        TenantSlug:   tenant.Slug,
        DatabaseName: tenant.DatabaseName,
        UserID:       adminUser.ID,
        AdminEmail:   adminUser.Email,
        AdminName:    input.AdminName,
    }, nil
}
```

### Environment Variables

```bash
PORT=8082
DB_HOST=trolley.proxy.rlwy.net
DB_PORT=11306
DB_USERNAME=postgres
DB_PASSWORD=your-password
JWT_SIGNING_KEY=your-secret-key
JWT_EXPIRATION=168h
LOG_LEVEL=info
```

---

## Shared Library (go-pkg-lib)

### Purpose
- Database management
- JWT token handling
- Middleware
- Migrations
- Logging
- Configuration

### Project Structure

```
go-pkg-lib/
├── database/
│   ├── manager.go           # Database manager
│   ├── context.go           # Request context
│   └── migrations.go        # Migration engine
├── jwt/
│   ├── token.go             # Token generation/validation
│   └── claims.go            # Custom claims
├── middleware/
│   ├── handler.go           # Handler wrapper
│   ├── public.go            # Public route middleware
│   └── private.go           # Private route middleware
├── logger/
│   └── logger.go            # Structured logging
├── config/
│   └── config.go            # Configuration utilities
├── go.mod
└── go.sum
```

### Key Components

#### Database Manager

**Location:** `database/manager.go`

**Features:**
- Connection pooling
- Multi-tenant database management
- Lazy connection creation
- Automatic cleanup

```go
type DatabaseManager struct {
    sharedDB    *gorm.DB
    tenantDBs   map[string]*gorm.DB
    mu          sync.RWMutex
    dbHost      string
    dbPort      string
    dbUsername  string
    dbPassword  string
}

func (dm *DatabaseManager) GetTenantDB(tenantSlug string) (*gorm.DB, error) {
    // Check cache
    dm.mu.RLock()
    db, exists := dm.tenantDBs[tenantSlug]
    dm.mu.RUnlock()
    
    if exists {
        return db, nil
    }
    
    // Create new connection
    dm.mu.Lock()
    defer dm.mu.Unlock()
    
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    
    // Configure connection pool
    sqlDB, _ := db.DB()
    sqlDB.SetMaxIdleConns(10)
    sqlDB.SetMaxOpenConns(100)
    sqlDB.SetConnMaxLifetime(time.Hour)
    sqlDB.SetConnMaxIdleTime(10 * time.Minute)
    
    dm.tenantDBs[tenantSlug] = db
    return db, nil
}
```

#### JWT Token Generator

**Location:** `jwt/token.go`

```go
type TokenGenerator struct {
    signingKey []byte
    expiration time.Duration
}

func (tg *TokenGenerator) GenerateToken(userID, tenantID int, tenantSlug string) (string, error) {
    now := time.Now()
    claims := CustomClaims{
        TenantID:   tenantID,
        TenantSlug: tenantSlug,
        RegisteredClaims: jwt.RegisteredClaims{
            Subject:   strconv.Itoa(userID),
            IssuedAt:  jwt.NewNumericDate(now),
            ExpiresAt: jwt.NewNumericDate(now.Add(tg.expiration)),
        },
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(tg.signingKey)
}
```

#### Custom Claims

**Location:** `jwt/claims.go`

```go
type CustomClaims struct {
    TenantID   int    `json:"tenant_id"`
    TenantSlug string `json:"tenant_slug"`
    jwt.RegisteredClaims
}
```

#### Private Route Middleware

**Location:** `middleware/private.go`

**Extracts headers and provides database connections:**

```go
func Wrap(dbManager *database.DatabaseManager) HandlerFunc {
    return func(handler CustomHandler) http.HandlerFunc {
        return func(w http.ResponseWriter, r *http.Request) {
            // Extract headers (injected by gateway)
            tenantID := r.Header.Get("X-Tenant-ID")
            userID := r.Header.Get("X-User-ID")
            tenantSlug := r.Header.Get("X-Tenant-Slug")
            
            // Validate headers
            if tenantSlug == "" {
                respondError(w, http.StatusUnauthorized, "missing X-Tenant-Slug header")
                return
            }
            
            // Load tenant database
            tenantDB, err := dbManager.GetTenantDB(tenantSlug)
            if err != nil {
                respondError(w, http.StatusInternalServerError, "failed to connect to tenant database")
                return
            }
            
            // Create request context
            ctx := &database.RequestContext{
                TenantID: tenantID,
                UserID:   userID,
                TenantDB: tenantDB,
                SharedDB: dbManager.GetSharedDB(),
            }
            
            // Call handler with context
            handler(w, r, ctx)
        }
    }
}
```

#### Migration Engine

**Location:** `database/migrations.go`

**Features:**
- SQL file-based migrations
- Schema support
- Version tracking
- Rollback support

```go
func (me *MigrationEngine) RunMigrationsWithSchema(migrationsDir string, schemaName string) error {
    // Create schema
    me.db.Exec(fmt.Sprintf("CREATE SCHEMA IF NOT EXISTS %s", schemaName))
    
    // Create migrations table
    me.db.Table(schemaName + ".migrations").AutoMigrate(&Migration{})
    
    // Get applied migrations
    var applied []Migration
    me.db.Table(schemaName + ".migrations").Find(&applied)
    
    // Read migration files
    files, _ := os.ReadDir(migrationsDir)
    
    // Apply pending migrations
    for _, file := range files {
        if !strings.HasSuffix(file.Name(), ".up.sql") {
            continue
        }
        
        version := extractVersion(file.Name())
        if isApplied(version, applied) {
            continue
        }
        
        // Read and execute SQL
        sql, _ := os.ReadFile(filepath.Join(migrationsDir, file.Name()))
        me.db.Exec(string(sql))
        
        // Record migration
        me.db.Table(schemaName + ".migrations").Create(&Migration{
            Version:   version,
            AppliedAt: time.Now().Unix(),
        })
    }
    
    return nil
}
```

---

## Service Communication

### Internal Communication

Services communicate via HTTP:
- Gateway → Users: `http://users:8081`
- Gateway → Tenant: `http://tenant:8082`

### External Communication

Clients communicate via Gateway:
- Client → Gateway: `http://api.localhost`
- Gateway → Service: Internal routing

---

## Error Handling

### Standard Error Response

```go
type ErrorResponse struct {
    Error   string `json:"error"`
    Message string `json:"message"`
}

func respondError(w http.ResponseWriter, status int, message string, err error) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(ErrorResponse{
        Error:   http.StatusText(status),
        Message: err.Error(),
    })
}
```

### Error Types

```go
var (
    ErrInvalidEmail     = errors.New("invalid email format")
    ErrNameTooShort     = errors.New("name must be at least 2 characters")
    ErrPasswordTooShort = errors.New("password must be at least 8 characters")
    ErrUserNotFound     = errors.New("user not found")
    ErrInvalidPassword  = errors.New("invalid password")
)
```

---

## Logging

### Structured Logging

```go
logger.Info("user created", "user_id", user.ID, "email", user.Email)
logger.Error("failed to create user", "error", err)
logger.Debug("processing request", "path", r.URL.Path)
logger.Warn("slow query detected", "duration", duration)
```

### Log Levels

- `DEBUG`: Detailed information for debugging
- `INFO`: General information
- `WARN`: Warning messages
- `ERROR`: Error messages

---

## Testing

### Unit Tests

```bash
cd go-users-service
go test ./...
```

### Integration Tests

```bash
go test -tags=integration ./...
```

### Test Coverage

```bash
go test -cover ./...
```

---

## Build & Deployment

### Local Build

```bash
cd go-users-service
go build -o users ./cmd/usuario
./users
```

### Docker Build

```bash
docker build -f Dockerfile.dev -t users-service .
docker run -p 8081:8081 users-service
```

### Production Build

```bash
CGO_ENABLED=0 GOOS=linux go build -o users ./cmd/usuario
```

---

## Performance Considerations

### Connection Pooling
- Reuse database connections
- Configure pool limits
- Monitor pool usage

### Caching
- Cache tenant lookups
- Cache JWT validation results
- Use Redis for distributed cache

### Concurrency
- Use goroutines for parallel operations
- Implement rate limiting
- Use context for cancellation

### Database Optimization
- Use indexes
- Optimize queries
- Monitor slow queries
- Use prepared statements

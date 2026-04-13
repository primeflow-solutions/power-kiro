# Go - PKG Library (primeflow.solutions/pkg)

## 🎯 Visão Geral

O `go-pkg-lib` é uma biblioteca compartilhada que fornece funcionalidades comuns para todos os serviços Go da PrimeFlow Solutions. Esta biblioteca centraliza código reutilizável e garante consistência entre serviços.

**IMPORTANTE**: NUNCA recrie funcionalidades que já existem no PKG. Sempre use o que está disponível.

## 📦 Pacotes Disponíveis

### 1. **config** - Gerenciamento de Variáveis de Ambiente
### 2. **database** - Gerenciamento de Conexões Multi-Tenant
### 3. **jwt** - Geração e Validação de Tokens JWT
### 4. **logger** - Logs Estruturados (slog)
### 5. **middleware** - Middlewares HTTP
### 6. **pagination** - Paginação, Filtros e Ordenação
### 7. **repository** - Repository Pattern com Generics
### 8. **serviceclient** - Cliente HTTP para Comunicação entre Serviços
### 9. **validator** - Validação de Structs

---

## 1️⃣ config - Gerenciamento de Variáveis de Ambiente

### **Funções Disponíveis**

```go
import "primeflow.solutions/pkg/config"

// GetEnv retorna valor da env var ou default
value := config.GetEnv("PORT", "8080")

// GetEnvRequired retorna valor ou erro se não existir
dbHost, err := config.GetEnvRequired("DB_HOST")

// GetEnvInt retorna env var como inteiro
port := config.GetEnvInt("PORT", 8080)

// GetEnvDuration retorna env var como duration
timeout := config.GetEnvDuration("TIMEOUT", 30*time.Second)

// ValidateRequired valida múltiplas env vars obrigatórias
err := config.ValidateRequired("DB_HOST", "DB_PORT", "DB_USERNAME", "DB_PASSWORD")
```

### **Exemplo de Uso**

```go
func main() {
    // Validar env vars obrigatórias
    if err := config.ValidateRequired("DB_HOST", "DB_PORT", "JWT_SIGNING_KEY"); err != nil {
        log.Fatal(err)
    }

    // Carregar configurações
    port := config.GetEnv("PORT", "8080")
    dbHost, _ := config.GetEnvRequired("DB_HOST")
    dbPort := config.GetEnvInt("DB_PORT", 5432)
    jwtExpiration := config.GetEnvDuration("JWT_EXPIRATION", 24*time.Hour)
}
```

---

## 2️⃣ database - Gerenciamento de Conexões Multi-Tenant

### **DatabaseManager**

Gerencia conexões para banco shared e bancos tenant com pool de conexões otimizado.

```go
import "primeflow.solutions/pkg/database"

// Criar DatabaseManager
dbManager, err := database.NewDatabaseManager(dbHost, dbPort, dbUsername, dbPassword)

// Configurar migrations
dbManager.SetSharedMigrationFunc(func(db *gorm.DB) error {
    return db.AutoMigrate(&Tenant{}, &User{})
})

dbManager.SetMigrationFunc(func(db *gorm.DB, tenantSlug string) error {
    return db.AutoMigrate(&Organization{}, &Product{})
})

// Obter banco shared
sharedDB := dbManager.GetSharedDB()

// Obter banco tenant (cria conexão se não existir)
tenantDB, err := dbManager.GetTenantDB("tenant-slug")

// Criar novo banco
err := dbManager.CreateDatabase("new-tenant-db")

// Fechar todas as conexões
dbManager.Close()
```

### **RequestContext**

Contexto de requisição com informações de tenant e usuário.

```go
type RequestContext struct {
    TenantID int
    UserID   int
    TenantDB *gorm.DB
    SharedDB *gorm.DB
}

// Verificar se está autenticado
if reqCtx.IsAuthenticated() {
    // TenantID > 0 e UserID > 0
}
```

### **Configuração de Pool de Conexões**

O DatabaseManager configura automaticamente:
- `MaxIdleConns`: 10 (conexões idle no pool)
- `MaxOpenConns`: 100 (conexões abertas máximas)
- `ConnMaxLifetime`: 1 hora (tempo máximo de vida)
- `ConnMaxIdleTime`: 10 minutos (tempo máximo idle)

---

## 3️⃣ jwt - Geração e Validação de Tokens JWT

### **TokenGenerator**

```go
import "primeflow.solutions/pkg/jwt"

// Criar TokenGenerator
tokenGen := jwt.NewTokenGenerator(signingKey, 24*time.Hour)

// Gerar token
token, err := tokenGen.GenerateToken(userID, tenantID, tenantSlug)

// Validar token
claims, err := tokenGen.ValidateToken(tokenString)

// Acessar claims
userID, _ := strconv.Atoi(claims.Subject)
tenantID := claims.TenantID
tenantSlug := claims.TenantSlug
```

### **CustomClaims**

```go
type CustomClaims struct {
    TenantID   int    `json:"tenantId"`
    TenantSlug string `json:"tenantSlug"`
    jwt.RegisteredClaims
}
```

### **Exemplo Completo**

```go
// Inicializar
signingKey := config.GetEnvRequired("JWT_SIGNING_KEY")
expiration := config.GetEnvDuration("JWT_EXPIRATION", 24*time.Hour)
tokenGen := jwt.NewTokenGenerator(signingKey, expiration)

// Login - Gerar token
token, err := tokenGen.GenerateToken(user.ID, tenant.ID, tenant.Slug)
if err != nil {
    return fmt.Errorf("failed to generate token: %w", err)
}

// Middleware - Validar token
claims, err := tokenGen.ValidateToken(tokenString)
if err != nil {
    return fmt.Errorf("invalid token: %w", err)
}
```

---

## 4️⃣ logger - Logs Estruturados (slog)

### **Funções Disponíveis**

```go
import "primeflow.solutions/pkg/logger"

// Inicializar logger (no main.go)
logger.Init("INFO") // DEBUG, INFO, WARN, ERROR

// Logs estruturados
logger.Debug("debug message", "key", "value")
logger.Info("info message", "userId", 123, "action", "create")
logger.Warn("warning message", "reason", "timeout")
logger.Error("error message", "error", err, "userId", 123)
```

### **Níveis de Log**

- `DEBUG`: Informações detalhadas para debugging
- `INFO`: Informações gerais (operações bem-sucedidas)
- `WARN`: Avisos (situações anormais mas não críticas)
- `ERROR`: Erros (falhas que precisam atenção)

### **Exemplo de Uso em Use Cases**

```go
func (uc *CreateUseCase) Execute(ctx context.Context, req CreateRequest) (*User, error) {
    logger.Info("creating user", "username", req.Username, "email", req.Email)

    user := ToEntity(req)

    if err := uc.repo.Create(ctx, user); err != nil {
        logger.Error("failed to create user", "username", req.Username, "error", err)
        return nil, fmt.Errorf("failed to create user: %w", err)
    }

    logger.Info("user created successfully", "id", user.ID, "username", user.Username)
    return user, nil
}
```

### **Regras de Logging**

- ✅ `logger.Info()` para Create, Update, Delete
- ✅ `logger.Debug()` para Get, List
- ✅ `logger.Error()` para todos os erros
- ✅ Sempre incluir contexto relevante (id, name, etc)
- ✅ Logs estruturados (key-value pairs)

---

## 5️⃣ middleware - Middlewares HTTP

### **HandlerFunc**

Tipo para converter CustomHandler em http.HandlerFunc.

```go
import "primeflow.solutions/pkg/middleware"

// CustomHandler recebe RequestContext
type CustomHandler func(w http.ResponseWriter, r *http.Request, ctx *database.RequestContext)

// HandlerFunc converte CustomHandler para http.HandlerFunc
type HandlerFunc func(CustomHandler) http.HandlerFunc
```

### **Uso no Main**

```go
func setupRouter(dbManager *database.DatabaseManager) *http.ServeMux {
    mux := http.NewServeMux()

    // Criar wrapper de middleware
    wrap := middleware.Wrap(dbManager)

    // User module
    userHandler := user.NewHandler(func(reqCtx *database.RequestContext) *user.Repository {
        return user.NewRepository(reqCtx.TenantDB)
    })

    // Registrar rotas com middleware
    mux.HandleFunc("GET /v1/users", wrap(userHandler.List))
    mux.HandleFunc("POST /v1/users", wrap(userHandler.Create))

    return mux
}
```

### **Middlewares Disponíveis**

```go
// Wrap - Middleware principal (autenticação + tenant)
wrap := middleware.Wrap(dbManager)

// Private - Rotas privadas (requer autenticação)
// Public - Rotas públicas (sem autenticação)
```

---

## 6️⃣ pagination - Paginação, Filtros e Ordenação

### **Estruturas**

```go
// QueryRequest - Input da API
type QueryRequest struct {
    Page          int           `json:"page"`
    PageSize      int           `json:"pageSize"`
    Filter        string        `json:"filter,omitempty"`
    OrderBy       []OrderByItem `json:"orderBy,omitempty"`
    DisplayFields []string      `json:"displayFields,omitempty"`
}

// QueryResponse - Output da API
type QueryResponse[T any] struct {
    Data          []T `json:"data"`
    TotalPage     int `json:"totalPage"`
    TotalElements int `json:"totalElements"`
}
```

### **Parsing**

```go
import "primeflow.solutions/pkg/pagination"

// Parse de query params (GET)
req, err := pagination.ParseQueryParams(r)

// Parse de body JSON (POST)
req, err := pagination.ParseQueryRequest(r)
```

### **QueryBuilder**

```go
// Definir campos permitidos
allowedFields := map[string]string{
    "id":        "id",
    "name":      "name",
    "email":     "email",
    "active":    "active",
    "createdAt": "created_at",
    "updatedAt": "updated_at",
}

// Criar QueryBuilder
qb := pagination.NewQueryBuilder(db, req, allowedFields)

// Executar query
var users []*User
response, err := qb.Build(&users)

// Ou usar BuildResponse para criar resposta manualmente
dtos := ToListResponse(users)
response := pagination.BuildResponse(dtos, total, req.PageSize)
```

### **Sintaxe de Filtros**

```
// Operadores: =, !=, >, >=, <, <=, like, in, not in
// Lógicos: and, or, ()
// Valores: `string`, number, true/false

// Exemplos:
name = `João`
age > 18
status in (`active`, `pending`)
active = true
name like `%Silva%`
(age > 18 and age < 65) or status = `vip`
```

### **Exemplo Completo no Handler**

```go
func (h *Handler) List(w http.ResponseWriter, r *http.Request, reqCtx *database.RequestContext) {
    req, err := pagination.ParseQueryParams(r)
    if err != nil {
        respondError(w, http.StatusBadRequest, "Invalid query parameters", err)
        return
    }

    repo := h.repoFactory(reqCtx)
    uc := NewListUseCase(repo)

    ctx, cancel := withTimeout(r.Context())
    defer cancel()

    entities, total, err := uc.Execute(ctx, req, AllowedFields)
    if err != nil {
        respondError(w, http.StatusInternalServerError, "Failed to list users", err)
        return
    }

    dtos := ToListResponse(entities)
    response := pagination.BuildResponse(dtos, total, req.PageSize)
    respondJSON(w, http.StatusOK, response)
}
```

### **Constantes**

```go
const (
    DefaultPage     = 1
    DefaultPageSize = 10
    MaxPageSize     = 100
)
```

---

## 7️⃣ repository - Repository Pattern com Generics

### **BaseRepository[T]**

Implementação genérica de repositório com CRUD básico.

```go
import pkgRepo "primeflow.solutions/pkg/repository"

// Criar repository específico
type Repository struct {
    *pkgRepo.BaseRepository[User]
}

func NewRepository(db *gorm.DB) *Repository {
    return &Repository{
        BaseRepository: pkgRepo.NewBaseRepository[User](db),
    }
}

// Override FindByID para erro customizado
func (r *Repository) FindByID(ctx context.Context, id int) (*User, error) {
    user, err := r.BaseRepository.FindByID(ctx, id)
    if err != nil {
        if err == gorm.ErrRecordNotFound || err.Error() == "record not found" {
            return nil, ErrUserNotFound
        }
        return nil, err
    }
    return user, nil
}

// Adicionar métodos específicos
func (r *Repository) FindByEmail(ctx context.Context, email string) (*User, error) {
    var user User
    err := r.DB().WithContext(ctx).Where("email = ?", email).First(&user).Error
    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, ErrUserNotFound
        }
        return nil, fmt.Errorf("failed to find user by email: %w", err)
    }
    return &user, nil
}
```

### **Métodos Herdados**

```go
// CRUD básico (herdados do BaseRepository)
Create(ctx context.Context, entity *T) error
FindByID(ctx context.Context, id int) (*T, error)
FindAll(ctx context.Context) ([]*T, error)
List(ctx context.Context, req pagination.QueryRequest, allowedFields map[string]string) ([]*T, int, error)
Update(ctx context.Context, entity *T) error
Delete(ctx context.Context, id int) error

// Acesso ao GORM DB
DB() *gorm.DB
```

### **Uso no Use Case**

```go
type CreateUseCase struct {
    repo *Repository
}

func NewCreateUseCase(repo *Repository) *CreateUseCase {
    return &CreateUseCase{repo: repo}
}

func (uc *CreateUseCase) Execute(ctx context.Context, req CreateRequest) (*User, error) {
    user := ToEntity(req)

    // Usa método herdado
    if err := uc.repo.Create(ctx, user); err != nil {
        return nil, fmt.Errorf("failed to create user: %w", err)
    }

    return user, nil
}
```

---

## 8️⃣ serviceclient - Cliente HTTP para Comunicação entre Serviços

### **Client**

Cliente HTTP para comunicação service-to-service via gateway.

```go
import "primeflow.solutions/pkg/serviceclient"

// Criar client
client := serviceclient.New()

// Ou com configuração customizada
client := serviceclient.NewWithConfig(serviceclient.Config{
    GatewayURL: "http://gateway:8080",
    Timeout:    30 * time.Second,
})

// Com headers
client = client.WithHeaders(map[string]string{
    "X-Custom-Header": "value",
})

// Com tenant
client = client.WithTenant("tenant-slug")

// Com autenticação
client = client.WithAuth(token)
```

### **Métodos HTTP**

```go
// GET
resp, err := client.Get(ctx, "users", "/v1/users/123")

// POST
resp, err := client.Post(ctx, "users", "/v1/users", createRequest)

// PUT
resp, err := client.Put(ctx, "users", "/v1/users/123", updateRequest)

// DELETE
resp, err := client.Delete(ctx, "users", "/v1/users/123")

// Request customizado
resp, err := client.Do(ctx, serviceclient.Request{
    Service: "users",
    Path:    "/v1/users",
    Method:  http.MethodPost,
    Body:    createRequest,
    Headers: map[string]string{"X-Custom": "value"},
})
```

### **Response**

```go
// Verificar status
if resp.IsSuccess() {
    // 2xx
}

if resp.IsError() {
    // 4xx ou 5xx
}

// Decodificar JSON
var user User
if err := resp.DecodeJSON(&user); err != nil {
    return err
}

// Acessar body raw
body := resp.Body

// Acessar headers
contentType := resp.Headers.Get("Content-Type")
```

### **Exemplo Completo**

```go
// No use case
func (uc *CreateOrderUseCase) Execute(ctx context.Context, req CreateRequest) (*Order, error) {
    // Criar client
    client := serviceclient.New().
        WithTenant(req.TenantSlug).
        WithAuth(req.Token)

    // Buscar usuário do serviço de users
    resp, err := client.Get(ctx, "users", fmt.Sprintf("/v1/users/%d", req.UserID))
    if err != nil {
        return nil, fmt.Errorf("failed to get user: %w", err)
    }

    if !resp.IsSuccess() {
        return nil, fmt.Errorf("user not found: status %d", resp.StatusCode)
    }

    var user User
    if err := resp.DecodeJSON(&user); err != nil {
        return nil, fmt.Errorf("failed to decode user: %w", err)
    }

    // Criar order
    order := &Order{
        UserID: user.ID,
        Total:  req.Total,
    }

    if err := uc.repo.Create(ctx, order); err != nil {
        return nil, fmt.Errorf("failed to create order: %w", err)
    }

    return order, nil
}
```

---

## 9️⃣ validator - Validação de Structs

### **Funções Disponíveis**

```go
import pkgValidator "primeflow.solutions/pkg/validator"

// Validar struct
err := pkgValidator.Validate(req)

// Formatar erros de validação
validationErrors := pkgValidator.FormatValidationErrors(err)

// Registrar validação customizada
pkgValidator.RegisterValidation("cnpj", ValidateCNPJ)
```

### **Tags de Validação**

```go
type CreateRequest struct {
    Name     string  `json:"name" validate:"required,min=2,max=255"`
    Email    string  `json:"email" validate:"required,email"`
    Age      int     `json:"age" validate:"required,gte=18,lte=120"`
    Price    float64 `json:"price" validate:"required,gt=0"`
    Website  string  `json:"website" validate:"omitempty,url"`
    Username string  `json:"username" validate:"required,alphanum,min=3,max=20"`
}
```

### **Tags Disponíveis**

- `required` - Campo obrigatório
- `email` - Email válido
- `min=N` - Mínimo N caracteres
- `max=N` - Máximo N caracteres
- `gt=N` - Maior que N
- `gte=N` - Maior ou igual a N
- `lt=N` - Menor que N
- `lte=N` - Menor ou igual a N
- `alpha` - Apenas letras
- `alphanum` - Letras e números
- `numeric` - Apenas números
- `url` - URL válida
- `omitempty` - Opcional (não valida se vazio)

### **Uso no Handler**

```go
func (h *Handler) Create(w http.ResponseWriter, r *http.Request, reqCtx *database.RequestContext) {
    var req CreateRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid request body", err)
        return
    }

    // Validar
    if err := pkgValidator.Validate(req); err != nil {
        respondValidationError(w, err)
        return
    }

    // ... continuar
}

func respondValidationError(w http.ResponseWriter, err error) {
    validationErrors := pkgValidator.FormatValidationErrors(err)
    response := map[string]interface{}{
        "error":  "Validation failed",
        "errors": validationErrors,
    }
    respondJSON(w, http.StatusBadRequest, response)
}
```

### **Resposta de Erro de Validação**

```json
{
  "error": "Validation failed",
  "errors": [
    {
      "field": "name",
      "tag": "required",
      "message": "name is required"
    },
    {
      "field": "email",
      "tag": "email",
      "message": "email must be a valid email address"
    },
    {
      "field": "age",
      "tag": "gte",
      "value": "15",
      "message": "age must be greater than or equal to 18"
    }
  ]
}
```

### **Validação Customizada**

```go
// Definir função de validação
func ValidateCNPJ(fl validator.FieldLevel) bool {
    cnpj := fl.Field().String()
    // Lógica de validação
    return cnpjRegex.MatchString(cnpj)
}

// Registrar no init
func init() {
    pkgValidator.RegisterValidation("cnpj", ValidateCNPJ)
}

// Usar no DTO
type CreateRequest struct {
    TaxID string `json:"taxId" validate:"required,cnpj"`
}
```

---

## ✅ Checklist de Uso do PKG

Ao desenvolver um novo serviço ou módulo, verifique:

- [ ] Usar `config` para env vars (não `os.Getenv` direto)
- [ ] Usar `database.DatabaseManager` para conexões
- [ ] Usar `jwt.TokenGenerator` para tokens
- [ ] Usar `logger` para logs estruturados (não `fmt.Println`)
- [ ] Usar `middleware.Wrap` para rotas
- [ ] Usar `pagination` para listagens
- [ ] Usar `BaseRepository[T]` para CRUD
- [ ] Usar `serviceclient` para comunicação entre serviços
- [ ] Usar `validator` para validação de DTOs

## 🚫 O que NÃO fazer

### ❌ NÃO recrie funcionalidades do PKG

```go
// ❌ ERRADO - Reimplementar logger
func log(msg string) {
    fmt.Println(time.Now(), msg)
}

// ✅ CORRETO - Usar logger do PKG
logger.Info("message", "key", "value")
```

### ❌ NÃO use `os.Getenv` direto

```go
// ❌ ERRADO
port := os.Getenv("PORT")

// ✅ CORRETO
port := config.GetEnv("PORT", "8080")
```

### ❌ NÃO crie seu próprio repository base

```go
// ❌ ERRADO - Criar BaseRepository próprio
type MyBaseRepository[T any] struct { ... }

// ✅ CORRETO - Usar do PKG
type Repository struct {
    *pkgRepo.BaseRepository[User]
}
```

### ❌ NÃO implemente paginação manualmente

```go
// ❌ ERRADO
offset := (page - 1) * pageSize
db.Limit(pageSize).Offset(offset).Find(&users)

// ✅ CORRETO
qb := pagination.NewQueryBuilder(db, req, allowedFields)
response, err := qb.Build(&users)
```

### ❌ NÃO crie cliente HTTP próprio

```go
// ❌ ERRADO
client := &http.Client{}
resp, err := client.Get("http://users-service/v1/users")

// ✅ CORRETO
client := serviceclient.New()
resp, err := client.Get(ctx, "users", "/v1/users")
```

## 📚 Importações Padrão

```go
import (
    "primeflow.solutions/pkg/config"
    "primeflow.solutions/pkg/database"
    "primeflow.solutions/pkg/jwt"
    "primeflow.solutions/pkg/logger"
    "primeflow.solutions/pkg/middleware"
    "primeflow.solutions/pkg/pagination"
    pkgRepo "primeflow.solutions/pkg/repository"
    "primeflow.solutions/pkg/serviceclient"
    pkgValidator "primeflow.solutions/pkg/validator"
)
```

## 🎯 Conclusão

O PKG fornece todas as funcionalidades comuns necessárias para desenvolvimento de serviços Go. **SEMPRE use o que está disponível no PKG** ao invés de reimplementar.

**Benefícios:**
- ✅ Consistência entre serviços
- ✅ Menos código duplicado
- ✅ Manutenção centralizada
- ✅ Padrões testados e validados
- ✅ Desenvolvimento mais rápido

**Lembre-se**: Se você está prestes a implementar algo que parece comum, verifique primeiro se já existe no PKG!

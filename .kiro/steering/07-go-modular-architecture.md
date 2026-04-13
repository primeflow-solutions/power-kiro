# Go - Arquitetura Modular por Domínio

## 🎯 Visão Geral

Este documento define o padrão de **Arquitetura Modular baseada em Domínios** para projetos Go. Este padrão deve ser seguido em TODOS os novos projetos Go e em refatorações de projetos existentes.

## 📋 Princípios Fundamentais

### **1. Organização por Módulo/Domínio (NÃO por Camada)**

❌ **EVITE** (Arquitetura em Camadas):
```
internal/
├── domain/entity/          # 80 arquivos
├── domain/repository/      # 80 arquivos
├── usecase/                # 400 arquivos (80 × 5)
├── infrastructure/         # 80 arquivos
└── api/
    ├── dto/                # 80 arquivos
    ├── mapper/             # 80 arquivos
    └── handler/            # 80 arquivos
```
**Problema**: Código de uma feature espalhado em 7 lugares diferentes.

✅ **USE** (Arquitetura Modular):
```
internal/
├── module/
│   ├── organization/       # 8 arquivos (tudo de Organization)
│   ├── unit/               # 8 arquivos (tudo de Unit)
│   └── product/            # 8 arquivos (tudo de Product)
└── shared/                 # Código compartilhado
```
**Benefício**: Tudo de uma feature em um único lugar.

### **2. Cada Módulo é Independente**

- Módulos NÃO devem depender uns dos outros (exceto via interfaces)
- Mudanças em um módulo NÃO afetam outros módulos
- Cada módulo pode ser desenvolvido, testado e mantido isoladamente

### **3. Estrutura Consistente**

Todos os módulos seguem a MESMA estrutura de 7 arquivos:
1. `entity.go` - Entidade do domínio
2. `repository.go` - Struct + implementação do repositório
3. `dto.go` - DTOs (Request/Response)
4. `mapper.go` - Conversões DTO ↔ Entity
5. `usecase.go` - Todos os use cases
6. `handler.go` - HTTP Handler
7. `routes.go` - Registro de rotas

## 📁 Estrutura Padrão de um Módulo

```
internal/module/[nome]/
├── entity.go      # Entidade do domínio
├── repository.go  # Struct + implementação (embute BaseRepository)
├── dto.go         # DTOs (CreateRequest, UpdateRequest, Response)
├── mapper.go      # Conversões DTO ↔ Entity
├── usecase.go     # Todos os use cases (Create, List, Get, Update, Delete)
├── handler.go     # HTTP Handler (todos os endpoints)
└── routes.go      # Registro de rotas
```

## 📝 Templates de Código

### **1. entity.go**

```go
package [nome]

import (
	"errors"
	"time"
)

// [Nome] represents a [nome] entity
type [Nome] struct {
	ID        int       `gorm:"primaryKey"`
	Name      string    `gorm:"size:255;not null"`
	// Adicione outros campos aqui
	CreatedAt time.Time `gorm:"autoCreateTime"`
	UpdatedAt time.Time `gorm:"autoUpdateTime"`
}

// TableName specifies the table name for GORM
func ([Nome]) TableName() string {
	return "[schema].[table_name]"
}

var (
	Err[Nome]NotFound = errors.New("[nome] not found")
)
```

**Regras:**
- ✅ Entidade simples, apenas dados
- ✅ Tags GORM para mapeamento
- ✅ Timestamps automáticos (`autoCreateTime`, `autoUpdateTime`)
- ✅ Erro customizado para "not found"
- ❌ SEM validação na entidade (validação é no DTO)
- ❌ SEM lógica de negócio na entidade

### **2. repository.go**

```go
package [nome]

import (
	"context"

	"gorm.io/gorm"
	pkgRepo "primeflow.solutions/pkg/repository"
)

// Repository handles [nome] persistence
type Repository struct {
	*pkgRepo.BaseRepository[[Nome]]
}

// NewRepository creates a new [nome] repository instance
func NewRepository(db *gorm.DB) *Repository {
	return &Repository{
		BaseRepository: pkgRepo.NewBaseRepository[[Nome]](db),
	}
}

// FindByID overrides base method to return custom error
func (r *Repository) FindByID(ctx context.Context, id int) (*[Nome], error) {
	entity, err := r.BaseRepository.FindByID(ctx, id)
	if err != nil {
		if err == gorm.ErrRecordNotFound || err.Error() == "record not found" {
			return nil, Err[Nome]NotFound
		}
		return nil, err
	}
	return entity, nil
}

// Adicione métodos específicos aqui quando necessário
// Exemplo:
// func (r *Repository) FindByName(ctx context.Context, name string) (*[Nome], error) {
//     var entity [Nome]
//     err := r.DB().WithContext(ctx).Where("name = ?", name).First(&entity).Error
//     if err != nil {
//         if err == gorm.ErrRecordNotFound {
//             return nil, Err[Nome]NotFound
//         }
//         return nil, err
//     }
//     return &entity, nil
// }
```

**Regras:**
- ✅ Struct que embute `*BaseRepository[T]` (composição)
- ✅ BaseRepository fornece CRUD básico (Create, FindByID, List, Update, Delete)
- ✅ Override de `FindByID` para retornar erro customizado
- ✅ Adicione métodos específicos do domínio quando necessário
- ✅ Acesso ao DB via `r.DB()` para queries customizadas
- ✅ Use generics do Go para type-safety
- ❌ NÃO crie interface separada (YAGNI - You Ain't Gonna Need It)

### **3. repository_impl.go**

**REMOVIDO** - Não é mais necessário! A implementação agora está diretamente em `repository.go`.

**Motivo da simplificação:**
- ❌ Interface + Implementação separadas = complexidade desnecessária
- ❌ Nunca temos múltiplas implementações (sempre GORM)
- ✅ YAGNI (You Ain't Gonna Need It) - Simplicidade > Flexibilidade prematura
- ✅ Struct direta = menos arquivos, menos código, mais clareza

### **3. dto.go**

```go
package [nome]

import "time"

// CreateRequest represents the request to create a [nome]
type CreateRequest struct {
	Name string `json:"name" validate:"required,min=2,max=255"`
	// Adicione outros campos com validações
}

// UpdateRequest represents the request to update a [nome]
type UpdateRequest struct {
	Name string `json:"name" validate:"required,min=2,max=255"`
	// Adicione outros campos com validações
}

// Response represents the [nome] response
type Response struct {
	ID        int       `json:"id"`
	Name      string    `json:"name"`
	// Adicione outros campos
	CreatedAt time.Time `json:"createdAt"`
	UpdatedAt time.Time `json:"updatedAt"`
}
```

**Regras:**
- ✅ DTOs separados para Create, Update e Response
- ✅ Tags `json` para serialização
- ✅ Tags `validate` para validação (go-playground/validator)
- ✅ Validações: `required`, `min`, `max`, `gt`, `gte`, `lt`, `lte`, `email`, `url`, etc
- ✅ Campos opcionais: `omitempty` no JSON e sem `required` no validate

### **4. mapper.go**

```go
package [nome]

// ToResponse converts a [Nome] entity to Response DTO
func ToResponse(entity *[Nome]) Response {
	return Response{
		ID:        entity.ID,
		Name:      entity.Name,
		CreatedAt: entity.CreatedAt,
		UpdatedAt: entity.UpdatedAt,
	}
}

// ToListResponse converts a list of [Nome] entities to Response DTOs
func ToListResponse(entities []*[Nome]) []Response {
	dtos := make([]Response, len(entities))
	for i, entity := range entities {
		dtos[i] = ToResponse(entity)
	}
	return dtos
}

// ToEntity converts a CreateRequest DTO to [Nome] entity
func ToEntity(req CreateRequest) *[Nome] {
	return &[Nome]{
		Name: req.Name,
	}
}

// UpdateFromDTO updates an existing [Nome] entity with data from UpdateRequest DTO
func UpdateFromDTO(entity *[Nome], req UpdateRequest) {
	entity.Name = req.Name
}
```

**Regras:**
- ✅ Funções puras de conversão
- ✅ `ToResponse()` - Entity → Response
- ✅ `ToListResponse()` - []*Entity → []Response
- ✅ `ToEntity()` - CreateRequest → Entity
- ✅ `UpdateFromDTO()` - UpdateRequest → Entity (atualiza campos)
- ❌ SEM lógica de negócio nos mappers

### **5. usecase.go**

```go
package [nome]

import (
	"context"
	"fmt"

	"primeflow.solutions/pkg/logger"
	"primeflow.solutions/pkg/pagination"
)

// ============================================================================
// CREATE USE CASE
// ============================================================================

type CreateUseCase struct {
	repo Repository
}

func NewCreateUseCase(repo Repository) *CreateUseCase {
	return &CreateUseCase{repo: repo}
}

func (uc *CreateUseCase) Execute(ctx context.Context, name string) (*[Nome], error) {
	logger.Info("creating [nome]", "name", name)

	entity := &[Nome]{
		Name: name,
	}

	if err := uc.repo.Create(ctx, entity); err != nil {
		logger.Error("failed to create [nome]", "name", name, "error", err)
		return nil, fmt.Errorf("failed to create [nome]: %w", err)
	}

	logger.Info("[nome] created successfully", "id", entity.ID, "name", entity.Name)
	return entity, nil
}

// ============================================================================
// LIST USE CASE
// ============================================================================

type ListUseCase struct {
	repo Repository
}

func NewListUseCase(repo Repository) *ListUseCase {
	return &ListUseCase{repo: repo}
}

func (uc *ListUseCase) Execute(ctx context.Context, req pagination.QueryRequest, allowedFields map[string]string) ([]*[Nome], int, error) {
	logger.Debug("listing [nome]s", "page", req.Page, "pageSize", req.PageSize)

	entities, total, err := uc.repo.List(ctx, req, allowedFields)
	if err != nil {
		logger.Error("failed to list [nome]s", "error", err)
		return nil, 0, fmt.Errorf("failed to list [nome]s: %w", err)
	}

	logger.Info("[nome]s listed successfully", "count", len(entities), "total", total)
	return entities, total, nil
}

// ============================================================================
// GET USE CASE
// ============================================================================

type GetUseCase struct {
	repo Repository
}

func NewGetUseCase(repo Repository) *GetUseCase {
	return &GetUseCase{repo: repo}
}

func (uc *GetUseCase) Execute(ctx context.Context, id int) (*[Nome], error) {
	logger.Debug("getting [nome]", "id", id)

	entity, err := uc.repo.FindByID(ctx, id)
	if err != nil {
		logger.Error("failed to find [nome]", "id", id, "error", err)
		return nil, fmt.Errorf("failed to find [nome]: %w", err)
	}

	logger.Debug("[nome] found", "id", entity.ID, "name", entity.Name)
	return entity, nil
}

// ============================================================================
// UPDATE USE CASE
// ============================================================================

type UpdateUseCase struct {
	repo Repository
}

func NewUpdateUseCase(repo Repository) *UpdateUseCase {
	return &UpdateUseCase{repo: repo}
}

func (uc *UpdateUseCase) Execute(ctx context.Context, id int, name string) (*[Nome], error) {
	logger.Info("updating [nome]", "id", id, "name", name)

	entity, err := uc.repo.FindByID(ctx, id)
	if err != nil {
		logger.Error("failed to find [nome] for update", "id", id, "error", err)
		return nil, fmt.Errorf("failed to find [nome]: %w", err)
	}

	entity.Name = name

	if err := uc.repo.Update(ctx, entity); err != nil {
		logger.Error("failed to update [nome]", "id", id, "error", err)
		return nil, fmt.Errorf("failed to update [nome]: %w", err)
	}

	logger.Info("[nome] updated successfully", "id", entity.ID, "name", entity.Name)
	return entity, nil
}

// ============================================================================
// DELETE USE CASE
// ============================================================================

type DeleteUseCase struct {
	repo Repository
}

func NewDeleteUseCase(repo Repository) *DeleteUseCase {
	return &DeleteUseCase{repo: repo}
}

func (uc *DeleteUseCase) Execute(ctx context.Context, id int) error {
	logger.Info("deleting [nome]", "id", id)

	// Check if entity exists
	_, err := uc.repo.FindByID(ctx, id)
	if err != nil {
		logger.Error("failed to find [nome] for deletion", "id", id, "error", err)
		return fmt.Errorf("failed to find [nome]: %w", err)
	}

	if err := uc.repo.Delete(ctx, id); err != nil {
		logger.Error("failed to delete [nome]", "id", id, "error", err)
		return fmt.Errorf("failed to delete [nome]: %w", err)
	}

	logger.Info("[nome] deleted successfully", "id", id)
	return nil
}
```

**Regras:**
- ✅ Todos os 5 use cases em UM único arquivo
- ✅ Separados por comentários `// ============`
- ✅ Logs estruturados em TODAS as operações:
  - `logger.Info()` para Create, Update, Delete
  - `logger.Debug()` para Get, List
  - `logger.Error()` para erros
- ✅ Error wrapping com `fmt.Errorf("...: %w", err)`
- ✅ Contexto rico nos logs (id, name, etc)
- ✅ Verificar existência antes de Delete

### **6. handler.go**

```go
package [nome]

import (
	"context"
	"encoding/json"
	"errors"
	"net/http"
	"strconv"
	"time"

	"primeflow.solutions/pkg/database"
	"primeflow.solutions/pkg/pagination"
	pkgValidator "primeflow.solutions/pkg/validator"
)

const (
	DefaultTimeout = 5 * time.Second
)

type Handler struct {
	repoFactory func(*database.RequestContext) *Repository
}

func NewHandler(repoFactory func(*database.RequestContext) *Repository) *Handler {
	return &Handler{repoFactory: repoFactory}
}

var AllowedFields = map[string]string{
	"id":        "id",
	"name":      "name",
	"createdAt": "created_at",
	"updatedAt": "updated_at",
}

func withTimeout(ctx context.Context) (context.Context, context.CancelFunc) {
	return context.WithTimeout(ctx, DefaultTimeout)
}

// List handles GET /v1/[nome]s
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
		respondError(w, http.StatusInternalServerError, "Failed to list [nome]s", err)
		return
	}

	dtos := ToListResponse(entities)
	response := pagination.BuildResponse(dtos, total, req.PageSize)
	respondJSON(w, http.StatusOK, response)
}

// Create handles POST /v1/[nome]s
func (h *Handler) Create(w http.ResponseWriter, r *http.Request, reqCtx *database.RequestContext) {
	var req CreateRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		respondError(w, http.StatusBadRequest, "Invalid request body", err)
		return
	}

	if err := pkgValidator.Validate(req); err != nil {
		respondValidationError(w, err)
		return
	}

	repo := h.repoFactory(reqCtx)
	uc := NewCreateUseCase(repo)

	ctx, cancel := withTimeout(r.Context())
	defer cancel()

	entity, err := uc.Execute(ctx, req.Name)
	if err != nil {
		respondError(w, http.StatusInternalServerError, "Failed to create [nome]", err)
		return
	}

	respondJSON(w, http.StatusCreated, ToResponse(entity))
}

// GetByID handles GET /v1/[nome]s/{id}
func (h *Handler) GetByID(w http.ResponseWriter, r *http.Request, reqCtx *database.RequestContext) {
	idStr := r.PathValue("id")
	id, err := strconv.Atoi(idStr)
	if err != nil {
		respondError(w, http.StatusBadRequest, "Invalid [nome] ID", err)
		return
	}

	repo := h.repoFactory(reqCtx)
	uc := NewGetUseCase(repo)

	ctx, cancel := withTimeout(r.Context())
	defer cancel()

	entity, err := uc.Execute(ctx, id)
	if err != nil {
		if errors.Is(err, Err[Nome]NotFound) {
			respondError(w, http.StatusNotFound, "[Nome] not found", err)
			return
		}
		respondError(w, http.StatusInternalServerError, "Failed to get [nome]", err)
		return
	}

	respondJSON(w, http.StatusOK, ToResponse(entity))
}

// Update handles PUT /v1/[nome]s/{id}
func (h *Handler) Update(w http.ResponseWriter, r *http.Request, reqCtx *database.RequestContext) {
	idStr := r.PathValue("id")
	id, err := strconv.Atoi(idStr)
	if err != nil {
		respondError(w, http.StatusBadRequest, "Invalid [nome] ID", err)
		return
	}

	var req UpdateRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		respondError(w, http.StatusBadRequest, "Invalid request body", err)
		return
	}

	if err := pkgValidator.Validate(req); err != nil {
		respondValidationError(w, err)
		return
	}

	repo := h.repoFactory(reqCtx)
	uc := NewUpdateUseCase(repo)

	ctx, cancel := withTimeout(r.Context())
	defer cancel()

	entity, err := uc.Execute(ctx, id, req.Name)
	if err != nil {
		if errors.Is(err, Err[Nome]NotFound) {
			respondError(w, http.StatusNotFound, "[Nome] not found", err)
			return
		}
		respondError(w, http.StatusInternalServerError, "Failed to update [nome]", err)
		return
	}

	respondJSON(w, http.StatusOK, ToResponse(entity))
}

// Delete handles DELETE /v1/[nome]s/{id}
func (h *Handler) Delete(w http.ResponseWriter, r *http.Request, reqCtx *database.RequestContext) {
	idStr := r.PathValue("id")
	id, err := strconv.Atoi(idStr)
	if err != nil {
		respondError(w, http.StatusBadRequest, "Invalid [nome] ID", err)
		return
	}

	repo := h.repoFactory(reqCtx)
	uc := NewDeleteUseCase(repo)

	ctx, cancel := withTimeout(r.Context())
	defer cancel()

	err = uc.Execute(ctx, id)
	if err != nil {
		if errors.Is(err, Err[Nome]NotFound) {
			respondError(w, http.StatusNotFound, "[Nome] not found", err)
			return
		}
		respondError(w, http.StatusInternalServerError, "Failed to delete [nome]", err)
		return
	}

	w.WriteHeader(http.StatusNoContent)
}

// ============================================================================
// HTTP HELPERS
// ============================================================================

type errorResponse struct {
	Error   string `json:"error"`
	Message string `json:"message,omitempty"`
}

func respondError(w http.ResponseWriter, status int, message string, err error) {
	response := errorResponse{
		Error:   message,
		Message: err.Error(),
	}
	respondJSON(w, status, response)
}

func respondJSON(w http.ResponseWriter, status int, data interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(data)
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

**Regras:**
- ✅ Handler recebe `repoFactory` via Dependency Injection
- ✅ Timeout de 5 segundos em TODAS as operações
- ✅ Validação estruturada antes de chamar use case
- ✅ Tratamento de erros específicos (NotFound → 404)
- ✅ HTTP Status corretos:
  - 200 OK - Get, Update, List
  - 201 Created - Create
  - 204 No Content - Delete
  - 400 Bad Request - Validação
  - 404 Not Found - Não encontrado
  - 500 Internal Server Error - Erro interno
- ✅ AllowedFields para paginação/filtros
- ✅ HTTP helpers no final do arquivo

### **7. routes.go**

```go
package [nome]

import (
	"net/http"

	"primeflow.solutions/pkg/middleware"
)

// RegisterRoutes registra todas as rotas do módulo [nome]
func RegisterRoutes(mux *http.ServeMux, handler *Handler, wrap middleware.HandlerFunc) {
	mux.HandleFunc("GET /v1/[nome]s", wrap(handler.List))
	mux.HandleFunc("POST /v1/[nome]s", wrap(handler.Create))
	mux.HandleFunc("GET /v1/[nome]s/{id}", wrap(handler.GetByID))
	mux.HandleFunc("PUT /v1/[nome]s/{id}", wrap(handler.Update))
	mux.HandleFunc("DELETE /v1/[nome]s/{id}", wrap(handler.Delete))
}
```

**Regras:**
- ✅ Função `RegisterRoutes()` para registrar todas as rotas
- ✅ Prefixo `/v1/` para versionamento
- ✅ Plural no nome do recurso (`/v1/organizations`, `/v1/products`)
- ✅ Middleware wrapper para autenticação/autorização
- ✅ Padrão REST:
  - GET /v1/[nome]s - List
  - POST /v1/[nome]s - Create
  - GET /v1/[nome]s/{id} - Get
  - PUT /v1/[nome]s/{id} - Update
  - DELETE /v1/[nome]s/{id} - Delete

## 🔧 Registro no main.go

```go
func setupRouter(dbManager *database.DatabaseManager) *http.ServeMux {
	mux := http.NewServeMux()

	wrap := middleware.Wrap(dbManager)

	// [Nome] module
	[nome]Handler := [nome].NewHandler(func(reqCtx *database.RequestContext) [nome].Repository {
		return [nome].NewRepository(reqCtx.TenantDB)
	})
	[nome].RegisterRoutes(mux, [nome]Handler, wrap)

	return mux
}
```

**Regras:**
- ✅ Factory function para criar repositório
- ✅ Dependency Injection via factory
- ✅ Registro via `RegisterRoutes()`
- ✅ Middleware aplicado via `wrap`

## 📊 Shared Kernel

Código compartilhado entre módulos fica em `internal/shared/`:

```
internal/shared/
├── health/         # Health check
├── errors/         # Erros customizados compartilhados
├── middleware/     # Middlewares HTTP
└── types/          # Tipos compartilhados
```

**Regras:**
- ✅ Apenas código REALMENTE compartilhado
- ❌ NÃO coloque lógica de negócio em shared
- ❌ NÃO crie dependências entre módulos via shared

## ✅ Checklist de Qualidade

Ao criar um novo módulo, verifique:

- [ ] Estrutura de 7 arquivos criada
- [ ] Package name correto
- [ ] Entidade com timestamps automáticos
- [ ] Repository embute BaseRepository (struct, não interface)
- [ ] DTOs com validações
- [ ] Mappers implementados
- [ ] 5 use cases com logs
- [ ] Handler com timeout e validação
- [ ] Rotas registradas
- [ ] Registrado no main.go
- [ ] Código compila sem erros
- [ ] Imports corretos

## 🚫 Anti-Padrões (O que NÃO fazer)

### ❌ NÃO organize por camadas
```
internal/
├── domain/
├── usecase/
├── infrastructure/
└── api/
```

### ❌ NÃO coloque validação na entidade
```go
func (o *Organization) Validate() error {
    if o.Name == "" {
        return errors.New("name is required")
    }
    return nil
}
```

### ❌ NÃO crie dependências diretas entre módulos
```go
// ❌ ERRADO
import "primeflow.solutions/service/internal/module/organization"

func (uc *CreateUnitUseCase) Execute(...) {
    org := organization.Organization{} // Dependência direta
}
```

### ❌ NÃO espalhe use cases em múltiplos arquivos
```
usecase/
├── create_organization.go
├── list_organizations.go
├── get_organization.go
├── update_organization.go
└── delete_organization.go
```

### ❌ NÃO esqueça logs estruturados
```go
// ❌ ERRADO
func (uc *CreateUseCase) Execute(...) {
    entity := &Organization{Name: name}
    return uc.repo.Create(ctx, entity) // Sem logs!
}
```

### ❌ NÃO esqueça timeout
```go
// ❌ ERRADO
func (h *Handler) Create(w, r, reqCtx) {
    entity, err := uc.Execute(r.Context(), req.Name) // Sem timeout!
}
```

### ❌ NÃO esqueça error wrapping
```go
// ❌ ERRADO
if err != nil {
    return err // Perde contexto
}

// ✅ CORRETO
if err != nil {
    return fmt.Errorf("failed to create organization: %w", err)
}
```

## 🎯 Benefícios desta Arquitetura

1. **Alta Coesão** - Tudo de uma feature em um lugar
2. **Baixo Acoplamento** - Módulos independentes
3. **Escalabilidade** - Fácil adicionar 80+ módulos
4. **Trabalho em Equipe** - Cada dev em um módulo
5. **Manutenibilidade** - Mudanças localizadas
6. **Testabilidade** - Fácil testar módulos isoladamente
7. **Onboarding** - 76% mais rápido
8. **Navegação** - Fácil encontrar código

## 📚 Referências

- Modular Monolith Architecture
- Domain-Driven Design (Eric Evans)
- Clean Architecture (Robert C. Martin)
- SOLID Principles
- Go Project Layout

---

**Este padrão deve ser seguido em TODOS os projetos Go!**

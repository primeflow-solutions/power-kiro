# Go - Migração para Arquitetura Modular

## 🎯 Objetivo

Este documento fornece um guia passo a passo para migrar projetos Go existentes (organizados por camadas) para a arquitetura modular.

## 📋 Pré-requisitos

Antes de iniciar a migração, certifique-se de que o projeto possui:

- ✅ Repositório base genérico (`pkg/repository/base_repository.go`)
- ✅ Validador estruturado (`pkg/validator/validator.go`)
- ✅ Logger estruturado (`pkg/logger/`)
- ✅ Sistema de paginação (`pkg/pagination/`)
- ✅ Middleware de autenticação (`pkg/middleware/`)

Se algum desses componentes não existir, crie-os primeiro no `go-pkg-lib`.

## 🗺️ Processo de Migração

### **Fase 1: Análise e Planejamento**

#### **1.1. Identificar Entidades**

Liste todas as entidades do projeto:

```bash
# Listar entidades
ls internal/domain/entity/

# Exemplo de saída:
# organization.go
# unit.go
# product.go
# customer.go
```

#### **1.2. Identificar Dependências entre Entidades**

Mapeie as dependências:

```
Organization (independente)
  └── Unit (depende de Organization)
      └── Employee (depende de Unit)

Customer (independente)
  └── Order (depende de Customer)
      └── OrderItem (depende de Order e Product)

Product (independente)
```

#### **1.3. Definir Ordem de Migração**

Migre primeiro as entidades independentes, depois as dependentes:

```
1. Organization
2. Product
3. Customer
4. Unit (depende de Organization)
5. Order (depende de Customer)
6. OrderItem (depende de Order e Product)
```

### **Fase 2: Criar Estrutura Modular**

#### **2.1. Criar Pastas dos Módulos**

```bash
mkdir -p internal/module/organization
mkdir -p internal/module/unit
mkdir -p internal/module/product
mkdir -p internal/shared/health
```

#### **2.2. Criar Estrutura de Arquivos**

Para cada módulo, crie os 8 arquivos:

```bash
# Exemplo para organization
touch internal/module/organization/entity.go
touch internal/module/organization/repository.go
touch internal/module/organization/repository_impl.go
touch internal/module/organization/dto.go
touch internal/module/organization/mapper.go
touch internal/module/organization/usecase.go
touch internal/module/organization/handler.go
touch internal/module/organization/routes.go
```

### **Fase 3: Migrar Módulo por Módulo**

#### **3.1. Migrar Entity**

**Antes** (`internal/domain/entity/organization.go`):
```go
package entity

type Organization struct {
	ID        int
	Name      string
	CreatedAt time.Time
	UpdatedAt time.Time
}

func (o *Organization) Validate() error {
	if o.Name == "" {
		return errors.New("name is required")
	}
	return nil
}
```

**Depois** (`internal/module/organization/entity.go`):
```go
package organization

type Organization struct {
	ID        int       `gorm:"primaryKey"`
	Name      string    `gorm:"size:255;not null"`
	CreatedAt time.Time `gorm:"autoCreateTime"`
	UpdatedAt time.Time `gorm:"autoUpdateTime"`
}

func (Organization) TableName() string {
	return "organizations.organizations"
}

var (
	ErrOrganizationNotFound = errors.New("organization not found")
)
```

**Mudanças:**
- ✅ Removida validação (vai para DTO)
- ✅ Adicionadas tags GORM
- ✅ Timestamps automáticos
- ✅ TableName() definido
- ✅ Erro customizado criado

#### **3.2. Migrar Repository**

**Antes** (`internal/domain/repository.go`):
```go
package domain

type OrganizationRepository interface {
	Create(ctx context.Context, org *entity.Organization) error
	FindByID(ctx context.Context, id int) (*entity.Organization, error)
	List(ctx context.Context) ([]*entity.Organization, error)
	Update(ctx context.Context, org *entity.Organization) error
	Delete(ctx context.Context, id int) error
}
```

**Depois** (`internal/module/organization/repository.go`):
```go
package organization

import (
	pkgRepo "primeflow.solutions/pkg/repository"
)

type Repository interface {
	pkgRepo.BaseRepository[Organization]
}
```

**Mudanças:**
- ✅ Herda BaseRepository (elimina código duplicado)
- ✅ Usa generics
- ✅ Métodos CRUD já incluídos

#### **3.3. Migrar Repository Implementation**

**Antes** (`internal/infrastructure/repository/postgres_organization_repository.go`):
```go
package repository

type postgresOrganizationRepository struct {
	db *gorm.DB
}

func NewPostgresOrganizationRepository(db *gorm.DB) domain.OrganizationRepository {
	return &postgresOrganizationRepository{db: db}
}

func (r *postgresOrganizationRepository) Create(ctx context.Context, org *entity.Organization) error {
	return r.db.WithContext(ctx).Create(org).Error
}

func (r *postgresOrganizationRepository) FindByID(ctx context.Context, id int) (*entity.Organization, error) {
	var org entity.Organization
	err := r.db.WithContext(ctx).First(&org, id).Error
	if err != nil {
		if err == gorm.ErrRecordNotFound {
			return nil, errors.New("organization not found")
		}
		return nil, err
	}
	return &org, nil
}

// ... outros métodos CRUD
```

**Depois** (`internal/module/organization/repository_impl.go`):
```go
package organization

import (
	"context"
	"gorm.io/gorm"
	pkgRepo "primeflow.solutions/pkg/repository"
)

type repository struct {
	*pkgRepo.GormBaseRepository[Organization]
}

func NewRepository(db *gorm.DB) Repository {
	return &repository{
		GormBaseRepository: pkgRepo.NewGormBaseRepository[Organization](db),
	}
}

func (r *repository) FindByID(ctx context.Context, id int) (*Organization, error) {
	org, err := r.GormBaseRepository.FindByID(ctx, id)
	if err != nil {
		if err == gorm.ErrRecordNotFound || err.Error() == "record not found" {
			return nil, ErrOrganizationNotFound
		}
		return nil, err
	}
	return org, nil
}
```

**Mudanças:**
- ✅ Composição do GormBaseRepository
- ✅ Elimina ~80% do código
- ✅ Override apenas de FindByID para erro customizado

#### **3.4. Migrar DTOs**

**Antes** (`internal/presentation/http/dto/organization_dto.go`):
```go
package dto

type CreateOrganizationRequest struct {
	Name string `json:"name"`
}

type OrganizationResponse struct {
	ID        int       `json:"id"`
	Name      string    `json:"name"`
	CreatedAt time.Time `json:"createdAt"`
	UpdatedAt time.Time `json:"updatedAt"`
}
```

**Depois** (`internal/module/organization/dto.go`):
```go
package organization

type CreateRequest struct {
	Name string `json:"name" validate:"required,min=2,max=255"`
}

type UpdateRequest struct {
	Name string `json:"name" validate:"required,min=2,max=255"`
}

type Response struct {
	ID        int       `json:"id"`
	Name      string    `json:"name"`
	CreatedAt time.Time `json:"createdAt"`
	UpdatedAt time.Time `json:"updatedAt"`
}
```

**Mudanças:**
- ✅ Adicionadas tags de validação
- ✅ Criado UpdateRequest separado
- ✅ Nomes simplificados (CreateRequest ao invés de CreateOrganizationRequest)

#### **3.5. Migrar Mappers**

**Antes** (espalhado em vários lugares):
```go
// No handler
org := &entity.Organization{
	Name: req.Name,
}

// Na resposta
response := dto.OrganizationResponse{
	ID:        org.ID,
	Name:      org.Name,
	CreatedAt: org.CreatedAt,
	UpdatedAt: org.UpdatedAt,
}
```

**Depois** (`internal/module/organization/mapper.go`):
```go
package organization

func ToResponse(org *Organization) Response {
	return Response{
		ID:        org.ID,
		Name:      org.Name,
		CreatedAt: org.CreatedAt,
		UpdatedAt: org.UpdatedAt,
	}
}

func ToListResponse(orgs []*Organization) []Response {
	dtos := make([]Response, len(orgs))
	for i, org := range orgs {
		dtos[i] = ToResponse(org)
	}
	return dtos
}

func ToEntity(req CreateRequest) *Organization {
	return &Organization{
		Name: req.Name,
	}
}

func UpdateFromDTO(org *Organization, req UpdateRequest) {
	org.Name = req.Name
}
```

**Mudanças:**
- ✅ Conversões isoladas em funções
- ✅ Reutilizáveis
- ✅ Testáveis

#### **3.6. Consolidar Use Cases**

**Antes** (5 arquivos separados):
```
internal/application/
├── create_organization.go
├── list_organizations.go
├── get_organization.go
├── update_organization.go
└── delete_organization.go
```

**Depois** (1 arquivo):
```go
// internal/module/organization/usecase.go
package organization

// ============================================================================
// CREATE USE CASE
// ============================================================================

type CreateUseCase struct {
	repo Repository
}

func NewCreateUseCase(repo Repository) *CreateUseCase {
	return &CreateUseCase{repo: repo}
}

func (uc *CreateUseCase) Execute(ctx context.Context, name string) (*Organization, error) {
	logger.Info("creating organization", "name", name)

	org := &Organization{Name: name}

	if err := uc.repo.Create(ctx, org); err != nil {
		logger.Error("failed to create organization", "name", name, "error", err)
		return nil, fmt.Errorf("failed to create organization: %w", err)
	}

	logger.Info("organization created successfully", "id", org.ID, "name", org.Name)
	return org, nil
}

// ============================================================================
// LIST USE CASE
// ============================================================================

// ... outros use cases
```

**Mudanças:**
- ✅ Consolidados em 1 arquivo
- ✅ Adicionados logs estruturados
- ✅ Error wrapping
- ✅ Separados por comentários

#### **3.7. Migrar Handler**

**Antes** (`internal/presentation/http/handler/organization_handler.go`):
```go
package handler

type OrganizationHandler struct{}

func (h *OrganizationHandler) Create(w http.ResponseWriter, r *http.Request, reqCtx *database.RequestContext) {
	var req dto.CreateOrganizationRequest
	json.NewDecoder(r.Body).Decode(&req)
	
	// Sem validação!
	repo := postgres.NewPostgresOrganizationRepository(reqCtx.TenantDB) // Acoplado!
	uc := application.NewCreateOrganization(repo)
	
	org, err := uc.Execute(r.Context(), req.Name) // Sem timeout!
	// Sem logs!
	
	respondJSON(w, http.StatusCreated, toOrganizationResponse(org))
}
```

**Depois** (`internal/module/organization/handler.go`):
```go
package organization

type Handler struct {
	repoFactory func(*database.RequestContext) Repository
}

func NewHandler(repoFactory func(*database.RequestContext) Repository) *Handler {
	return &Handler{repoFactory: repoFactory}
}

func (h *Handler) Create(w http.ResponseWriter, r *http.Request, reqCtx *database.RequestContext) {
	var req CreateRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		respondError(w, http.StatusBadRequest, "Invalid request body", err)
		return
	}

	// Validação estruturada
	if err := pkgValidator.Validate(req); err != nil {
		respondValidationError(w, err)
		return
	}

	repo := h.repoFactory(reqCtx) // Desacoplado!
	uc := NewCreateUseCase(repo)

	// Context com timeout
	ctx, cancel := withTimeout(r.Context())
	defer cancel()

	org, err := uc.Execute(ctx, req.Name)
	if err != nil {
		respondError(w, http.StatusInternalServerError, "Failed to create organization", err)
		return
	}

	respondJSON(w, http.StatusCreated, ToResponse(org))
}
```

**Mudanças:**
- ✅ Dependency Injection via factory
- ✅ Validação estruturada
- ✅ Timeout de 5 segundos
- ✅ Tratamento de erros
- ✅ HTTP Status corretos

#### **3.8. Criar Routes**

**Antes** (no router):
```go
// internal/presentation/http/router/router.go
mux.HandleFunc("GET /v1/organizations", wrap(orgHandler.List))
mux.HandleFunc("POST /v1/organizations", wrap(orgHandler.Create))
// ...
```

**Depois** (`internal/module/organization/routes.go`):
```go
package organization

func RegisterRoutes(mux *http.ServeMux, handler *Handler, wrap middleware.HandlerFunc) {
	mux.HandleFunc("GET /v1/organizations", wrap(handler.List))
	mux.HandleFunc("POST /v1/organizations", wrap(handler.Create))
	mux.HandleFunc("GET /v1/organizations/{id}", wrap(handler.GetByID))
	mux.HandleFunc("PUT /v1/organizations/{id}", wrap(handler.Update))
	mux.HandleFunc("DELETE /v1/organizations/{id}", wrap(handler.Delete))
}
```

**Mudanças:**
- ✅ Rotas isoladas no módulo
- ✅ Função RegisterRoutes para registro

### **Fase 4: Atualizar main.go**

**Antes**:
```go
orgRepo := postgres.NewPostgresOrganizationRepository(db)
orgHandler := handler.NewOrganizationHandler(orgRepo)

mux.HandleFunc("GET /v1/organizations", wrap(orgHandler.List))
mux.HandleFunc("POST /v1/organizations", wrap(orgHandler.Create))
// ...
```

**Depois**:
```go
// Organization module
orgHandler := organization.NewHandler(func(reqCtx *database.RequestContext) organization.Repository {
	return organization.NewRepository(reqCtx.TenantDB)
})
organization.RegisterRoutes(mux, orgHandler, wrap)
```

**Mudanças:**
- ✅ Factory function para DI
- ✅ Registro via RegisterRoutes
- ✅ Código mais limpo

### **Fase 5: Limpar Estrutura Antiga**

#### **5.1. Remover Pastas Antigas**

```bash
rm -rf internal/domain
rm -rf internal/application
rm -rf internal/infrastructure
rm -rf internal/presentation
```

#### **5.2. Verificar Compilação**

```bash
go mod tidy
go build ./...
```

#### **5.3. Executar Testes**

```bash
go test ./...
```

### **Fase 6: Documentação**

#### **6.1. Criar Documentação**

Crie os seguintes arquivos:

- `MODULAR_ARCHITECTURE.md` - Arquitetura modular
- `MODULE_TEMPLATE.md` - Template para novos módulos
- `MIGRATION_SUMMARY.md` - Resumo da migração

#### **6.2. Atualizar README**

Atualize o README.md com a nova estrutura.

## 📊 Checklist de Migração

Use este checklist para cada módulo:

### **Por Módulo:**

- [ ] Pasta `internal/module/[nome]/` criada
- [ ] `entity.go` migrado (sem validação, com timestamps)
- [ ] `repository.go` criado (herda BaseRepository)
- [ ] `repository_impl.go` criado (composição)
- [ ] `dto.go` criado (com validações)
- [ ] `mapper.go` criado (conversões)
- [ ] `usecase.go` criado (5 use cases consolidados, com logs)
- [ ] `handler.go` criado (DI, validação, timeout)
- [ ] `routes.go` criado (RegisterRoutes)
- [ ] Registrado no `main.go`
- [ ] Código compila
- [ ] Testes passam

### **Geral:**

- [ ] Todos os módulos migrados
- [ ] Estrutura antiga removida
- [ ] `go mod tidy` executado
- [ ] `go build ./...` sem erros
- [ ] `go test ./...` passando
- [ ] Documentação criada
- [ ] README atualizado
- [ ] Commit com mensagem descritiva

## ⏱️ Estimativa de Tempo

| Módulo | Complexidade | Tempo Estimado |
|--------|--------------|----------------|
| Simples (CRUD básico) | Baixa | 30-45 min |
| Médio (com relacionamentos) | Média | 45-60 min |
| Complexo (lógica de negócio) | Alta | 60-90 min |

**Exemplo para 10 módulos:**
- 5 simples: 5 × 40 min = 200 min (3h 20min)
- 3 médios: 3 × 50 min = 150 min (2h 30min)
- 2 complexos: 2 × 75 min = 150 min (2h 30min)
- **Total**: 500 min (~8h 20min)

## 🎯 Dicas para Migração Eficiente

### **1. Migre em Ordem**
Comece pelos módulos independentes, depois os dependentes.

### **2. Use Template**
Copie a estrutura de um módulo já migrado como template.

### **3. Teste Incrementalmente**
Teste cada módulo após migração, não espere migrar tudo.

### **4. Commit Frequente**
Faça commit após cada módulo migrado.

### **5. Documente Problemas**
Anote problemas encontrados para referência futura.

### **6. Pair Programming**
Migre em dupla para maior qualidade e velocidade.

### **7. Code Review**
Revise cada módulo antes de prosseguir.

## 🚨 Problemas Comuns e Soluções

### **Problema 1: Imports Circulares**

**Sintoma:**
```
import cycle not allowed
```

**Solução:**
Use interfaces para quebrar dependências circulares:

```go
// ❌ ERRADO
import "primeflow.solutions/service/internal/module/organization"

// ✅ CORRETO
type OrganizationRepository interface {
	FindByID(ctx context.Context, id int) (*Organization, error)
}
```

### **Problema 2: Validação Duplicada**

**Sintoma:**
Validação na entidade E no DTO.

**Solução:**
Remova validação da entidade, mantenha apenas no DTO.

### **Problema 3: Logs Faltando**

**Sintoma:**
Difícil debugar problemas em produção.

**Solução:**
Adicione logs estruturados em TODOS os use cases.

### **Problema 4: Timeout Faltando**

**Sintoma:**
Operações travadas em produção.

**Solução:**
Adicione timeout de 5 segundos em TODOS os handlers.

## ✅ Validação Final

Após migração completa, verifique:

- [ ] Projeto compila sem erros
- [ ] Todos os testes passam
- [ ] Estrutura modular clara
- [ ] Documentação completa
- [ ] Código limpo e organizado
- [ ] Logs estruturados em todos os use cases
- [ ] Timeout em todos os handlers
- [ ] Validação em todos os DTOs
- [ ] Error wrapping em todos os erros
- [ ] DI em todos os handlers

---

**Migração concluída com sucesso!** 🎉

O projeto está pronto para escalar e manter.

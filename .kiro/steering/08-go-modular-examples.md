# Go - Arquitetura Modular: Exemplos Práticos

## 🎯 Objetivo

Este documento fornece exemplos práticos e casos de uso reais da arquitetura modular.

## 📝 Exemplo Completo: Módulo Product

### **Cenário**
Criar um módulo completo para gerenciar produtos com os seguintes campos:
- ID (int)
- Name (string)
- Description (string)
- Price (float64)
- Stock (int)
- Active (bool)
- CreatedAt (time.Time)
- UpdatedAt (time.Time)

### **1. entity.go**

```go
package product

import (
	"errors"
	"time"
)

type Product struct {
	ID          int       `gorm:"primaryKey"`
	Name        string    `gorm:"size:255;not null"`
	Description string    `gorm:"type:text"`
	Price       float64   `gorm:"not null"`
	Stock       int       `gorm:"not null;default:0"`
	Active      bool      `gorm:"not null;default:true"`
	CreatedAt   time.Time `gorm:"autoCreateTime"`
	UpdatedAt   time.Time `gorm:"autoUpdateTime"`
}

func (Product) TableName() string {
	return "products.products"
}

var (
	ErrProductNotFound = errors.New("product not found")
)
```

### **2. repository.go**

```go
package product

import (
	"context"

	"gorm.io/gorm"
	pkgRepo "primeflow.solutions/pkg/repository"
)

// Repository handles product persistence
type Repository struct {
	*pkgRepo.BaseRepository[Product]
}

// NewRepository creates a new product repository instance
func NewRepository(db *gorm.DB) *Repository {
	return &Repository{
		BaseRepository: pkgRepo.NewBaseRepository[Product](db),
	}
}

// FindByID overrides base method to return custom error
func (r *Repository) FindByID(ctx context.Context, id int) (*Product, error) {
	product, err := r.BaseRepository.FindByID(ctx, id)
	if err != nil {
		if err == gorm.ErrRecordNotFound || err.Error() == "record not found" {
			return nil, ErrProductNotFound
		}
		return nil, err
	}
	return product, nil
}

// FindByName retrieves a product by name
func (r *Repository) FindByName(ctx context.Context, name string) (*Product, error) {
	var product Product
	err := r.DB().WithContext(ctx).Where("name = ?", name).First(&product).Error
	if err != nil {
		if err == gorm.ErrRecordNotFound {
			return nil, ErrProductNotFound
		}
		return nil, err
	}
	return &product, nil
}

// FindActiveProducts retrieves all active products
func (r *Repository) FindActiveProducts(ctx context.Context) ([]*Product, error) {
	var products []*Product
	err := r.DB().WithContext(ctx).Where("active = ?", true).Find(&products).Error
	return products, err
}
```

### **3. dto.go**

```go
package product

import "time"

type CreateRequest struct {
	Name        string  `json:"name" validate:"required,min=2,max=255"`
	Description string  `json:"description" validate:"omitempty,max=1000"`
	Price       float64 `json:"price" validate:"required,gt=0"`
	Stock       int     `json:"stock" validate:"required,gte=0"`
	Active      bool    `json:"active"`
}

type UpdateRequest struct {
	Name        string  `json:"name" validate:"required,min=2,max=255"`
	Description string  `json:"description" validate:"omitempty,max=1000"`
	Price       float64 `json:"price" validate:"required,gt=0"`
	Stock       int     `json:"stock" validate:"required,gte=0"`
	Active      bool    `json:"active"`
}

type Response struct {
	ID          int       `json:"id"`
	Name        string    `json:"name"`
	Description string    `json:"description"`
	Price       float64   `json:"price"`
	Stock       int       `json:"stock"`
	Active      bool      `json:"active"`
	CreatedAt   time.Time `json:"createdAt"`
	UpdatedAt   time.Time `json:"updatedAt"`
}
```

### **4. mapper.go**

```go
package product

func ToResponse(product *Product) Response {
	return Response{
		ID:          product.ID,
		Name:        product.Name,
		Description: product.Description,
		Price:       product.Price,
		Stock:       product.Stock,
		Active:      product.Active,
		CreatedAt:   product.CreatedAt,
		UpdatedAt:   product.UpdatedAt,
	}
}

func ToListResponse(products []*Product) []Response {
	dtos := make([]Response, len(products))
	for i, product := range products {
		dtos[i] = ToResponse(product)
	}
	return dtos
}

func ToEntity(req CreateRequest) *Product {
	return &Product{
		Name:        req.Name,
		Description: req.Description,
		Price:       req.Price,
		Stock:       req.Stock,
		Active:      req.Active,
	}
}

func UpdateFromDTO(product *Product, req UpdateRequest) {
	product.Name = req.Name
	product.Description = req.Description
	product.Price = req.Price
	product.Stock = req.Stock
	product.Active = req.Active
}
```

### **5. usecase.go** (apenas Create como exemplo)

```go
package product

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
	repo *Repository
}

func NewCreateUseCase(repo *Repository) *CreateUseCase {
	return &CreateUseCase{repo: repo}
}

func (uc *CreateUseCase) Execute(ctx context.Context, req CreateRequest) (*Product, error) {
	logger.Info("creating product", "name", req.Name, "price", req.Price)

	// Verificar se produto com mesmo nome já existe
	existing, err := uc.repo.FindByName(ctx, req.Name)
	if err == nil && existing != nil {
		logger.Error("product with same name already exists", "name", req.Name)
		return nil, fmt.Errorf("product with name '%s' already exists", req.Name)
	}

	product := ToEntity(req)

	if err := uc.repo.Create(ctx, product); err != nil {
		logger.Error("failed to create product", "name", req.Name, "error", err)
		return nil, fmt.Errorf("failed to create product: %w", err)
	}

	logger.Info("product created successfully", "id", product.ID, "name", product.Name, "price", product.Price)
	return product, nil
}

// ... outros use cases (List, Get, Update, Delete)
```

### **6. handler.go** (apenas Create como exemplo)

```go
package product

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

const DefaultTimeout = 5 * time.Second

type Handler struct {
	repoFactory func(*database.RequestContext) *Repository
}

func NewHandler(repoFactory func(*database.RequestContext) *Repository) *Handler {
	return &Handler{repoFactory: repoFactory}
}

var AllowedFields = map[string]string{
	"id":          "id",
	"name":        "name",
	"price":       "price",
	"stock":       "stock",
	"active":      "active",
	"createdAt":   "created_at",
	"updatedAt":   "updated_at",
}

func withTimeout(ctx context.Context) (context.Context, context.CancelFunc) {
	return context.WithTimeout(ctx, DefaultTimeout)
}

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

	product, err := uc.Execute(ctx, req)
	if err != nil {
		// Verificar se é erro de duplicação
		if err.Error() == fmt.Sprintf("product with name '%s' already exists", req.Name) {
			respondError(w, http.StatusConflict, "Product already exists", err)
			return
		}
		respondError(w, http.StatusInternalServerError, "Failed to create product", err)
		return
	}

	respondJSON(w, http.StatusCreated, ToResponse(product))
}

// ... outros handlers (List, GetByID, Update, Delete)

// HTTP Helpers
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

### **7. routes.go**

```go
package product

import (
	"net/http"

	"primeflow.solutions/pkg/middleware"
)

func RegisterRoutes(mux *http.ServeMux, handler *Handler, wrap middleware.HandlerFunc) {
	mux.HandleFunc("GET /v1/products", wrap(handler.List))
	mux.HandleFunc("POST /v1/products", wrap(handler.Create))
	mux.HandleFunc("GET /v1/products/{id}", wrap(handler.GetByID))
	mux.HandleFunc("PUT /v1/products/{id}", wrap(handler.Update))
	mux.HandleFunc("DELETE /v1/products/{id}", wrap(handler.Delete))
}
```

### **8. Registro no main.go**

```go
// Product module
productHandler := product.NewHandler(func(reqCtx *database.RequestContext) product.Repository {
	return product.NewRepository(reqCtx.TenantDB)
})
product.RegisterRoutes(mux, productHandler, wrap)
```

## 🔗 Exemplo: Módulo com Relacionamento

### **Cenário: Order (Pedido) que pertence a Customer**

```go
// internal/module/order/entity.go
package order

type Order struct {
	ID         int       `gorm:"primaryKey"`
	CustomerID int       `gorm:"not null;index"`
	TotalPrice float64   `gorm:"not null"`
	Status     string    `gorm:"size:50;not null"`
	CreatedAt  time.Time `gorm:"autoCreateTime"`
	UpdatedAt  time.Time `gorm:"autoUpdateTime"`
}

// internal/module/order/usecase.go
type CreateUseCase struct {
	orderRepo    *Repository
	customerRepo *customer.Repository // Dependência de outro módulo
}

func NewCreateUseCase(orderRepo *Repository, customerRepo *customer.Repository) *CreateUseCase {
	return &CreateUseCase{
		orderRepo:    orderRepo,
		customerRepo: customerRepo,
	}
}

func (uc *CreateUseCase) Execute(ctx context.Context, req CreateRequest) (*Order, error) {
	logger.Info("creating order", "customerId", req.CustomerID)

	// Verificar se customer existe
	_, err := uc.customerRepo.FindByID(ctx, req.CustomerID)
	if err != nil {
		logger.Error("customer not found for order", "customerId", req.CustomerID, "error", err)
		return nil, fmt.Errorf("customer not found: %w", err)
	}

	order := ToEntity(req)

	if err := uc.orderRepo.Create(ctx, order); err != nil {
		logger.Error("failed to create order", "customerId", req.CustomerID, "error", err)
		return nil, fmt.Errorf("failed to create order: %w", err)
	}

	logger.Info("order created successfully", "id", order.ID, "customerId", order.CustomerID)
	return order, nil
}

// internal/module/order/handler.go
type Handler struct {
	orderRepoFactory    func(*database.RequestContext) *Repository
	customerRepoFactory func(*database.RequestContext) *customer.Repository
}

func NewHandler(
	orderRepoFactory func(*database.RequestContext) *Repository,
	customerRepoFactory func(*database.RequestContext) *customer.Repository,
) *Handler {
	return &Handler{
		orderRepoFactory:    orderRepoFactory,
		customerRepoFactory: customerRepoFactory,
	}
}

func (h *Handler) Create(w http.ResponseWriter, r *http.Request, reqCtx *database.RequestContext) {
	// ... validação ...

	orderRepo := h.orderRepoFactory(reqCtx)
	customerRepo := h.customerRepoFactory(reqCtx)
	uc := NewCreateUseCase(orderRepo, customerRepo)

	// ... execução ...
}

// cmd/main.go
orderHandler := order.NewHandler(
	func(reqCtx *database.RequestContext) order.Repository {
		return order.NewRepository(reqCtx.TenantDB)
	},
	func(reqCtx *database.RequestContext) customer.Repository {
		return customer.NewRepository(reqCtx.TenantDB)
	},
)
order.RegisterRoutes(mux, orderHandler, wrap)
```

## 📊 Exemplo: Paginação e Filtros

### **Requisição com filtros:**

```
GET /v1/products?page=1&pageSize=10&filter=price>100 and active=true&orderBy=price:DESC&displayFields=id,name,price
```

### **Handler:**

```go
func (h *Handler) List(w http.ResponseWriter, r *http.Request, reqCtx *database.RequestContext) {
	// ParseQueryParams já faz o parsing de page, pageSize, filter, orderBy, displayFields
	req, err := pagination.ParseQueryParams(r)
	if err != nil {
		respondError(w, http.StatusBadRequest, "Invalid query parameters", err)
		return
	}

	repo := h.repoFactory(reqCtx)
	uc := NewListUseCase(repo)

	ctx, cancel := withTimeout(r.Context())
	defer cancel()

	// AllowedFields define quais campos podem ser filtrados/ordenados
	products, total, err := uc.Execute(ctx, req, AllowedFields)
	if err != nil {
		respondError(w, http.StatusInternalServerError, "Failed to list products", err)
		return
	}

	dtos := ToListResponse(products)
	
	// BuildResponse cria resposta paginada com metadata
	response := pagination.BuildResponse(dtos, total, req.PageSize)
	respondJSON(w, http.StatusOK, response)
}
```

### **Resposta:**

```json
{
  "data": [
    {
      "id": 1,
      "name": "Product A",
      "price": 150.00
    },
    {
      "id": 2,
      "name": "Product B",
      "price": 120.00
    }
  ],
  "metadata": {
    "page": 1,
    "pageSize": 10,
    "totalElements": 25,
    "totalPages": 3
  }
}
```

## 🔒 Exemplo: Validações Customizadas

### **Validação de CNPJ Alfanumérico:**

```go
// internal/shared/validator/cnpj.go
package validator

import (
	"regexp"

	"github.com/go-playground/validator/v10"
)

var cnpjRegex = regexp.MustCompile(`^[A-Z0-9]{12}[0-9]{2}$`)

func ValidateCNPJ(fl validator.FieldLevel) bool {
	cnpj := fl.Field().String()
	return cnpjRegex.MatchString(cnpj)
}

// Registrar no init do pkg/validator
func init() {
	pkgValidator.RegisterValidation("cnpj", ValidateCNPJ)
}

// Uso no DTO
type CreateRequest struct {
	Name  string `json:"name" validate:"required,min=2,max=255"`
	TaxID string `json:"taxId" validate:"required,cnpj"`
}
```

## 🧪 Exemplo: Testes

### **Teste de Use Case:**

```go
// internal/module/product/usecase_test.go
package product_test

import (
	"context"
	"testing"

	"primeflow.solutions/service/internal/module/product"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/mock"
)

type MockRepository struct {
	mock.Mock
}

func (m *MockRepository) Create(ctx context.Context, p *product.Product) error {
	args := m.Called(ctx, p)
	return args.Error(0)
}

func (m *MockRepository) FindByName(ctx context.Context, name string) (*product.Product, error) {
	args := m.Called(ctx, name)
	if args.Get(0) == nil {
		return nil, args.Error(1)
	}
	return args.Get(0).(*product.Product), args.Error(1)
}

func TestCreateUseCase_Execute_Success(t *testing.T) {
	// Arrange
	mockRepo := new(MockRepository)
	uc := product.NewCreateUseCase(mockRepo)
	
	req := product.CreateRequest{
		Name:  "Test Product",
		Price: 100.0,
		Stock: 10,
	}

	mockRepo.On("FindByName", mock.Anything, req.Name).Return(nil, product.ErrProductNotFound)
	mockRepo.On("Create", mock.Anything, mock.AnythingOfType("*product.Product")).Return(nil)

	// Act
	result, err := uc.Execute(context.Background(), req)

	// Assert
	assert.NoError(t, err)
	assert.NotNil(t, result)
	assert.Equal(t, req.Name, result.Name)
	assert.Equal(t, req.Price, result.Price)
	mockRepo.AssertExpectations(t)
}

func TestCreateUseCase_Execute_DuplicateName(t *testing.T) {
	// Arrange
	mockRepo := new(MockRepository)
	uc := product.NewCreateUseCase(mockRepo)
	
	req := product.CreateRequest{
		Name:  "Existing Product",
		Price: 100.0,
		Stock: 10,
	}

	existing := &product.Product{ID: 1, Name: req.Name}
	mockRepo.On("FindByName", mock.Anything, req.Name).Return(existing, nil)

	// Act
	result, err := uc.Execute(context.Background(), req)

	// Assert
	assert.Error(t, err)
	assert.Nil(t, result)
	assert.Contains(t, err.Error(), "already exists")
	mockRepo.AssertExpectations(t)
}
```

## 📝 Exemplo: Soft Delete

### **Entity com Soft Delete:**

```go
type Product struct {
	ID        int            `gorm:"primaryKey"`
	Name      string         `gorm:"size:255;not null"`
	DeletedAt gorm.DeletedAt `gorm:"index"` // Soft delete
	CreatedAt time.Time      `gorm:"autoCreateTime"`
	UpdatedAt time.Time      `gorm:"autoUpdateTime"`
}
```

### **Repository com Soft Delete:**

```go
func (r *repository) Delete(ctx context.Context, id int) error {
	// GORM automaticamente faz soft delete se DeletedAt existir
	return r.DB().WithContext(ctx).Delete(&Product{}, id).Error
}

func (r *repository) HardDelete(ctx context.Context, id int) error {
	// Forçar hard delete
	return r.DB().WithContext(ctx).Unscoped().Delete(&Product{}, id).Error
}

func (r *repository) FindWithDeleted(ctx context.Context, id int) (*Product, error) {
	var product Product
	err := r.DB().WithContext(ctx).Unscoped().First(&product, id).Error
	if err != nil {
		if err == gorm.ErrRecordNotFound {
			return nil, ErrProductNotFound
		}
		return nil, err
	}
	return &product, nil
}
```

## 🎯 Resumo de Boas Práticas

### ✅ SEMPRE faça:
1. Logs estruturados em todos os use cases
2. Error wrapping com contexto
3. Timeout de 5 segundos em handlers
4. Validação estruturada nos DTOs
5. Dependency Injection via factory
6. Timestamps automáticos nas entidades
7. Erros customizados por módulo
8. HTTP Status corretos
9. AllowedFields para paginação
10. Verificar existência antes de Delete

### ❌ NUNCA faça:
1. Validação na entidade
2. Lógica de negócio no handler
3. Dependências diretas entre módulos
4. Esquecer logs
5. Esquecer timeout
6. Esquecer error wrapping
7. Retornar erro genérico
8. Expor detalhes internos na API
9. Hardcode de valores
10. Ignorar erros

---

**Use estes exemplos como referência ao criar novos módulos!**

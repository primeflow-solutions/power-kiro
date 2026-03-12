# API Documentation

## Base URLs

- **Local Development**: `http://api.localhost`
- **Production**: `https://api.primeflow.solutions`

## Authentication

### JWT Token

All authenticated endpoints require a JWT token in the Authorization header:

```
Authorization: Bearer <token>
```

### Token Structure

```json
{
  "sub": "123",           // user_id
  "tenant_id": 1,         // tenant_id
  "tenant_slug": "acme",  // tenant database name
  "iat": 1234567890,      // issued at
  "exp": 1234567890       // expires at (7 days)
}
```

### Headers Injected by Gateway

The gateway automatically injects these headers for authenticated requests:

```
X-Tenant-ID: 1
X-Tenant-Slug: acme_corp
X-User-ID: 123
```

## Public Endpoints

### Tenant Service

#### POST /tenant/v1/signup
Create a new tenant (self-service registration).

**Request:**
```json
{
  "company_name": "Acme Corporation",
  "email": "admin@acme.com",
  "domain": "acme.com",
  "admin_name": "John Doe",
  "password": "SecurePass123"
}
```

**Response:** (201 Created)
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": 1,
    "email": "admin@acme.com",
    "first_name": "John",
    "last_name": "Doe",
    "active": true
  },
  "tenant": {
    "id": 1,
    "name": "Acme Corporation",
    "slug": "acme_corp",
    "database_name": "acme_corp"
  }
}
```

**Errors:**
- `400`: Validation error (missing fields, weak password)
- `409`: Domain already registered
- `500`: Server error

---

### Users Service

#### POST /users/v1/login
Authenticate user and get JWT token.

**Request:**
```json
{
  "email": "admin@acme.com",
  "password": "SecurePass123"
}
```

**Response:** (200 OK)
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": 1,
    "email": "admin@acme.com",
    "first_name": "John",
    "last_name": "Doe",
    "active": true
  },
  "tenant": {
    "id": 1,
    "name": "Acme Corporation",
    "slug": "acme_corp",
    "database_name": "acme_corp"
  }
}
```

**Errors:**
- `400`: Invalid request body
- `401`: Invalid credentials or inactive user
- `404`: Tenant not found for email domain
- `500`: Server error

---

## Authenticated Endpoints

### Users Service

#### GET /users/v1/users
List all users in the authenticated tenant.

**Headers:**
```
Authorization: Bearer <token>
```

**Response:** (200 OK)
```json
[
  {
    "id": 1,
    "email": "admin@acme.com",
    "first_name": "John",
    "last_name": "Doe",
    "active": true,
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-01T00:00:00Z"
  },
  {
    "id": 2,
    "email": "user@acme.com",
    "first_name": "Jane",
    "last_name": "Smith",
    "active": true,
    "created_at": "2024-01-02T00:00:00Z",
    "updated_at": "2024-01-02T00:00:00Z"
  }
]
```

**Errors:**
- `401`: Missing or invalid token
- `500`: Server error

---

#### POST /users/v1/users
Create a new user in the authenticated tenant.

**Headers:**
```
Authorization: Bearer <token>
```

**Request:**
```json
{
  "email": "newuser@acme.com",
  "first_name": "Alice",
  "last_name": "Johnson",
  "password": "SecurePass123"
}
```

**Response:** (201 Created)
```json
{
  "id": 3,
  "email": "newuser@acme.com",
  "first_name": "Alice",
  "last_name": "Johnson",
  "active": true,
  "created_at": "2024-01-03T00:00:00Z",
  "updated_at": "2024-01-03T00:00:00Z"
}
```

**Errors:**
- `400`: Validation error (invalid email, weak password, name too short)
- `401`: Missing or invalid token
- `409`: Email already exists
- `500`: Server error

---

#### PUT /users/v1/users/{id}
Update an existing user.

**Headers:**
```
Authorization: Bearer <token>
```

**Request:**
```json
{
  "email": "newemail@acme.com",
  "first_name": "Alice",
  "last_name": "Johnson-Smith",
  "password": "NewSecurePass123",
  "active": false
}
```

**Note:** All fields are optional. Only provided fields will be updated.

**Response:** (200 OK)
```json
{
  "id": 3,
  "email": "newemail@acme.com",
  "first_name": "Alice",
  "last_name": "Johnson-Smith",
  "active": false,
  "created_at": "2024-01-03T00:00:00Z",
  "updated_at": "2024-01-03T10:00:00Z"
}
```

**Errors:**
- `400`: Validation error or invalid user ID
- `401`: Missing or invalid token
- `404`: User not found
- `500`: Server error

---

#### DELETE /users/v1/users/{id}
Delete a user.

**Headers:**
```
Authorization: Bearer <token>
```

**Response:** (204 No Content)

**Errors:**
- `400`: Invalid user ID
- `401`: Missing or invalid token
- `404`: User not found
- `500`: Server error

---

### Tenant Service

#### GET /tenant/v1/tenants
List all tenants (administrative endpoint).

**Headers:**
```
Authorization: Bearer <token>
```

**Response:** (200 OK)
```json
[
  {
    "id": 1,
    "slug": "acme_corp",
    "name": "Acme Corporation",
    "database_name": "acme_corp",
    "active": true,
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-01T00:00:00Z"
  }
]
```

---

#### POST /tenant/v1/tenants
Create a new tenant (administrative endpoint).

**Headers:**
```
Authorization: Bearer <token>
```

**Request:**
```json
{
  "name": "New Company",
  "slug": "new_company"
}
```

**Response:** (201 Created)
```json
{
  "id": 2,
  "slug": "new_company",
  "name": "New Company",
  "database_name": "new_company",
  "active": true,
  "created_at": "2024-01-04T00:00:00Z",
  "updated_at": "2024-01-04T00:00:00Z"
}
```

---

#### GET /tenant/v1/tenants/{id}
Get tenant details.

**Headers:**
```
Authorization: Bearer <token>
```

**Response:** (200 OK)
```json
{
  "id": 1,
  "slug": "acme_corp",
  "name": "Acme Corporation",
  "database_name": "acme_corp",
  "active": true,
  "created_at": "2024-01-01T00:00:00Z",
  "updated_at": "2024-01-01T00:00:00Z"
}
```

---

#### PUT /tenant/v1/tenants/{id}
Update tenant details.

**Headers:**
```
Authorization: Bearer <token>
```

**Request:**
```json
{
  "name": "Acme Corporation Inc.",
  "active": true
}
```

**Response:** (200 OK)
```json
{
  "id": 1,
  "slug": "acme_corp",
  "name": "Acme Corporation Inc.",
  "database_name": "acme_corp",
  "active": true,
  "created_at": "2024-01-01T00:00:00Z",
  "updated_at": "2024-01-04T10:00:00Z"
}
```

---

#### DELETE /tenant/v1/tenants/{id}
Delete a tenant.

**Headers:**
```
Authorization: Bearer <token>
```

**Response:** (204 No Content)

**Note:** This does NOT delete the tenant database. Manual cleanup required.

---

#### GET /tenant/v1/tenants/{id}/domains
List email domains for a tenant.

**Headers:**
```
Authorization: Bearer <token>
```

**Response:** (200 OK)
```json
[
  {
    "id": 1,
    "tenant_id": 1,
    "domain": "acme.com",
    "created_at": "2024-01-01T00:00:00Z"
  },
  {
    "id": 2,
    "tenant_id": 1,
    "domain": "acmecorp.com",
    "created_at": "2024-01-02T00:00:00Z"
  }
]
```

---

#### POST /tenant/v1/tenants/{id}/domains
Add an email domain to a tenant.

**Headers:**
```
Authorization: Bearer <token>
```

**Request:**
```json
{
  "domain": "acme.io"
}
```

**Response:** (201 Created)
```json
{
  "id": 3,
  "tenant_id": 1,
  "domain": "acme.io",
  "created_at": "2024-01-04T00:00:00Z"
}
```

---

#### DELETE /tenant/v1/tenants/{tenant_id}/domains/{domain_id}
Remove an email domain from a tenant.

**Headers:**
```
Authorization: Bearer <token>
```

**Response:** (204 No Content)

---

## Health Check Endpoints

### GET /v1/health
Check service health (all services).

**Response:** (200 OK)
```json
{
  "status": "healthy",
  "timestamp": "2024-01-04T10:00:00Z"
}
```

---

## Error Response Format

All errors follow this format:

```json
{
  "error": "Error Title",
  "message": "Detailed error message"
}
```

### Common HTTP Status Codes

- `200 OK`: Successful GET/PUT request
- `201 Created`: Successful POST request
- `204 No Content`: Successful DELETE request
- `400 Bad Request`: Invalid request data
- `401 Unauthorized`: Missing or invalid authentication
- `404 Not Found`: Resource not found
- `409 Conflict`: Resource already exists
- `500 Internal Server Error`: Server error

---

## Rate Limiting

Currently not implemented. Consider adding rate limiting for production.

---

## API Versioning

All endpoints are versioned with `/v1/` in the path. Future versions will use `/v2/`, etc.

---

## CORS

CORS is configured in nginx.conf:
- Allowed Origins: `*` (development), specific domains (production)
- Allowed Methods: GET, POST, PUT, DELETE, OPTIONS
- Allowed Headers: Content-Type, Authorization, X-Tenant-Slug

---

## Testing with cURL

### Signup
```bash
curl -X POST http://api.localhost/tenant/v1/signup \
  -H "Content-Type: application/json" \
  -d '{
    "company_name": "Test Corp",
    "email": "admin@test.com",
    "domain": "test.com",
    "admin_name": "Admin User",
    "password": "Test1234"
  }'
```

### Login
```bash
curl -X POST http://api.localhost/users/v1/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@test.com",
    "password": "Test1234"
  }'
```

### List Users (with token)
```bash
TOKEN="<your-jwt-token>"
curl -X GET http://api.localhost/users/v1/users \
  -H "Authorization: Bearer $TOKEN"
```

### Create User
```bash
TOKEN="<your-jwt-token>"
curl -X POST http://api.localhost/users/v1/users \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "newuser@test.com",
    "first_name": "New",
    "last_name": "User",
    "password": "Pass1234"
  }'
```

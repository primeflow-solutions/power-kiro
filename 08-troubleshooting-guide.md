# Troubleshooting Guide

## Common Issues and Solutions

---

## Authentication Issues

### Issue: "Invalid or expired token"

**Symptoms:**
- 401 Unauthorized errors
- Token validation fails
- User logged out unexpectedly

**Possible Causes:**
1. Token expired (7 days)
2. JWT_SIGNING_KEY mismatch between services
3. Token not being sent in requests

**Solutions:**

1. **Check token expiration:**
```bash
# Decode JWT token (use jwt.io)
# Check 'exp' claim
```

2. **Verify JWT_SIGNING_KEY:**
```bash
# Check gateway
docker exec primeflow-gateway env | grep JWT_SIGNING_KEY

# Check users service
docker exec primeflow-users env | grep JWT_SIGNING_KEY

# They must match!
```

3. **Check browser cookies:**
```javascript
// Open browser console
document.cookie
// Should see: auth_token=...
```

4. **Check interceptor:**
```typescript
// Verify AuthInterceptor is adding token
console.log('Token:', this.authService.getToken());
```

---

### Issue: "Missing X-Tenant-Slug header"

**Symptoms:**
- 401 Unauthorized
- Error message: "missing X-Tenant-Slug header"

**Possible Causes:**
1. JWT token doesn't contain tenant_slug
2. Gateway not injecting header
3. Old token without tenant_slug

**Solutions:**

1. **Logout and login again:**
```typescript
// This generates new token with tenant_slug
authService.logout();
// Login again
```

2. **Check JWT claims:**
```bash
# Decode token at jwt.io
# Should contain:
{
  "sub": "123",
  "tenant_id": 1,
  "tenant_slug": "acme_corp"  // Must be present!
}
```

3. **Verify gateway middleware:**
```bash
# Check gateway logs
docker logs primeflow-gateway | grep "tenant_slug"
```

---

## Database Issues

### Issue: "Failed to connect to database"

**Symptoms:**
- Services fail to start
- Database connection errors
- Timeout errors

**Possible Causes:**
1. Database credentials incorrect
2. Database not accessible
3. Network issues
4. Firewall blocking connection

**Solutions:**

1. **Test database connection:**
```bash
# From your machine
psql -h trolley.proxy.rlwy.net -p 11306 -U postgres -d shared

# From container
docker exec -it primeflow-users sh
ping trolley.proxy.rlwy.net
```

2. **Check credentials:**
```bash
# Verify environment variables
docker exec primeflow-users env | grep DB_
```

3. **Check Railway dashboard:**
- Verify database is running
- Check connection details
- Review database logs

---

### Issue: "Relation does not exist"

**Symptoms:**
- Error: `relation "users.users" does not exist`
- Error: `relation "tenant.tenants" does not exist`

**Possible Causes:**
1. Migrations not run
2. Wrong database
3. Schema not created

**Solutions:**

1. **Check if migrations ran:**
```sql
-- Connect to database
\c shared
SELECT * FROM tenant.migrations;

-- Connect to tenant database
\c primeflow_solutions
SELECT * FROM users.migrations;
```

2. **Manually run migrations:**
```bash
# Restart tenant service (runs migrations on startup)
docker-compose restart tenant

# Or create new tenant (runs migrations)
# Signup at http://app.localhost/auth/signup
```

3. **Verify schema exists:**
```sql
-- List schemas
\dn

-- Should see: tenant, users
```

---

### Issue: "Connection pool exhausted"

**Symptoms:**
- Slow responses
- Timeout errors
- "Too many connections" error

**Possible Causes:**
1. Connection leaks
2. Pool too small
3. Long-running queries

**Solutions:**

1. **Check pool configuration:**
```go
// In database/manager.go
sqlDB.SetMaxIdleConns(10)
sqlDB.SetMaxOpenConns(100)
```

2. **Monitor connections:**
```sql
-- Check active connections
SELECT count(*) FROM pg_stat_activity;

-- Check by database
SELECT datname, count(*) 
FROM pg_stat_activity 
GROUP BY datname;
```

3. **Restart services:**
```bash
docker-compose restart users tenant
```

---

## Frontend Issues

### Issue: "CORS error"

**Symptoms:**
- Browser console: "CORS policy blocked"
- Preflight OPTIONS request fails
- API calls fail from browser

**Possible Causes:**
1. Nginx CORS headers missing
2. Wrong API URL
3. Credentials not allowed

**Solutions:**

1. **Check nginx.conf:**
```nginx
add_header 'Access-Control-Allow-Origin' '*' always;
add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
add_header 'Access-Control-Allow-Headers' 'Content-Type, Authorization' always;
```

2. **Restart nginx:**
```bash
docker-compose restart nginx
```

3. **Check API URL:**
```typescript
// environment.dev.ts
apiUrl: 'http://api.localhost'  // Must match nginx config
```

---

### Issue: "Cookie not persisting after F5"

**Symptoms:**
- User logged out after page refresh
- Cookies disappear
- Token lost

**Possible Causes:**
1. Cookie domain mismatch
2. Cookie not being saved
3. Browser blocking cookies

**Possible Solutions:**

1. **Check cookie domain:**
```typescript
// CookieService should detect localhost
const isLocalhost = hostname.endsWith('.localhost');
// Don't set domain for localhost
```

2. **Check browser cookies:**
```
Application → Cookies → http://app.localhost
Should see: auth_token, user_data, backend_user, tenant_data
```

3. **Clear cookies and login again:**
```javascript
// Browser console
document.cookie.split(';').forEach(c => {
  document.cookie = c.trim().split('=')[0] + '=;expires=Thu, 01 Jan 1970 00:00:00 GMT';
});
```

---

### Issue: "Angular compilation errors"

**Symptoms:**
- TypeScript errors
- Build fails
- Red errors in console

**Solutions:**

1. **Check logs:**
```bash
docker logs primeflow-webapp
```

2. **Rebuild container:**
```bash
docker-compose build webapp
docker-compose up webapp
```

3. **Clear node_modules:**
```bash
cd ts-webapp-frontend
rm -rf node_modules
npm install
```

---

## Docker Issues

### Issue: "Port already in use"

**Symptoms:**
- Error: "bind: address already in use"
- Container won't start

**Solutions:**

1. **Find process using port:**
```bash
lsof -i :80
```

2. **Kill process:**
```bash
kill -9 <PID>
```

3. **Change port:**
```yaml
# docker-compose.yml
ports:
  - "8080:80"  # Use port 8080 instead
```

---

### Issue: "Container keeps restarting"

**Symptoms:**
- Container status: Restarting
- Service unavailable

**Solutions:**

1. **Check logs:**
```bash
docker logs primeflow-gateway
```

2. **Check health:**
```bash
docker inspect primeflow-gateway | grep Health
```

3. **Remove and recreate:**
```bash
docker-compose down
docker-compose up --build
```

---

### Issue: "Build fails"

**Symptoms:**
- Docker build errors
- Dependency errors
- Compilation errors

**Solutions:**

1. **Clean build cache:**
```bash
docker builder prune
```

2. **Remove all images:**
```bash
docker-compose down --rmi all
```

3. **Rebuild from scratch:**
```bash
docker-compose build --no-cache
docker-compose up
```

---

## Multi-Tenant Issues

### Issue: "User sees wrong tenant data"

**Symptoms:**
- User sees data from another tenant
- Cross-tenant data leak

**Possible Causes:**
1. Tenant context not enforced
2. Wrong database connection
3. Middleware not working

**Solutions:**

1. **Check X-Tenant-Slug header:**
```bash
# Check gateway logs
docker logs primeflow-gateway | grep "X-Tenant-Slug"
```

2. **Verify database connection:**
```go
// Handler should use reqCtx.TenantDB
userRepo := repository.NewPostgresUserRepository(reqCtx.TenantDB)
```

3. **Check middleware:**
```go
// Middleware must extract X-Tenant-Slug
tenantSlug := r.Header.Get("X-Tenant-Slug")
```

---

### Issue: "Tenant database not found"

**Symptoms:**
- Error: "database does not exist"
- Login fails for tenant

**Possible Causes:**
1. Database not provisioned
2. Wrong database name
3. Signup failed

**Solutions:**

1. **Check tenant record:**
```sql
SELECT * FROM tenant.tenants WHERE slug = 'acme_corp';
```

2. **Check database exists:**
```sql
SELECT datname FROM pg_database WHERE datname = 'acme_corp';
```

3. **Re-run signup:**
```bash
# Delete tenant and signup again
# This will provision database
```

---

## Performance Issues

### Issue: "Slow API responses"

**Symptoms:**
- High latency
- Timeout errors
- Slow page loads

**Possible Causes:**
1. Slow database queries
2. Connection pool exhausted
3. Network latency
4. Missing indexes

**Solutions:**

1. **Check slow queries:**
```sql
-- Enable slow query log
ALTER DATABASE shared SET log_min_duration_statement = 200;

-- View slow queries
SELECT * FROM pg_stat_statements 
ORDER BY mean_exec_time DESC 
LIMIT 10;
```

2. **Add indexes:**
```sql
CREATE INDEX idx_users_email ON users.users(email);
CREATE INDEX idx_tenants_slug ON tenant.tenants(slug);
```

3. **Monitor connection pool:**
```bash
# Check logs for pool warnings
docker logs primeflow-users | grep "pool"
```

---

### Issue: "High memory usage"

**Symptoms:**
- Container using too much memory
- System slow
- OOM errors

**Solutions:**

1. **Check memory usage:**
```bash
docker stats
```

2. **Set memory limits:**
```yaml
# docker-compose.yml
deploy:
  resources:
    limits:
      memory: 512M
```

3. **Optimize connection pool:**
```go
sqlDB.SetMaxIdleConns(5)   // Reduce from 10
sqlDB.SetMaxOpenConns(50)  // Reduce from 100
```

---

## Debugging Tips

### Enable Debug Logging

```yaml
# docker-compose.yml
environment:
  - LOG_LEVEL=debug
```

### Check Service Health

```bash
# Gateway
curl http://api.localhost/v1/health

# Users
curl http://api.localhost/users/v1/health

# Tenant
curl http://api.localhost/tenant/v1/health
```

### Inspect JWT Token

1. Copy token from browser cookies
2. Go to https://jwt.io
3. Paste token
4. Verify claims

### Test API with cURL

```bash
# Login
curl -X POST http://api.localhost/users/v1/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@test.com","password":"Test1234"}'

# List users (with token)
TOKEN="your-token-here"
curl -X GET http://api.localhost/users/v1/users \
  -H "Authorization: Bearer $TOKEN"
```

### Monitor Database

```sql
-- Active queries
SELECT pid, query, state, query_start 
FROM pg_stat_activity 
WHERE state != 'idle';

-- Database size
SELECT pg_database.datname, 
       pg_size_pretty(pg_database_size(pg_database.datname)) AS size
FROM pg_database;

-- Table sizes
SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

---

## Getting Help

### Check Logs First

```bash
# All services
docker-compose logs

# Specific service
docker-compose logs gateway

# Follow logs
docker-compose logs -f users
```

### Collect Debug Information

```bash
# Service status
docker-compose ps

# Container details
docker inspect primeflow-gateway

# Network details
docker network inspect primeflowsolutions_primeflow-network

# Environment variables
docker exec primeflow-gateway env
```

### Common Log Patterns

**Success:**
```
✅ Auth state loaded successfully
✅ Login completed successfully
✅ Migrations completed successfully
```

**Errors:**
```
❌ Error loading auth state
❌ Login error
❌ Failed to connect to database
```

**Warnings:**
```
⚠️ Auth state not loaded
⚠️ Cookie not saved correctly
⚠️ Slow query detected
```

---

## Prevention

### Best Practices

1. **Always check logs first**
2. **Use health checks**
3. **Monitor database connections**
4. **Keep services updated**
5. **Test in development first**
6. **Use proper error handling**
7. **Implement retry logic**
8. **Set resource limits**
9. **Use connection pooling**
10. **Monitor performance**

### Regular Maintenance

```bash
# Weekly
docker system prune
docker volume prune

# Monthly
docker-compose down
docker-compose build --no-cache
docker-compose up

# Database
VACUUM ANALYZE;
REINDEX DATABASE shared;
```

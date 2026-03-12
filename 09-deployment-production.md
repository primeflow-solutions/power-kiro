# Production Deployment Guide

## Overview

This guide covers deploying PrimeFlow Solutions to production environments.

---

## Pre-Deployment Checklist

### Security
- [ ] Change all default passwords
- [ ] Generate strong JWT_SIGNING_KEY
- [ ] Enable SSL/TLS
- [ ] Configure firewall rules
- [ ] Set up secrets management
- [ ] Enable database encryption
- [ ] Configure CORS properly
- [ ] Implement rate limiting
- [ ] Enable security headers
- [ ] Set up monitoring

### Configuration
- [ ] Update environment variables
- [ ] Configure production database
- [ ] Set up backup strategy
- [ ] Configure logging
- [ ] Set resource limits
- [ ] Configure health checks
- [ ] Set up load balancing
- [ ] Configure CDN
- [ ] Set up DNS
- [ ] Configure SSL certificates

### Testing
- [ ] Run all tests
- [ ] Load testing
- [ ] Security testing
- [ ] Backup/restore testing
- [ ] Failover testing
- [ ] Performance testing

---

## Environment Configuration

### Production Environment Variables

#### Gateway Service
```bash
PORT=8080
JWT_SIGNING_KEY=<strong-random-key-64-chars>
JWT_EXPIRATION=168h
USER_SERVICE_URL=http://users:8081
TENANT_SERVICE_URL=http://tenant:8082
PUBLIC_ROUTES=/users/v1/login,/tenant/v1/signup,/v1/health
LOG_LEVEL=info
CORS_ALLOWED_ORIGINS=https://app.primeflow.solutions,https://primeflow.solutions
```

#### Users Service
```bash
PORT=8081
DB_HOST=<production-db-host>
DB_PORT=5432
DB_USERNAME=<db-user>
DB_PASSWORD=<strong-password>
JWT_SIGNING_KEY=<same-as-gateway>
JWT_EXPIRATION=168h
LOG_LEVEL=info
DB_SSL_MODE=require
```

#### Tenant Service
```bash
PORT=8082
DB_HOST=<production-db-host>
DB_PORT=5432
DB_USERNAME=<db-user>
DB_PASSWORD=<strong-password>
JWT_SIGNING_KEY=<same-as-gateway>
JWT_EXPIRATION=168h
LOG_LEVEL=info
DB_SSL_MODE=require
```

### Secrets Management

**Using Docker Secrets:**
```yaml
services:
  gateway:
    secrets:
      - jwt_signing_key
      - db_password

secrets:
  jwt_signing_key:
    external: true
  db_password:
    external: true
```

**Using Environment Files:**
```bash
# .env.production (never commit!)
JWT_SIGNING_KEY=<secret>
DB_PASSWORD=<secret>
```

---

## Database Setup

### Production Database

**Recommended:** Managed PostgreSQL service
- AWS RDS
- Google Cloud SQL
- Azure Database for PostgreSQL
- DigitalOcean Managed Databases

**Configuration:**
```sql
-- Enable SSL
ALTER SYSTEM SET ssl = on;

-- Set connection limits
ALTER SYSTEM SET max_connections = 200;

-- Enable slow query log
ALTER SYSTEM SET log_min_duration_statement = 1000;

-- Enable query statistics
CREATE EXTENSION pg_stat_statements;
```

### Backup Strategy

**Automated Backups:**
```bash
# Daily full backup
pg_dump -h <host> -U <user> -d shared > backup_$(date +%Y%m%d).sql

# Backup all tenant databases
for db in $(psql -h <host> -U <user> -t -c "SELECT datname FROM pg_database WHERE datname NOT IN ('postgres', 'template0', 'template1', 'shared')"); do
  pg_dump -h <host> -U <user> -d $db > backup_${db}_$(date +%Y%m%d).sql
done
```

**Retention Policy:**
- Daily backups: 7 days
- Weekly backups: 4 weeks
- Monthly backups: 12 months

**Backup Verification:**
```bash
# Test restore
pg_restore -h <test-host> -U <user> -d test_db backup.sql
```

---

## Docker Production Configuration

### Production Dockerfile

**Go Services:**
```dockerfile
# Build stage
FROM golang:1.24-alpine AS builder

RUN apk add --no-cache git ca-certificates

WORKDIR /build
COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -ldflags="-w -s" -o app ./cmd/*/

# Runtime stage
FROM scratch

COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /build/app /app
COPY --from=builder /build/migrations /migrations

EXPOSE 8080

USER 1000:1000

CMD ["/app"]
```

**Angular Frontend:**
```dockerfile
# Build stage
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build -- --configuration production

# Runtime stage
FROM nginx:alpine

COPY --from=builder /app/dist/ts-webapp-frontend /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### Production Docker Compose

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - gateway
      - webapp
      - website
    networks:
      - primeflow-network

  gateway:
    image: registry.example.com/primeflow-gateway:${VERSION}
    restart: always
    environment:
      - PORT=8080
      - JWT_SIGNING_KEY=${JWT_SIGNING_KEY}
      - USER_SERVICE_URL=http://users:8081
      - TENANT_SERVICE_URL=http://tenant:8082
      - LOG_LEVEL=info
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:8080/v1/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    networks:
      - primeflow-network

  users:
    image: registry.example.com/primeflow-users:${VERSION}
    restart: always
    environment:
      - PORT=8081
      - DB_HOST=${DB_HOST}
      - DB_PORT=${DB_PORT}
      - DB_USERNAME=${DB_USERNAME}
      - DB_PASSWORD=${DB_PASSWORD}
      - JWT_SIGNING_KEY=${JWT_SIGNING_KEY}
      - LOG_LEVEL=info
      - DB_SSL_MODE=require
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1'
          memory: 1G
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:8081/v1/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - primeflow-network

  tenant:
    image: registry.example.com/primeflow-tenant:${VERSION}
    restart: always
    environment:
      - PORT=8082
      - DB_HOST=${DB_HOST}
      - DB_PORT=${DB_PORT}
      - DB_USERNAME=${DB_USERNAME}
      - DB_PASSWORD=${DB_PASSWORD}
      - JWT_SIGNING_KEY=${JWT_SIGNING_KEY}
      - LOG_LEVEL=info
      - DB_SSL_MODE=require
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '1'
          memory: 1G
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:8082/v1/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - primeflow-network

  webapp:
    image: registry.example.com/primeflow-webapp:${VERSION}
    restart: always
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    networks:
      - primeflow-network

  website:
    image: registry.example.com/primeflow-website:${VERSION}
    restart: always
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    networks:
      - primeflow-network

networks:
  primeflow-network:
    driver: overlay
```

---

## SSL/TLS Configuration

### Nginx SSL Configuration

```nginx
server {
    listen 443 ssl http2;
    server_name app.primeflow.solutions;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    location / {
        proxy_pass http://webapp;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name app.primeflow.solutions;
    return 301 https://$server_name$request_uri;
}
```

### Let's Encrypt (Certbot)

```bash
# Install certbot
apt-get install certbot python3-certbot-nginx

# Generate certificate
certbot --nginx -d primeflow.solutions -d app.primeflow.solutions -d api.primeflow.solutions

# Auto-renewal
certbot renew --dry-run
```

---

## Monitoring & Logging

### Prometheus Metrics

**Add to services:**
```go
import "github.com/prometheus/client_golang/prometheus/promhttp"

http.Handle("/metrics", promhttp.Handler())
```

**Metrics to track:**
- Request count
- Request duration
- Error rate
- Database connections
- Memory usage
- CPU usage

### Logging

**Structured logging:**
```go
logger.Info("request processed",
    "method", r.Method,
    "path", r.URL.Path,
    "status", status,
    "duration", duration,
    "tenant_id", tenantID,
)
```

**Log aggregation:**
- ELK Stack (Elasticsearch, Logstash, Kibana)
- Loki + Grafana
- CloudWatch Logs
- Datadog

### Alerting

**Critical alerts:**
- Service down
- High error rate (>5%)
- Database connection failures
- High response time (>1s)
- Disk space low (<10%)
- Memory usage high (>90%)

---

## Load Balancing

### Nginx Load Balancer

```nginx
upstream gateway_backend {
    least_conn;
    server gateway1:8080 max_fails=3 fail_timeout=30s;
    server gateway2:8080 max_fails=3 fail_timeout=30s;
    server gateway3:8080 max_fails=3 fail_timeout=30s;
}

server {
    listen 443 ssl http2;
    server_name api.primeflow.solutions;

    location / {
        proxy_pass http://gateway_backend;
        proxy_next_upstream error timeout http_502 http_503 http_504;
    }
}
```

### Health Checks

```nginx
location /health {
    access_log off;
    proxy_pass http://gateway_backend/v1/health;
}
```

---

## Scaling Strategy

### Horizontal Scaling

**Scale services:**
```bash
docker-compose up --scale gateway=3 --scale users=3 --scale tenant=2
```

**Auto-scaling (Kubernetes):**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: gateway-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: gateway
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Database Scaling

**Read replicas:**
```go
// Use read replica for queries
readDB := dbManager.GetReadReplica()
users, err := userRepo.List(ctx, readDB)

// Use primary for writes
writeDB := dbManager.GetPrimaryDB()
err := userRepo.Create(ctx, writeDB, user)
```

**Connection pooling:**
```go
// Increase pool size for production
sqlDB.SetMaxIdleConns(20)
sqlDB.SetMaxOpenConns(200)
```

---

## Disaster Recovery

### Backup Procedures

**Automated backups:**
```bash
#!/bin/bash
# backup.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups"

# Backup shared database
pg_dump -h $DB_HOST -U $DB_USER -d shared | gzip > $BACKUP_DIR/shared_$DATE.sql.gz

# Backup all tenant databases
for db in $(psql -h $DB_HOST -U $DB_USER -t -c "SELECT database_name FROM tenant.tenants WHERE active = true"); do
  pg_dump -h $DB_HOST -U $DB_USER -d $db | gzip > $BACKUP_DIR/${db}_$DATE.sql.gz
done

# Upload to S3
aws s3 sync $BACKUP_DIR s3://primeflow-backups/

# Cleanup old backups (keep 7 days)
find $BACKUP_DIR -name "*.sql.gz" -mtime +7 -delete
```

### Recovery Procedures

**Restore database:**
```bash
# Restore shared database
gunzip < backup_shared.sql.gz | psql -h $DB_HOST -U $DB_USER -d shared

# Restore tenant database
gunzip < backup_tenant.sql.gz | psql -h $DB_HOST -U $DB_USER -d tenant_db
```

**Service recovery:**
```bash
# Rollback to previous version
docker-compose pull
docker-compose up -d --force-recreate

# Or specific version
docker-compose up -d gateway:v1.0.0
```

---

## Performance Optimization

### Database Optimization

**Indexes:**
```sql
-- Add indexes for frequent queries
CREATE INDEX CONCURRENTLY idx_users_email ON users.users(email);
CREATE INDEX CONCURRENTLY idx_users_active ON users.users(active);
CREATE INDEX CONCURRENTLY idx_tenants_slug ON tenant.tenants(slug);
CREATE INDEX CONCURRENTLY idx_email_domains_domain ON tenant.email_domains(domain);
```

**Query optimization:**
```sql
-- Analyze query performance
EXPLAIN ANALYZE SELECT * FROM users.users WHERE email = 'test@example.com';

-- Update statistics
ANALYZE users.users;
```

### Caching

**Redis caching:**
```go
// Cache tenant lookups
func (r *TenantRepository) FindBySlug(ctx context.Context, slug string) (*Tenant, error) {
    // Check cache
    cached, err := r.cache.Get(ctx, "tenant:"+slug)
    if err == nil {
        return cached, nil
    }
    
    // Query database
    tenant, err := r.db.FindBySlug(ctx, slug)
    if err != nil {
        return nil, err
    }
    
    // Cache result
    r.cache.Set(ctx, "tenant:"+slug, tenant, 1*time.Hour)
    
    return tenant, nil
}
```

### CDN Configuration

**Static assets:**
```nginx
location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}
```

---

## Security Best Practices

### Application Security

1. **Input validation**
2. **SQL injection prevention** (use parameterized queries)
3. **XSS prevention** (sanitize output)
4. **CSRF protection**
5. **Rate limiting**
6. **Password hashing** (bcrypt cost 12)
7. **JWT token expiration**
8. **HTTPS only**
9. **Security headers**
10. **Regular security audits**

### Infrastructure Security

1. **Firewall rules**
2. **VPC/Private networks**
3. **Secrets management**
4. **Regular updates**
5. **Access control**
6. **Audit logging**
7. **Intrusion detection**
8. **DDoS protection**
9. **Backup encryption**
10. **Compliance (GDPR, SOC2)**

---

## Deployment Checklist

### Pre-Deployment
- [ ] All tests passing
- [ ] Security audit completed
- [ ] Performance testing done
- [ ] Backup strategy in place
- [ ] Monitoring configured
- [ ] SSL certificates ready
- [ ] DNS configured
- [ ] Secrets configured
- [ ] Documentation updated

### Deployment
- [ ] Build production images
- [ ] Tag images with version
- [ ] Push to registry
- [ ] Update environment variables
- [ ] Run database migrations
- [ ] Deploy services
- [ ] Verify health checks
- [ ] Test critical paths
- [ ] Monitor logs
- [ ] Monitor metrics

### Post-Deployment
- [ ] Verify all services running
- [ ] Test user flows
- [ ] Check error rates
- [ ] Monitor performance
- [ ] Review logs
- [ ] Update documentation
- [ ] Notify team
- [ ] Schedule post-mortem

---

## Rollback Plan

### Quick Rollback

```bash
# Rollback to previous version
docker-compose pull primeflow-gateway:v1.0.0
docker-compose up -d gateway

# Or rollback all services
docker-compose -f docker-compose.prod.yml down
docker-compose -f docker-compose.prod.yml up -d --force-recreate
```

### Database Rollback

```bash
# Restore from backup
pg_restore -h $DB_HOST -U $DB_USER -d shared backup.sql

# Or rollback migrations
# (requires down migrations)
```

---

## Maintenance

### Regular Tasks

**Daily:**
- Check error logs
- Monitor metrics
- Verify backups

**Weekly:**
- Review performance
- Check disk space
- Update dependencies
- Security patches

**Monthly:**
- Database maintenance (VACUUM, REINDEX)
- Review access logs
- Capacity planning
- Security audit

### Maintenance Window

```bash
# Put site in maintenance mode
docker-compose stop gateway users tenant

# Perform maintenance
# - Database updates
# - Schema changes
# - Data migrations

# Bring site back online
docker-compose up -d
```

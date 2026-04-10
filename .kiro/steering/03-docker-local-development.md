---
inclusion: auto
---

# Docker - Desenvolvimento Local

## Estrutura do Projeto

```
.
├── docker-compose.yml          # Orquestração de todos os serviços
├── nginx.conf                  # Configuração do reverse proxy
├── .dockerignore              # Arquivos a ignorar no build
├── go-gateway-service/        # API Gateway
│   ├── Dockerfile.dev
│   └── services.dev.yaml
├── go-tenant-service/         # Serviço de Tenants
│   └── Dockerfile.dev
├── go-users-service/          # Serviço de Usuários
│   └── Dockerfile.dev
├── go-organizations-service/  # Serviço de Organizações
│   └── Dockerfile.dev
├── go-pkg-lib/               # Biblioteca compartilhada
├── ts-webapp-frontend/       # Frontend Webapp (Angular)
│   └── Dockerfile.dev
└── ts-website-frontend/      # Frontend Website (Angular)
    └── Dockerfile.dev
```

## Comandos Essenciais

### Iniciar todos os serviços
```bash
docker-compose up -d
```

### Ver logs de todos os serviços
```bash
docker-compose logs -f
```

### Ver logs de um serviço específico
```bash
docker-compose logs -f gateway
docker-compose logs -f organizations
docker-compose logs -f webapp
```

### Parar todos os serviços
```bash
docker-compose down
```

### Rebuild de um serviço específico
```bash
docker-compose up -d --build gateway
docker-compose up -d --build organizations
```

### Rebuild de todos os serviços
```bash
docker-compose up -d --build
```

### Remover volumes (limpar dados)
```bash
docker-compose down -v
```

## URLs de Acesso

- **Website**: http://localhost
- **Webapp**: http://app.localhost
- **API Gateway**: http://api.localhost

## Arquitetura

### Nginx (Reverse Proxy)
- Porta: 80
- Roteia requisições para os serviços corretos baseado no hostname

### Backend Services (Go)
- **Gateway**: Porta 8080 (interno)
- **Users**: Porta 8081 (interno)
- **Tenant**: Porta 8082 (interno)
- **Organizations**: Porta 8083 (interno)

### Frontend Services (Angular)
- **Webapp**: Porta 4200 (interno)
- **Website**: Porta 4200 (interno)

## Rede Docker

Todos os serviços estão na mesma rede `primeflow-network`, permitindo comunicação interna entre eles.

## Hot Reload

### Backend (Go)
- Usa **Air** para hot reload
- Configurado em `.air.toml`
- Recompila automaticamente ao salvar arquivos

### Frontend (Angular)
- Usa servidor de desenvolvimento do Angular
- Recarrega automaticamente ao salvar arquivos
- WebSocket configurado no nginx para live reload

## Variáveis de Ambiente

Definidas no `docker-compose.yml`:

```yaml
environment:
  - PORT=8083
  - DB_HOST=trolley.proxy.rlwy.net
  - DB_PORT=11306
  - DB_USERNAME=postgres
  - DB_PASSWORD=clBcJPLpujzLWqRfXejgqvClSdJWefYn
  - JWT_SIGNING_KEY=your-secret-key
```

## Troubleshooting

### Serviço não inicia
```bash
# Ver logs detalhados
docker-compose logs -f <service-name>

# Rebuild forçado
docker-compose up -d --build --force-recreate <service-name>
```

### Porta já em uso
```bash
# Verificar o que está usando a porta 80
lsof -i :80

# Parar o processo ou mudar a porta no docker-compose.yml
```

### Mudanças não refletem
```bash
# Rebuild do serviço
docker-compose up -d --build <service-name>

# Limpar cache do Docker
docker system prune -a
```

### Problemas de rede
```bash
# Recriar a rede
docker-compose down
docker network prune
docker-compose up -d
```

## Boas Práticas

1. **Sempre use `-d` (detached)** para rodar em background
2. **Use logs com `-f`** para acompanhar em tempo real
3. **Rebuild após mudanças em dependências** (go.mod, package.json)
4. **Não commite senhas** - use .env para desenvolvimento local
5. **Mantenha volumes limpos** - remova volumes não utilizados periodicamente

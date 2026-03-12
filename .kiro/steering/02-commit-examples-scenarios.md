---
inclusion: auto
---

# Cenários Práticos de Commits - Semantic Release

## Cenários do Dia a Dia

### Cenário 1: Nova Feature Completa

**Situação**: Implementar sistema de autenticação completo

```bash
# Commit 1: Estrutura base
feat(auth): adiciona estrutura base de autenticação

Cria models, services e controllers para autenticação.
Prepara infraestrutura para JWT e refresh tokens.

# Commit 2: Implementação do login
feat(auth): implementa endpoint de login

POST /api/auth/login
- Valida credenciais
- Gera JWT token
- Retorna user data

# Commit 3: Implementação do registro
feat(auth): implementa endpoint de registro

POST /api/auth/register
- Valida dados do usuário
- Hash de senha com bcrypt
- Cria usuário no banco

# Commit 4: Refresh token
feat(auth): adiciona refresh token

POST /api/auth/refresh
- Valida refresh token
- Gera novo access token
- Rotaciona refresh token

# Commit 5: Testes
test(auth): adiciona testes de autenticação

Cobertura de 95% nos endpoints de auth

# Commit 6: Documentação
docs(auth): documenta API de autenticação

Adiciona exemplos de uso e códigos de erro
```

### Cenário 2: Correção de Bug Crítico

**Situação**: Usuários não conseguem fazer login

```bash
# Investigação e fix
fix(auth): corrige validação de senha

A comparação de hash estava usando algoritmo errado,
causando falha em 100% dos logins. Corrige para usar
bcrypt.compare() corretamente.

Fixes #789
Priority: Critical
```

### Cenário 3: Refatoração Grande

**Situação**: Melhorar arquitetura do código

```bash
# Commit 1: Extração de serviços
refactor(users): extrai lógica para UserService

Move lógica de negócio dos controllers para service layer.
Melhora testabilidade e separação de responsabilidades.

# Commit 2: Padronização de erros
refactor(errors): padroniza tratamento de erros

Cria ErrorHandler centralizado e custom exceptions.
Remove try-catch duplicados.

# Commit 3: Otimização
perf(database): otimiza queries de usuários

Adiciona índices e eager loading. Reduz queries N+1.
Tempo de resposta: 800ms → 120ms

# Commit 4: Testes
test(users): atualiza testes após refatoração

Ajusta mocks e assertions para nova arquitetura
```

### Cenário 4: Breaking Change Planejado

**Situação**: Migrar de REST para GraphQL

```bash
# Commit 1: Preparação
feat(graphql): adiciona suporte a GraphQL

Mantém compatibilidade com REST existente.
GraphQL disponível em /graphql

# Commit 2: Deprecação
feat(api): marca REST API como deprecated

Adiciona headers de deprecação:
- Deprecation: true
- Sunset: 2024-12-31

docs(api): documenta migração para GraphQL

# Commit 3: Breaking change
feat(api)!: remove REST API v1

BREAKING CHANGE: REST API v1 foi removida.
Use GraphQL API em /graphql

Migration guide: docs/migration-rest-to-graphql.md

Closes #456
```

### Cenário 5: Hotfix em Produção

**Situação**: Bug crítico em produção

```bash
# Branch: hotfix/critical-memory-leak
fix(websocket): corrige memory leak crítico

Conexões websocket não estavam sendo limpas após
disconnect, causando acúmulo de memória. Adiciona
cleanup adequado no evento 'close'.

Impact: Reduz uso de memória de 2GB/hora para estável
Fixes #999
Priority: Critical
```

### Cenário 6: Atualização de Dependências

**Situação**: Atualizar dependências com breaking changes

```bash
# Sem breaking change para usuários
build(deps): atualiza dependências de desenvolvimento

- jest: 28.0.0 → 29.0.0
- eslint: 8.0.0 → 8.50.0
- prettier: 2.8.0 → 3.0.0

# Com breaking change para usuários
build(deps)!: atualiza Node.js para v20

BREAKING CHANGE: Node.js 20+ agora é obrigatório.
Versões anteriores não são mais suportadas.

Motivo: Uso de novas APIs nativas (fetch, test runner)
```

### Cenário 7: Feature com Múltiplos Componentes

**Situação**: Sistema de notificações completo

```bash
# Commit 1: Backend
feat(notifications): adiciona serviço de notificações

- WebSocket para real-time
- Persistência no banco
- Queue para processamento

# Commit 2: Frontend
feat(ui): adiciona componente de notificações

- Badge com contador
- Dropdown com lista
- Marca como lida

# Commit 3: Integração
feat(notifications): integra com eventos do sistema

Envia notificações para:
- Novos comentários
- Menções
- Atualizações de status

# Commit 4: Configuração
feat(notifications): adiciona preferências de notificação

Usuários podem configurar:
- Tipos de notificação
- Canais (email, push, in-app)
- Frequência

# Commit 5: Testes
test(notifications): adiciona testes end-to-end

Testa fluxo completo de notificações
```

### Cenário 8: Correções Múltiplas

**Situação**: Várias correções pequenas

```bash
# Commits separados (CORRETO)
fix(auth): corrige validação de email
fix(users): corrige formatação de telefone
fix(api): corrige timeout em requests longos

# ❌ NÃO FAÇA (commit único para tudo)
fix: corrige vários bugs
```

### Cenário 9: Feature Flag

**Situação**: Feature gradual com flag

```bash
# Commit 1: Feature desabilitada
feat(dashboard): adiciona novo dashboard

Feature flag: NEW_DASHBOARD=false
Disponível apenas para testes internos

# Commit 2: Ajustes baseados em feedback
fix(dashboard): ajusta layout responsivo

# Commit 3: Habilita para todos
feat(dashboard): habilita novo dashboard para todos

Feature flag: NEW_DASHBOARD=true
Remove dashboard antigo
```

### Cenário 10: Segurança

**Situação**: Correção de vulnerabilidade

```bash
# Vulnerabilidade crítica
fix(security): corrige SQL injection em busca

Sanitiza input do usuário antes de usar em queries.
Adiciona prepared statements.

CVE: CVE-2024-12345
Severity: Critical
Fixes #1001

# Atualização de segurança
build(deps): atualiza express para versão segura

express: 4.17.1 → 4.18.2
Corrige CVE-2022-24999

# Breaking change de segurança
feat(auth)!: adiciona rate limiting obrigatório

BREAKING CHANGE: Todas as rotas agora têm rate limiting.
Clientes devem implementar retry com backoff.

Previne ataques de força bruta e DDoS.
```

## Padrões de Mensagens por Contexto

### API/Backend

```bash
feat(api): adiciona endpoint GET /api/users/:id
fix(api): corrige validação de request body
perf(api): otimiza serialização de resposta
refactor(api): padroniza formato de erro
```

### Frontend/UI

```bash
feat(ui): adiciona modal de confirmação
fix(ui): corrige alinhamento no mobile
style(ui): atualiza cores do tema
refactor(ui): migra para composition API
```

### Database

```bash
feat(db): adiciona migration para tabela orders
fix(db): corrige índice duplicado
perf(db): otimiza query de relatórios
refactor(db): normaliza estrutura de endereços
```

### DevOps/Infra

```bash
feat(docker): adiciona multi-stage build
fix(k8s): corrige health check
ci(github): adiciona workflow de deploy
build(webpack): otimiza bundle size
```

### Documentação

```bash
docs: atualiza README com instruções de setup
docs(api): adiciona exemplos de autenticação
docs(contributing): cria guia de contribuição
```

## Mensagens de Commit Ruins vs Boas

### ❌ Ruins

```bash
fix: bug
feat: new feature
update code
WIP
fix stuff
changes
asdfasdf
```

### ✅ Boas

```bash
fix(auth): corrige timeout no refresh token
feat(users): adiciona filtro de busca avançada
refactor(api): simplifica middleware de autenticação
perf(database): adiciona índice em users.email
docs(readme): atualiza instruções de instalação
```

## Template de Commit Complexo

```bash
<tipo>(<escopo>): <descrição curta>

<corpo detalhado explicando o QUE e POR QUÊ>
<não o COMO - isso está no código>

<impacto da mudança>
<considerações importantes>

<rodapés>
BREAKING CHANGE: <descrição da breaking change>
Fixes #<issue>
Closes #<issue>
Refs #<issue>
Co-authored-by: <nome> <email>
```

### Exemplo Real

```bash
feat(payment)!: integra gateway de pagamento Stripe

Adiciona integração completa com Stripe para processar
pagamentos com cartão de crédito. Suporta:
- Pagamentos únicos
- Assinaturas recorrentes
- Webhooks para confirmação

Substitui integração anterior com PagSeguro que será
descontinuada em 2024.

Performance: Tempo de processamento reduzido de 3s para 800ms
Security: Tokens de cartão nunca tocam nosso servidor

BREAKING CHANGE: Remove suporte a PagSeguro.
Clientes devem migrar para Stripe.
Migration guide: docs/payment-migration.md

Closes #234, #567
Refs #890
Co-authored-by: João Silva <joao@example.com>
```

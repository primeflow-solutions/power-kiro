---
inclusion: auto
---

# Padrões de Commit - Semantic Release

## REGRAS OBRIGATÓRIAS

Todos os commits DEVEM seguir rigorosamente o padrão Conventional Commits para garantir versionamento automático e geração de changelog.

## Formato do Commit

```
<tipo>[escopo opcional]: <descrição>

[corpo opcional]

[rodapé(s) opcional(is)]
```

## Tipos de Commit (OBRIGATÓRIO)

### Tipos que GERAM release:

- **feat**: Nova funcionalidade (MINOR version bump)
- **fix**: Correção de bug (PATCH version bump)
- **perf**: Melhoria de performance que corrige um problema (PATCH version bump)

### Tipos que NÃO geram release:

- **build**: Mudanças no sistema de build ou dependências externas
- **ci**: Mudanças em arquivos e scripts de CI/CD
- **docs**: Apenas mudanças em documentação
- **style**: Mudanças que não afetam o significado do código (espaços, formatação, etc)
- **refactor**: Mudança de código que não corrige bug nem adiciona funcionalidade
- **test**: Adição ou correção de testes
- **chore**: Outras mudanças que não modificam src ou arquivos de teste

### Breaking Changes (MAJOR version bump):

Adicione `!` após o tipo/escopo OU inclua `BREAKING CHANGE:` no rodapé:

```
feat!: remove suporte para Node 12

BREAKING CHANGE: Node 12 não é mais suportado
```

## Regras de Escrita

1. **Descrição**: DEVE estar em minúsculas, sem ponto final
2. **Tamanho**: Máximo 100 caracteres na primeira linha
3. **Imperativo**: Use modo imperativo ("adiciona" não "adicionado" ou "adicionando")
4. **Idioma**: Português ou inglês, mas seja consistente no projeto
5. **Escopo**: Use quando aplicável (ex: `feat(auth):`, `fix(api):`)

## Exemplos Corretos

### Features (MINOR bump)

```bash
# Feature simples
feat: adiciona autenticação JWT

# Feature com escopo
feat(auth): implementa login com OAuth2

# Feature com corpo explicativo
feat(api): adiciona endpoint de busca de usuários

Implementa busca paginada com filtros por nome,
email e status. Suporta ordenação por múltiplos campos.

# Feature com breaking change
feat(database)!: migra para PostgreSQL 15

BREAKING CHANGE: Requer PostgreSQL 15+. Versões anteriores não são mais suportadas.
```

### Fixes (PATCH bump)

```bash
# Fix simples
fix: corrige validação de email

# Fix com escopo e issue
fix(auth): resolve timeout no refresh token

Closes #123

# Fix com detalhes
fix(api): corrige race condition no cache

O cache estava sendo acessado concorrentemente sem lock,
causando inconsistências. Adiciona mutex para sincronização.

Fixes #456
```

### Performance (PATCH bump)

```bash
# Performance que corrige problema
perf(database): otimiza query de listagem

Reduz tempo de resposta de 2s para 200ms usando índices compostos.

# Performance crítica
perf(api): resolve memory leak no websocket

BREAKING CHANGE: Altera assinatura do método connect()
```

### Breaking Changes (MAJOR bump)

```bash
# Com ! no tipo
feat!: remove API v1

BREAKING CHANGE: API v1 foi removida. Migre para v2.

# Com ! no escopo
refactor(api)!: altera estrutura de resposta

BREAKING CHANGE: Respostas agora seguem formato JSON:API

# Múltiplas breaking changes
feat!: atualiza dependências principais

BREAKING CHANGE: Node.js 14+ agora é obrigatório
BREAKING CHANGE: TypeScript 5+ agora é obrigatório
```

### Outros Tipos (SEM release)

```bash
# Build
build: atualiza dependências de desenvolvimento
build(deps): bump typescript de 4.9 para 5.0

# CI/CD
ci: adiciona workflow de deploy automático
ci(github): configura cache de dependências

# Documentação
docs: atualiza README com exemplos de uso
docs(api): adiciona documentação do endpoint /users

# Style
style: formata código com prettier
style(components): ajusta indentação

# Refactor
refactor: simplifica lógica de validação
refactor(auth): extrai validação para service separado

# Tests
test: adiciona testes para UserService
test(integration): adiciona testes de API

# Chore
chore: atualiza .gitignore
chore(release): 1.2.3
```

## Exemplos com Múltiplos Commits

### Release com Features e Fixes

```bash
# Commit 1 (MINOR)
feat(users): adiciona CRUD de usuários

# Commit 2 (PATCH)
fix(users): corrige validação de CPF

# Commit 3 (sem release)
docs(users): adiciona documentação da API

# Commit 4 (MINOR)
feat(auth): implementa 2FA

# Resultado: versão 1.1.0 → 1.2.0 (MINOR prevalece)
```

### Release com Breaking Change

```bash
# Commit 1 (MINOR)
feat(api): adiciona paginação

# Commit 2 (MAJOR)
feat(database)!: migra para MongoDB

BREAKING CHANGE: PostgreSQL não é mais suportado

# Commit 3 (PATCH)
fix(api): corrige ordenação

# Resultado: versão 1.2.0 → 2.0.0 (MAJOR prevalece)
```

## Rodapés Especiais

### Referências a Issues

```bash
# Fecha uma issue
fix: corrige bug de login

Closes #123

# Fecha múltiplas issues
feat: implementa dashboard

Closes #45, #67, #89

# Referencia sem fechar
refactor: melhora performance

Refs #234
```

### Co-autores

```bash
feat: adiciona sistema de notificações

Co-authored-by: João Silva <joao@example.com>
Co-authored-by: Maria Santos <maria@example.com>
```

### Reviewed-by

```bash
fix: corrige vulnerabilidade de segurança

Reviewed-by: Security Team <security@example.com>
```

## Validação Automática

Configure commitlint para validar commits:

```json
{
  "extends": ["@commitlint/config-conventional"],
  "rules": {
    "type-enum": [
      2,
      "always",
      [
        "feat",
        "fix",
        "docs",
        "style",
        "refactor",
        "perf",
        "test",
        "build",
        "ci",
        "chore"
      ]
    ],
    "subject-case": [2, "always", "lower-case"],
    "subject-max-length": [2, "always", 100]
  }
}
```

## Erros Comuns a EVITAR

❌ **ERRADO:**
```bash
# Sem tipo
adiciona login

# Tipo inválido
feature: adiciona login

# Primeira letra maiúscula
feat: Adiciona login

# Ponto final
feat: adiciona login.

# Muito vago
fix: corrige bug

# Tempo verbal errado
feat: adicionando login
feat: adicionado login
```

✅ **CORRETO:**
```bash
feat: adiciona login
feat(auth): implementa autenticação JWT
fix(api): corrige timeout no endpoint de usuários
```

## Checklist Antes de Commitar

- [ ] Tipo está correto e em minúsculas?
- [ ] Descrição está em minúsculas e sem ponto final?
- [ ] Descrição usa modo imperativo?
- [ ] Descrição tem menos de 100 caracteres?
- [ ] Escopo está correto (se aplicável)?
- [ ] Breaking change está marcado com `!` ou `BREAKING CHANGE:`?
- [ ] Issues estão referenciadas corretamente?
- [ ] Commit contém apenas mudanças relacionadas?

## Ferramentas Recomendadas

- **commitizen**: CLI interativo para criar commits
- **commitlint**: Valida mensagens de commit
- **husky**: Git hooks para validação automática
- **semantic-release**: Automação de versionamento e release

## Configuração Recomendada

```bash
# Instalar ferramentas
npm install -D @commitlint/cli @commitlint/config-conventional husky

# Configurar husky
npx husky init
echo "npx --no -- commitlint --edit \$1" > .husky/commit-msg
```

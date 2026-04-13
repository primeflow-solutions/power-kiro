# Steering Files - Power Kiro

## 📋 Índice de Documentação

Este diretório contém todos os guias e padrões que a Kiro AI deve seguir ao desenvolver projetos.

### 🔄 Commits e Versionamento

#### **01-commit-standards.md**
Regras fundamentais do Conventional Commits para versionamento semântico automático.

**Quando usar**: Sempre que for fazer commits.

**Principais tópicos**:
- Tipos de commit (feat, fix, docs, etc)
- Estrutura de mensagens
- Breaking changes
- Versionamento automático

#### **02-commit-examples-scenarios.md**
Cenários práticos e exemplos reais de commits do dia a dia.

**Quando usar**: Para referência de como escrever commits em situações específicas.

**Principais tópicos**:
- 10+ cenários práticos
- Exemplos de commits bons e ruins
- Casos especiais

### 🐳 Docker e Desenvolvimento Local

#### **03-docker-local-development.md**
Comandos essenciais e arquitetura de desenvolvimento local com Docker.

**Quando usar**: Ao trabalhar com Docker no desenvolvimento local.

**Principais tópicos**:
- Comandos Docker essenciais
- Arquitetura com docker-compose
- Hot reload (Air para Go, Angular)
- Troubleshooting

#### **04-docker-compose-from-scratch.md**
Como criar uma estrutura docker-compose completa do zero.

**Quando usar**: Ao iniciar um novo projeto com Docker.

**Principais tópicos**:
- Estrutura de pastas
- docker-compose.yml completo
- Nginx como reverse proxy
- Configuração de serviços

#### **05-docker-add-new-service.md**
Templates para adicionar novos serviços ao docker-compose.

**Quando usar**: Ao adicionar um novo serviço (Go, Angular, etc) ao projeto.

**Principais tópicos**:
- Template para serviço Go
- Template para serviço Angular
- Template para banco de dados
- Configuração de rede

### 🌿 Git Workflow

#### **06-git-workflow.md**
Fluxo de trabalho com Git e estratégia de branches.

**Quando usar**: Ao trabalhar com Git e branches.

**Principais tópicos**:
- Estratégia de branches
- Pull requests
- Code review
- Merge vs Rebase

### 🏗️ Go - Arquitetura Modular

#### **07-go-modular-architecture.md** ⭐ NOVO
Padrão completo de arquitetura modular baseada em domínios para projetos Go.

**Quando usar**: 
- Ao criar QUALQUER novo projeto Go
- Ao refatorar projetos Go existentes
- Como referência para estrutura de código

**Principais tópicos**:
- Princípios fundamentais (organização por módulo, não por camada)
- Estrutura de 8 arquivos por módulo
- Templates completos de código
- Regras e anti-padrões
- Benefícios da arquitetura modular

**Estrutura de um módulo**:
```
module/[nome]/
├── entity.go           # Entidade do domínio
├── repository.go       # Interface do repositório
├── repository_impl.go  # Implementação Postgres
├── dto.go              # DTOs (Request/Response)
├── mapper.go           # Conversões DTO ↔ Entity
├── usecase.go          # Todos os use cases
├── handler.go          # HTTP Handler
└── routes.go           # Registro de rotas
```

#### **08-go-modular-examples.md** ⭐ NOVO
Exemplos práticos e casos de uso reais da arquitetura modular.

**Quando usar**:
- Como referência ao criar novos módulos
- Para entender casos específicos (relacionamentos, validações, etc)
- Para ver exemplos completos de código

**Principais tópicos**:
- Exemplo completo: Módulo Product
- Módulo com relacionamentos (Order → Customer)
- Paginação e filtros
- Validações customizadas (CNPJ)
- Testes unitários
- Soft delete

#### **09-go-modular-migration.md** ⭐ NOVO
Guia passo a passo para migrar projetos Go existentes para arquitetura modular.

**Quando usar**:
- Ao migrar projeto organizado por camadas para modular
- Como checklist de migração
- Para estimar tempo de migração

**Principais tópicos**:
- Processo de migração em 6 fases
- Migração módulo por módulo
- Checklist completo
- Problemas comuns e soluções
- Estimativa de tempo

## 🎯 Fluxo de Uso

### **Para Novos Projetos Go**

1. Leia `07-go-modular-architecture.md` para entender o padrão
2. Use `08-go-modular-examples.md` como referência
3. Crie módulos seguindo a estrutura de 8 arquivos
4. Faça commits seguindo `01-commit-standards.md`

### **Para Migração de Projetos Go**

1. Leia `09-go-modular-migration.md` para o processo completo
2. Use `07-go-modular-architecture.md` como referência
3. Use `08-go-modular-examples.md` para casos específicos
4. Siga o checklist de migração

### **Para Desenvolvimento com Docker**

1. Use `03-docker-local-development.md` para comandos do dia a dia
2. Use `04-docker-compose-from-scratch.md` para novos projetos
3. Use `05-docker-add-new-service.md` para adicionar serviços

### **Para Commits**

1. Sempre consulte `01-commit-standards.md`
2. Use `02-commit-examples-scenarios.md` para casos específicos

## 📊 Estatísticas

| Categoria | Arquivos | Linhas | Tópicos |
|-----------|----------|--------|---------|
| Commits | 2 | ~400 | 15+ |
| Docker | 3 | ~500 | 20+ |
| Git | 1 | ~100 | 5+ |
| Go Modular | 3 | ~1500 | 50+ |
| **Total** | **9** | **~2500** | **90+** |

## ✅ Checklist de Qualidade

Ao desenvolver com Kiro AI, certifique-se de:

### **Commits**
- [ ] Seguir Conventional Commits
- [ ] Mensagens claras e descritivas
- [ ] Breaking changes documentados

### **Docker**
- [ ] docker-compose.yml organizado
- [ ] Hot reload configurado
- [ ] Variáveis de ambiente documentadas

### **Go - Arquitetura Modular**
- [ ] Estrutura de 8 arquivos por módulo
- [ ] Logs estruturados em todos os use cases
- [ ] Timeout de 5s em todos os handlers
- [ ] Validação estruturada nos DTOs
- [ ] Error wrapping em todos os erros
- [ ] Dependency Injection via factory
- [ ] Timestamps automáticos nas entidades

## 🚀 Próximos Passos

Novos guias planejados:

- [ ] API Design e REST Best Practices
- [ ] Testing Strategies (Unit, Integration, E2E)
- [ ] CI/CD Pipelines
- [ ] Monitoring e Observability
- [ ] Security Best Practices
- [ ] Performance Optimization

## 📚 Referências Externas

- [Conventional Commits](https://www.conventionalcommits.org/)
- [Semantic Release](https://semantic-release.gitbook.io/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Domain-Driven Design](https://www.domainlanguage.com/ddd/)
- [Go Project Layout](https://github.com/golang-standards/project-layout)

---

**Mantido por**: PrimeFlow Solutions  
**Última atualização**: 2026-04-13  
**Versão**: 1.x.x

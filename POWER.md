---
name: "power-kiro"
displayName: "Power Kiro - Development Standards"
description: "Padrões rigorosos e instruções completas para desenvolvimento com Kiro AI - Inclui Conventional Commits, Semantic Release, Docker, e guias de desenvolvimento local"
keywords: ["standards", "commits", "docker", "development", "best-practices", "semantic-release", "conventional-commits", "git", "workflow"]
---

# Power Kiro

Padrões rigorosos e instruções completas para desenvolvimento de projetos com Kiro AI.

## 📋 Sobre

Power Kiro é um conjunto abrangente de regras, padrões e guias que garantem qualidade, consistência e profissionalismo em todos os projetos desenvolvidos com assistência da Kiro AI.

## 🎯 O que está incluído

### Padrões de Commit (Semantic Release)
- Regras obrigatórias de Conventional Commits
- Tipos de commit e versionamento automático
- Breaking changes e changelog
- 10+ cenários práticos do dia a dia
- Validação automática com commitlint e husky

### Docker e Desenvolvimento Local
- Estrutura completa de docker-compose
- Hot reload para Go (Air) e Angular
- Nginx como reverse proxy
- Templates para adicionar novos serviços
- Troubleshooting e boas práticas

### API Design (Em breve)
- Padrões de endpoints REST
- Estrutura de request/response
- Tratamento de erros
- Versionamento de APIs

## 🚀 Como usar

### Instalação Automática

Os arquivos de steering em `.kiro/steering/` são automaticamente carregados pela Kiro AI quando você trabalha em projetos que incluem este power.

### Instalação Manual

1. Clone o repositório:
```bash
git clone https://github.com/primeflow-solutions/power-kiro.git .kiro-powers/power-kiro
```

2. Copie os arquivos de steering:
```bash
cp -r .kiro-powers/power-kiro/.kiro/steering/* .kiro/steering/
```

## 📚 Conteúdo Detalhado

### Commits
- **01-commit-standards.md**: Regras fundamentais do Conventional Commits
- **02-commit-examples-scenarios.md**: Cenários práticos e exemplos reais

### Docker
- **03-docker-local-development.md**: Comandos essenciais e arquitetura
- **04-docker-compose-from-scratch.md**: Como criar do zero
- **05-docker-add-new-service.md**: Templates para novos serviços

## 🔄 Versionamento

Este projeto usa Semantic Release para versionamento automático:

- `feat:` → versão MINOR (1.0.0 → 1.1.0)
- `fix:` → versão PATCH (1.0.0 → 1.0.1)
- `BREAKING CHANGE:` → versão MAJOR (1.0.0 → 2.0.0)

Cada commit na branch `main` gera automaticamente uma nova versão com changelog.

## 🤝 Contribuindo

1. Fork o projeto
2. Crie uma branch para sua feature (`git checkout -b feat/nova-feature`)
3. Commit suas mudanças seguindo Conventional Commits
4. Push para a branch (`git push origin feat/nova-feature`)
5. Abra um Pull Request

## 📝 Licença

MIT License - veja [LICENSE](LICENSE) para detalhes.

## 🏢 Mantido por

PrimeFlow Solutions

---

**Nota**: Este power está em constante evolução. Novos padrões e guias são adicionados regularmente.

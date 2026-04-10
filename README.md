# Power Kiro

Padrões e instruções rigorosas para implementações de projetos usando Kiro AI.

## 📋 O que é?

Power Kiro é um conjunto de regras e padrões que garantem qualidade e consistência em projetos desenvolvidos com assistência da Kiro AI.

## 📚 Conteúdo

### Padrões de Commit

Regras rigorosas seguindo Conventional Commits e Semantic Release:

- **01-commit-standards.md**: Regras fundamentais do Conventional Commits
- **02-commit-examples-scenarios.md**: Cenários práticos e exemplos reais

Inclui validação automática com commitlint e husky, garantindo que todos os commits sigam o padrão.

### Docker e Desenvolvimento Local

Instruções completas para rodar projetos localmente:

- **03-docker-local-development.md**: Comandos essenciais, URLs, arquitetura
- **04-docker-compose-from-scratch.md**: Como criar docker-compose do zero
- **05-docker-add-new-service.md**: Templates para adicionar novos serviços

Suporte para hot reload (Air para Go, Angular dev server), nginx como reverse proxy, e troubleshooting.

## 🚀 Como usar

### No Kiro AI

Os arquivos de steering em `.kiro/steering/` são automaticamente carregados pela Kiro AI quando você trabalha em projetos que incluem este power.

### Instalação em Projeto

1. Clone ou adicione como submódulo:

```bash
# Como submódulo
git submodule add https://github.com/primeflow-solutions/power-kiro.git .kiro-powers/power-kiro

# Ou clone direto
git clone https://github.com/primeflow-solutions/power-kiro.git .kiro-powers/power-kiro
```

2. Copie os arquivos de steering para seu projeto:

```bash
cp -r .kiro-powers/power-kiro/.kiro/steering/* .kiro/steering/
```

## 🔄 Versionamento

Este projeto usa [Semantic Release](https://semantic-release.gitbook.io/) para versionamento automático:

- Commits `feat:` geram versão MINOR
- Commits `fix:` geram versão PATCH
- Commits com `BREAKING CHANGE:` geram versão MAJOR

## 📝 Licença

MIT

## 🏢 Mantido por

PrimeFlow Solutions


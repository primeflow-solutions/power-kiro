---
inclusion: auto
---

# Git Workflow - Regras de Commit e Push

## REGRA CRÍTICA: Commits Apenas Quando Solicitado

**NUNCA faça commit ou push automaticamente após implementações.**

### Quando Fazer Commit

✅ **APENAS quando o usuário solicitar explicitamente:**
- "pode commitar"
- "faz o commit"
- "commita isso"
- "pode fazer commit"
- "commit e push"

❌ **NUNCA faça automaticamente:**
- Após criar arquivos
- Após corrigir erros
- Após implementar features
- Após fazer qualquer mudança

### Fluxo Correto

1. **Implementar** - Criar/modificar arquivos conforme solicitado
2. **Informar** - Avisar o usuário que a implementação está completa
3. **Aguardar** - Esperar o usuário revisar e solicitar commit
4. **Commitar** - Apenas quando explicitamente solicitado

### Exemplo de Fluxo

```
Usuário: "cria uma tela de login"
Kiro: [cria os arquivos]
Kiro: "Pronto! Criei a tela de login com..."

Usuário: "pode commitar"
Kiro: [faz git add, commit e push]
```

### Exceções

Não há exceções. Sempre aguarde a solicitação explícita do usuário.

### Após Primeiro Commit

Mesmo após fazer um commit solicitado, **NÃO faça commits subsequentes automaticamente**.

Cada commit precisa de autorização explícita do usuário.

## Comandos Git

### Verificar Status
```bash
git status
git diff
```

### Preparar Commit (apenas quando solicitado)
```bash
git add -A
git commit -m "mensagem seguindo conventional commits"
git push origin main
```

### Verificar Histórico
```bash
git log --oneline
```

## Mensagens de Commit

Sempre seguir o padrão Conventional Commits conforme definido em `01-commit-standards.md`.

## Resumo

- ✅ Implementar quando solicitado
- ✅ Informar conclusão
- ⏸️ Aguardar autorização
- ✅ Commitar apenas quando autorizado
- 🔁 Repetir o ciclo para cada mudança

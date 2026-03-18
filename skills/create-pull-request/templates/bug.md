## 🔎 Problema

Descrição clara do erro.

- O que estava acontecendo?
- Em qual ambiente?
- Desde quando?
- Impacto para o usuário/negócio?

Exemplo:
> Endpoint /payments retornava 500 quando payload X era enviado.

---

## 🎯 Causa Raiz (Root Cause)

Explique tecnicamente o motivo do problema.

- Falha de validação?
- Condição de corrida?
- Migration incompatível?
- Deploy parcial?
- Regressão de outra PR?

⚠ Não aceitar respostas vagas como:
"Erro de lógica"

---

## 🛠 Solução Aplicada

O que foi alterado para corrigir?

- Arquivos afetados
- Mudança estrutural?
- Ajuste de contrato?
- Ajuste de índice?
- Ajuste em consumer/worker?

---

## 🧪 Como foi testado

- [ ] Teste local
- [ ] Teste em homolog
- [ ] Teste automatizado adicionado
- [ ] Teste manual reproduzindo o bug

Descrever cenário de teste:

1.
2.
3.

---

## 🔄 Regressão

- Existe risco de quebrar outro fluxo?
- O fix altera comportamento esperado?

Se sim, explicar.

---

## 🗃 Impacto em Dados / Migration

- [ ] Não há
- [ ] Inclui migration
- [ ] Alteração de índice
- [ ] Alteração de contrato de API

Se houver migration:
- Evidência de execução
- Evidência de rollback

---

## 📊 Impacto Esperado

- [ ] Nenhum impacto adicional
- [ ] Mudança de comportamento
- [ ] Impacto de performance
- [ ] Correção crítica em produção

---

## 🔁 Rollback Plan

Caso o fix gere novo problema:

1.
2.

Se houver migration:
- Rollback é possível?
- Estratégia alternativa?

---

## 🏷 Versionamento

Este bugfix irá gerar:

- [ ] PATCH (vX.Y.Z → vX.Y.Z+1)

---

## 📎 Evidências

- Logs
- Prints
- Link para alerta
- Link para incidente (se houver)

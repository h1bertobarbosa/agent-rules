# 🚀 Release vX.Y.Z

**Tag:** vX.Y.Z   
**Data:** DD-MM-YYYY -   DD-MM-YYYY
**Responsável:** @owner  
**Ambiente:** production  

---

## 📝 PRs incluídos
- #pr123


---

## ⚠ Breaking Changes
- [ ] Não há
- [ ] Sim (descrever impacto)

---

## 🗃 Migrações
- [ ] Não há
- [ ] Migration incluída
    - Nome:
    - Compatível com versão anterior? (sim/não)
- [ ] Backfill necessário

⚠ Ordem obrigatória:
1.
2.
3.

---

## 📊 Impacto Esperado
- [ ] Nenhum impacto visível ao usuário
- [ ] Alteração de comportamento
- [ ] Impacto de performance
- [ ] Alteração de contrato de API

---

## 🧪 Validação Pós-Deploy
Checklist rápido:

- [ ] API principal responde 200
- [ ] Healthcheck OK
- [ ] Métricas estáveis
- [ ] Fila sem acúmulo / DLQ vazia
- [ ] Logs sem erros inesperados

---

## 🔁 Rollback Plan
Versão anterior: vX.Y.(Z-1)

Como reverter:
1.
2.

⚠ Observação sobre banco:
- Rollback é seguro?
- Precisa forward-fix?

---

## 📡 Monitoramento
Acompanhar por 30 min:

- Error rate
- Latência endpoint
- CPU/Mem
- Consumo de fila
- Alarmes ativos

---

## 📣 Comunicação
Mensagem padrão, avisar no canal xpto

> Release vX.Y.Z publicada em produção.
> Mudanças principais: ...
> Monitoramento ativo.

# Diretrizes e Melhores Práticas de Performance para Mongoose/MongoDB

Este documento define as regras obrigatórias para manipulação de dados utilizando o Mongoose. O objetivo é evitar problemas de performance, vazamentos de memória e comportamentos inesperados em alta concorrência.

## 1. Inserção de Dados (Create / Insert)

### 1.1 Inserções em Lote (Bulk Inserts)
Para inserts em massa, use `insertMany()` ou `bulkWrite()`:
- **NÃO FAÇA:** Inserções em loop utilizando `await Model.create()` ou `await doc.save()`. Isso gera "N" chamadas de rede para o MongoDB (o clássico problema de N+1).
- **FAÇA:** `Model.insertMany([docs])`. Isso realiza uma única chamada otimizada ao banco de dados.
- **Dica:** Se precisar de performance extrema sem rodar validações/middlewares do Mongoose, passe a opção `{ lean: true }` ou prefira utilizar `bulkWrite()`.

## 2. Índices e Integridade

### 2.1 Regra ESR para Índices Compostos
Ao criar índices em múltiplos campos, siga rigorosamente a regra **E-S-R (Equality, Sort, Range)** na ordem em que as chaves são declaradas:
1. **Equality:** Campos de busca exata (ex: `{ status: "ACTIVE" }`).
2. **Sort:** Campos usados na ordenação (ex: `{ createdAt: -1 }`).
3. **Range:** Campos de busca por intervalo (ex: `{ age: { $gte: 18 } }`).

### 2.2 Índices Únicos
Aproveite Índices Únicos (`unique: true`): 
Declarar a opção `unique: true` no Schema cria um índice único no MongoDB. Isso garante a integridade dos dados no próprio motor do banco, prevenindo *race conditions* que ocorreriam se a validação dependesse puramente do fluxo lógico (Node.js).

## 3. Atualização de Dados (Update)

Existe uma diferença fundamental entre usar hooks de documento e atualizar dados de forma otimizada. É preciso escolher com sabedoria entre `.save()` e `updateOne() / updateMany()`.

### 3.1 Quando usar `Document.save()`
- **Comportamento:** Carrega o documento na memória, executa validações de Schema, hooks de *pre/post save* (ex: gerar hash de uma senha) e mantém o controle de versão (`__v`).
- **Uso:** É mais seguro e disparará a lógica de domínio atrelada aos hooks, mas consome mais recursos de I/O e memória (exige uma busca seguida de gravação).

### 3.2 Quando usar `Model.updateOne()` / `Model.updateMany()`
- **Comportamento:** Executa a alteração direto no banco em **1 operação**. É muito mais performático.
- **Atenção:** **Não** aciona hooks de documento/instância por padrão e pode pular validações se não configurado explicitamente (`{ runValidators: true }`). Use-os preferencialmente quando a performance for mandatória e hooks não forem estritamente necessários para o campo alterado.

### 3.3 Operadores Atômicos
Utilize Operadores Atômicos do MongoDB: Sempre modifique apenas os campos necessários utilizando os operadores nativos (`$set`, `$inc`, `$push`, `$pull`), evitando trazer o documento para a memória para ser inteiramente sobrescrito.

## 4. Otimização de Leitura (Regra de Ouro)

### 4.1 O uso obrigatório do `.lean()`
Para consultas de leitura (`find`, `findOne`), se você **não precisa modificar e salvar o documento** e não fará uso dos métodos instanciados do Mongoose, **SEMPRE** adicione `.lean()`.

```typescript
// Exemplo Correto
const users = await User.find({ active: true }).lean().exec();
```

O `.lean()` instrui o Mongoose a retornar objetos JavaScript puros (Plain Old JavaScript Objects - POJO) em vez de instâncias pesadas da classe Mongoose Document. 
Isso acelera as requisições em **5x a 10x** e reduz drasticamente o uso de memória RAM (garbage collection).

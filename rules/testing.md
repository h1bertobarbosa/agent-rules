# Testing - Regras de Uso

## Comando correto

```bash
pnpm test   # jest --config test/jest-unit.json --runInBand
```

NÃO usar `jest` diretamente sem `--config`: o bloco `jest` do `package.json` aponta para `src/` e não encontra nenhum teste.

---

## Localização e estrutura

```
test/
  jest-unit.json          # configuração canônica (rootDir: .., roots: test/)
  unit/
    <módulo>/
      <módulo>.service.spec.ts
  factories/
    shipment-order.factory.ts
    packing-list.factory.ts
    packing-list-item.factory.ts
```

- Todos os testes ficam em `test/` — nunca em `src/`.
- Um arquivo de spec por serviço, agrupando cenários em blocos `describe` por método.

---

## Sandbox Sinon — obrigatório

```typescript
import sinon from 'sinon';

describe('MyService', () => {
  let sandbox: sinon.SinonSandbox;

  beforeEach(() => {
    sandbox = sinon.createSandbox();
  });

  afterEach(() => {
    sandbox.restore(); // SEMPRE — evita vazamento de estado entre testes
  });
});
```

- **Nunca** criar stubs globais fora do `beforeEach`.
- **Nunca** misturar `jest.useFakeTimers()` com `sinon.useFakeTimers()`.
- `sandbox.restore()` no `afterEach` é inegociável — sem ele os stubs de um teste contaminam o próximo.

---

## Instanciação do service (sem NestJS Testing)

Testes unitários NÃO usam `@nestjs/testing` nem `Test.createTestingModule`. O service é instanciado diretamente com dependências stubadas:

```typescript
service = new PackingListService(
  mockConnection as any,
  counterModel as any,
  packingListModel as any,
  packingListItemModel as any,
  shipmentOrderGateway as any,
);
```

O token `SHIPMENT_ORDER_GATEWAY` é irrelevante nos testes — passe o stub diretamente ao construtor.

---

## Padrão makeThenable — para queries Mongoose encadeadas

Mongoose retorna objetos com `.session()` e `.lean()` encadeados. Simule com:

```typescript
const makeThenable = (value: any) => {
  const promise = Promise.resolve(value);
  const result: any = {};
  result.session = sandbox.stub().returns(result);
  result.lean = sandbox.stub().resolves(value);
  result.then = promise.then.bind(promise);
  result.catch = promise.catch.bind(promise);
  return result;
};

// Uso:
packingListModel.findOne.returns(makeThenable(someDoc));
```

Para queries sem `.session()` (ex: `findOne(...).lean()` direto):

```typescript
const leanResolve = (value: any) => ({
  lean: sandbox.stub().resolves(value),
});

packingListModel.findOne.returns(leanResolve(someDoc));
```

---

## Factories — use sempre, nunca construa objetos manualmente

```typescript
import { ShipmentOrderFactory } from '../../factories/shipment-order.factory';
import { PackingListFactory } from '../../factories/packing-list.factory';
import { PackingListItemFactory } from '../../factories/packing-list-item.factory';

// Construção básica com defaults
const so = ShipmentOrderFactory.build();

// Com overrides pontuais
const pl = PackingListFactory.build({ status: 'cancelled' });

// Pedido multi-volume
const so3 = ShipmentOrderFactory.buildMultiVolume(3);

// Item com controle de volumes bipados (2 de 3 adicionados)
const item = PackingListItemFactory.buildMultiVolume(pl.id, pl.code, orderSale, 3, 2);
```

Não construa `{ id: 'x', carrier: { ... }, packages: [...] }` inline nos testes — fragiliza o suite quando os tipos mudam.

---

## Padrão AAA (Arrange / Act / Assert)

Cada teste verifica **exatamente um comportamento**:

```typescript
it('should throw when carrier does not match (RN-G-04)', async () => {
  // Arrange
  const shipmentOrder = ShipmentOrderFactory.build({ carrier: { id: 'A', name: 'Carrier A', document: '123' } });
  const existingPL = PackingListFactory.build({ carrier: { id: 'B', name: 'Carrier B' } });
  shipmentOrderGateway.findByTrackingCode.resolves(toGatewayData(shipmentOrder));
  packingListModel.findOne.returns(makeThenable(existingPL));

  // Act & Assert
  await expect(
    service.create(
      { data: { trackingCode: shipmentOrder.packages[0].trackingCode, packingListId: existingPL.id } },
      defaultUser,
    ),
  ).rejects.toThrow('carrier mismatch');
});
```

---

## Asserções de comportamento (side effects)

Além do valor de retorno, verifique chamadas a dependências:

```typescript
// Verifica que foi chamado exatamente uma vez
sinon.assert.calledOnce(packingListModel.create);

// Verifica argumento específico
sinon.assert.calledWith(
  packingListModel.updateOne,
  { id: pl.id },
  { status: 'awaiting-dispatch' },
  sinon.match.any,
);

// Verifica que NÃO foi chamado
sinon.assert.notCalled(packingListItemModel.create);
```

---

## Testes de exceção — verifique a mensagem, não só o throw

```typescript
// Bom — valida o erro específico
await expect(service.findOne('x')).rejects.toThrow('Packing list not found: x');

// Ruim — passa em qualquer erro
await expect(service.findOne('x')).rejects.toThrow(Error);
```

---

## Transações MongoDB nos testes

O `connection.transaction` é stubado para executar o callback diretamente:

```typescript
mockConnection = {
  transaction: (fn: any) => fn(mockSession),
};
mockSession = {};
```

Stubs que recebem `{ session }` como terceiro argumento devem usar `sinon.match.any` nessa posição.

---

## O que NÃO fazer

| ❌ Errado | ✅ Certo |
|---|---|
| `Test.createTestingModule(...)` | Instanciar o service diretamente |
| `jest.mock(...)` global | `sandbox.stub()` com `sandbox.restore()` no `afterEach` |
| Construir objetos inline nos testes | Usar as factories de `test/factories/` |
| `jest` sem `--config` | `pnpm test` (usa `test/jest-unit.json`) |
| Stub fora do sandbox | Sempre `sandbox.stub()` |
| `.rejects.toThrow(Error)` genérico | Verificar a mensagem exata |
| Múltiplos comportamentos num `it` | Um `it` = um comportamento |

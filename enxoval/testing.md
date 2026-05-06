# `@enxoval/testing`

> **TL;DR** — Re-exports do Vitest + `generate()` para gerar valores aleatórios válidos a partir de schemas + `itCases()` para property-based testing (50–150 iterações, coleta todas as falhas). Também exporta `createTestDataSource` do `@enxoval/db`.

---

## Versão atual: 1.0.4

---

## Imports disponíveis

```ts
import {
  describe, it, expect,
  beforeAll, beforeEach, afterAll, afterEach,
  test,              // { fn, mock, spy, clearAll }
  generate,          // gerador de valores aleatórios
  itCases,           // property-based test runner
  createTestDataSource,  // SQLite in-memory
} from '@enxoval/testing';
```

---

## `generate(schema, overrides?)`

Gera um valor aleatório válido a partir de um schema `@enxoval/types`.

```ts
import { generate } from '@enxoval/testing';
import { Journey } from '../../src/model/journey';

// Valor totalmente aleatório
const journey = generate(Journey);

// Com campo fixo (override)
const journey = generate(Journey, { status: 'active' });

// Extrai apenas um campo
const id = generate(Journey).id;
```

### Tipos gerados por `_type` do FieldParser

| `_type` | Valor gerado |
|---------|-------------|
| `string` | string aleatória de 8 chars alfanuméricos |
| `uuid` | `crypto.randomUUID()` |
| `number` | inteiro aleatório em [0, 999] |
| `boolean` | 50% `true`, 50% `false` |
| `date` | ISO string nos últimos 30 dias |
| `literal` | um dos valores permitidos, escolhido aleatoriamente |
| `nullable` | 50% `null`, 50% valor do parser interno |
| `nested` | recursivo — gera valor do schema aninhado |

**Requisito:** o schema deve ter `_fields` (criado por `createSchema`). Schemas sem `_fields` (ex: `field.array()`, TypeORM DbWire) não funcionam com `generate()`.

### Padrão para testes de adapter (fromDbWire)

```ts
import { generate } from '@enxoval/testing';
import { EventRecord } from '../../../src/model/event-record';
import { fromDbWire, toDbWire } from '../../../src/adapters/event-record';

describe('fromDbWire', () => {
  it('converts db fields to domain', () => {
    const record = generate(EventRecord);
    const wire = toDbWire(record);
    const back = fromDbWire(wire);
    expect(back.id).toBe(record.id);
    expect(back.createdAt).toBeInstanceOf(Date);
  });
});
```

---

## `itCases(desc, schema, fn)`

Property-based test: gera N inputs aleatórios (50–150), roda `fn` em cada um, coleta todas as falhas e lança um erro consolidado.

```ts
import { itCases } from '@enxoval/testing';
import { EventRecordInput } from '../../../src/model/event-record';
import { toDbWire } from '../../../src/adapters/event-record';

describe('toDbWire', () => {
  itCases('maps journeyId', EventRecordInput, (input) => {
    expect(toDbWire(input).journey_id).toBe(input.journeyId);
  });

  itCases('maps name', EventRecordInput, (input) => {
    expect(toDbWire(input).name).toBe(input.name);
  });
});
```

### Com overrides

```ts
itCases('maps status active', Journey, { status: 'active' }, (input) => {
  expect(toDbWire(input).status).toBe('active');
});
```

### Assinatura completa

```ts
// Sem overrides
itCases(description: string, schema: Schema<T>, fn: (input: T) => void | Promise<void>): void

// Com overrides
itCases(description: string, schema: Schema<T>, overrides: Partial<Record<string, unknown>>, fn: (input: T) => void | Promise<void>): void
```

### Mensagem de falha

```
3/87 casos falharam:
  [12] {"id":"...","journeyId":"..."} → Expected "abc" to be "xyz"
  [34] {"id":"...","journeyId":"..."} → Expected "def" to be "xyz"
  [67] {"id":"...","journeyId":"..."} → Expected "ghi" to be "xyz"
```

---

## `test` — mocks em testes de integração

```ts
import { test } from '@enxoval/testing';

// Mockar um módulo inteiro
test.mock('../../src/diplomat/http-client/atreides', () => ({
  getUser: test.fn().mockResolvedValue({ userId: 'abc', role: 'student' }),
}));

// Espiar uma função
const spy = test.spy(someModule, 'someFunction');

// Limpar todos os mocks entre testes
beforeEach(() => test.clearAll());
```

**Regra:** usar sempre `test.mock` / `test.fn` / `test.spy` — **nunca `vi.mock` / `vi.fn` diretamente**. O `testAliasPlugin` exige os aliases.

---

## `createTestDataSource` — banco SQLite in-memory

```ts
import { createTestDataSource } from '@enxoval/testing';
import { JourneySchema } from '../../src/wire/journey';

const AppDataSource = createTestDataSource([JourneySchema]);

beforeAll(async () => { await AppDataSource.initialize(); });
afterAll(async () => { await AppDataSource.destroy(); });
afterEach(async () => {
  await AppDataSource.getRepository(JourneyDbWire).clear();
});
```

Cria um banco SQLite in-memory com `synchronize: true` — tabelas criadas automaticamente.

---

## Padrão de arquivo de teste de unidade (adapter)

```ts
import { describe, it, expect, itCases, generate } from '@enxoval/testing';
import { Journey, JourneyRecord } from '../../../src/model/journey';
import { fromDbWire, toDbWire } from '../../../src/adapters/journey';

describe('fromDbWire', () => {
  it('converts db record to domain', () => {
    const record = generate(JourneyRecord);
    // assign wire fields manually — DbWire tem no _fields
    const wire = { id: record.id, student_id: record.studentId, ... };
    const domain = fromDbWire(wire);
    expect(domain.id).toBe(record.id);
    expect(domain.createdAt).toBeInstanceOf(Date);
  });
});

describe('toDbWire', () => {
  itCases('maps studentId → student_id', JourneyRecord, (input) => {
    expect(toDbWire(input).student_id).toBe(input.studentId);
  });
  itCases('maps status', JourneyRecord, (input) => {
    expect(toDbWire(input).status).toBe(input.status);
  });
});
```

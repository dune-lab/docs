# `@enxoval/types`

> **TL;DR** — Schemas tipados (`createSchema` + `field.*`), wrappers de função com validação (`asyncFn`/`fn`), UUID branding e hierarquia de erros (`AppError`). Base de todos os outros pacotes.

---

## Versão atual: 1.0.27

---

## `createSchema` + `field.*`

Cria um schema com parsing e validação automáticos. Cada campo é um `FieldParser<T>`.

```ts
import { createSchema, field } from '@enxoval/types';

export const Journey = createSchema({
  id:          field.uuid(),
  studentId:   field.uuid(),
  status:      field.literal('active', 'completed', 'cancelled'),
  currentStep: field.number(),
  createdAt:   field.date(),
  label:       field.string(),
  finished:    field.boolean(),
  metadata:    field.nullable(field.string()),
});
```

### Campos disponíveis

| Parser | Tipo TS | Validação |
|--------|---------|-----------|
| `field.uuid()` | `UUID` | formato `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| `field.string()` | `string` | qualquer string |
| `field.number()` | `number` | typeof === 'number', !isNaN |
| `field.boolean()` | `boolean` | typeof === 'boolean' |
| `field.date()` | `Date` | Date, string ISO ou timestamp numérico |
| `field.literal(...values)` | `'a' \| 'b' \| ...` | valor deve ser um dos listados |
| `field.nullable(parser)` | `T \| null` | null/undefined → null; senão delega ao parser interno |
| `field.nested(schema)` | `T` | executa `schema.parse()` no valor |
| `field.array(schema)` | `Schema<T[]>` | array de T; retorna Schema, não FieldParser |

### `nullable()` (top-level — para output de asyncFn)

```ts
import { nullable } from '@enxoval/types';

export const getStudent = asyncFn(Input, nullable(Student), async (input) => {
  if (res.status === 404) return null;
  return Student.parse(await res.json());
});
```

### `schema.parse(data)`

Valida e converte `data` para o tipo do schema. Lança `TypeError` se algum campo for inválido.

```ts
const journey = Journey.parse(await res.json()); // necessário — res.json() retorna any
```

---

## `asyncFn` / `fn`

Wrappers obrigatórios para funções exportadas nas camadas de domínio. Garantem que inputs e outputs são validados nos limites.

### Assinaturas

```ts
// Com output schema (3 argumentos)
asyncFn(InputSchema, OutputSchema, async (input) => { ... })
fn(InputSchema, OutputSchema, (input) => { ... })

// Sem output (void — 2 argumentos)
asyncFn(InputSchema, async (input) => { ... })
```

### Exemplo completo

```ts
const Input = createSchema({ studentId: field.uuid() });

export const startJourney = asyncFn(Input, Journey, async (input) => {
  const res = await fetch(`${ODYSSEY_URL}/journeys`, {
    method: 'POST',
    body: JSON.stringify({ studentId: input.studentId }),
  });
  if (!res.ok) throw new NotFoundError('odyssey: journey not found');
  return Journey.parse(await res.json()); // .parse() explícito — obrigatório
});
```

**Regra crítica:** sempre usar `Schema.parse(await res.json())` — nunca retornar `res.json()` direto (`any` impede inferência do tipo `O`).

### Camadas onde `asyncFn`/`fn` é obrigatório

`src/adapters/`, `src/controllers/`, `src/logic/`, `src/model/`, `src/wire/`, `src/diplomat/http-client/`

---

## UUID

UUID branded type — evita passar strings comuns onde UUID é esperado.

```ts
import { UUID, isUUID, toUUID, asUUID, UUID_REGEX } from '@enxoval/types';

isUUID('abc')                         // false
isUUID('550e8400-e29b-41d4-a716-...')  // true

toUUID('invalid')   // lança Error
asUUID('any-string') // cast sem validação (evitar — usar apenas quando a origem já é confiável)
```

| Função | Comportamento |
|--------|--------------|
| `isUUID(str)` | type guard — retorna `true` se for UUID válido |
| `toUUID(str)` | valida e retorna `UUID`, lança `Error` se inválido |
| `asUUID(str)` | cast direto, sem validação |
| `UUID_REGEX` | `/^[0-9a-f]{8}-[0-9a-f]{4}-...$/ i` |

---

## Hierarquia `AppError`

Erros tipados que o `@enxoval/http` mapeia automaticamente para status HTTP.

```ts
import { AppError, NotFoundError, ConflictError, ValidationError,
         UnprocessableError, UnauthorizedError } from '@enxoval/types';
```

| Classe | Status HTTP |
|--------|-------------|
| `UnauthorizedError` | 401 |
| `ValidationError` | 400 |
| `NotFoundError` | 404 |
| `ConflictError` | 409 |
| `UnprocessableError` | 422 |
| `AppError` (base) | 500 |

**Regra:** http-clients devem relançar erros downstream como `AppError` específico — nunca `Error` genérico. `Error` genérico não é instância de `AppError` e o handler retorna 500.

```ts
if (res.status === 401) throw new UnauthorizedError('E-mail ou senha inválidos');
if (res.status === 409) throw new ConflictError('E-mail já está em uso');
if (res.status === 404) throw new NotFoundError('Recurso não encontrado');
```

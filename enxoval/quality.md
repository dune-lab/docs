# `@enxoval/quality`

> **TL;DR** — Configurações compartilhadas de ESLint e Prettier. `base()` proíbe imports de frameworks (use os wrappers `@enxoval/*`) e exige `asyncFn`/`fn` nas camadas de domínio. `boundaries()` define quais camadas podem importar quais.

---

## Versão atual: 1.0.1

---

## `base()`

Regras ESLint aplicadas a todos os serviços.

```ts
import { base } from '@enxoval/quality';

export default [...base()];
```

### Regras incluídas

**1. Imports de framework proibidos** (`no-restricted-imports`)

Os wrappers `@enxoval/*` devem ser usados em vez dos pacotes brutos:

| Pacote proibido | Mensagem |
|-----------------|----------|
| `typeorm` | Use lib/db instead. |
| `kafkajs` | Use lib/messaging instead. |
| `fastify` | Use lib/http instead. |
| `pino` | Use lib/observability instead. |
| `vitest` | Use lib/testing instead. |

**2. `asyncFn`/`fn` obrigatório nas camadas de domínio** (`no-restricted-syntax`)

Nas camadas `src/adapters/**`, `src/controllers/**`, `src/logic/**`, `src/model/**`, `src/wire/**`:

| Sintaxe proibida | Erro |
|---|---|
| `function foo() {}` | Use fn() or asyncFn() instead of function declarations. |
| `const foo = () => {}` | Use fn() or asyncFn() instead of arrow function variables. |
| `const foo = function() {}` | Use fn() or asyncFn() instead of function expressions. |

**3. Prettier** — integrado via `eslint-config-prettier` para desativar regras conflitantes.

---

## `boundaries(layers)`

Define as fronteiras de arquitetura: quais camadas podem importar quais.

```ts
import { boundaries } from '@enxoval/quality';

export default [
  ...base(),
  ...boundaries([
    { name: 'diplomat', pattern: ['src/diplomat/**'], allow: ['controllers', 'wire'] },
    { name: 'controllers', pattern: ['src/controllers/**'], allow: ['logic', 'model'] },
    { name: 'logic', pattern: ['src/logic/**'], allow: ['model'] },
    { name: 'adapters', pattern: ['src/adapters/**'], allow: ['model', 'db-wire', 'wire'] },
    { name: 'db', pattern: ['src/db/*.ts'], allow: ['db-wire', 'model', 'adapters'] },
    { name: 'db-wire', pattern: ['src/db/wire/**'], allow: [] },
    { name: 'model', pattern: ['src/model/**'], allow: [] },
    { name: 'wire', pattern: ['src/wire/**'], allow: [] },
  ]),
];
```

### Interface `Layer`

```ts
interface Layer {
  name: string;      // identificador da camada
  pattern: string[]; // globs dos arquivos desta camada
  allow: string[];   // nomes das camadas que esta pode importar
}
```

### Configuração por serviço

Cada serviço tem seu `eslint.config.ts` com as camadas específicas de sua arquitetura.

**odyssey / atreides / persona** (com banco + Kafka):
```ts
boundaries([
  { name: 'diplomat',    pattern: ['src/diplomat/**'],     allow: ['controllers', 'wire'] },
  { name: 'controllers', pattern: ['src/controllers/**'],  allow: ['logic', 'model'] },
  { name: 'logic',       pattern: ['src/logic/**'],        allow: ['model'] },
  { name: 'adapters',    pattern: ['src/adapters/**'],     allow: ['model', 'db-wire', 'wire'] },
  { name: 'db',          pattern: ['src/db/*.ts'],         allow: ['db-wire', 'model', 'adapters'] },
  { name: 'db-wire',     pattern: ['src/db/wire/**'],      allow: [] },
  { name: 'model',       pattern: ['src/model/**'],        allow: [] },
  { name: 'wire',        pattern: ['src/wire/**'],         allow: [] },
])
```

**imperium** (BFF — sem banco, com http-clients):
```ts
boundaries([
  { name: 'diplomat-server', pattern: ['src/diplomat/http-server/**'], allow: ['controllers', 'wire', 'model'] },
  { name: 'diplomat-client', pattern: ['src/diplomat/http-client/**'], allow: ['wire', 'model'] },
  { name: 'controllers',     pattern: ['src/controllers/**'],          allow: ['diplomat-client', 'model', 'wire'] },
  { name: 'adapters',        pattern: ['src/adapters/**'],             allow: ['model', 'wire'] },
  { name: 'model',           pattern: ['src/model/**'],                allow: [] },
  { name: 'wire',            pattern: ['src/wire/**'],                 allow: [] },
])
```

---

## `prettier`

Configuração Prettier compartilhada. Usada no `prettier.config.ts` de cada serviço.

```ts
import { prettier } from '@enxoval/quality';

export default prettier;
```

### Valores

| Opção | Valor |
|-------|-------|
| `tabWidth` | `2` |
| `printWidth` | `100` |
| `singleQuote` | `true` |
| `semi` | `true` |
| `trailingComma` | `'all'` |
| `bracketSpacing` | `true` |

---

## `eslint.config.ts` mínimo por serviço

```ts
import { base, boundaries } from '@enxoval/quality';

export default [
  ...base(),
  ...boundaries([
    // camadas do serviço
  ]),
];
```

O ESLint roda automaticamente no pre-commit via husky (hook `format:staged`).

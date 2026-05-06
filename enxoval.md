# @enxoval/* — Bibliotecas Compartilhadas

> **TL;DR** — Todos os serviços usam pacotes `@enxoval/*` do monorepo [dune-lab/enxoval](https://github.com/dune-lab/enxoval). Publicar uma nova versão = criar tag `v1.0.XX` → GitHub Actions publica no npm e abre PRs de bump em todos os repos automaticamente.

---

## Pacotes

| Pacote | Versão | Descrição |
|--------|--------|-----------|
| `@enxoval/types` | 1.0.27 | Schemas — `createSchema`, `field.*`, `asyncFn`, `fn`, UUID branding |
| `@enxoval/http` | 1.0.26 | Fastify wrapper — `listen`, `get`, `post`, `sseRoute`, hierarquia `AppError` |
| `@enxoval/db` | 1.0.3 | TypeORM wrapper — `defineEntity`, migrations, data source |
| `@enxoval/messaging` | 1.0.2 | Kafka wrapper — `subscribe`, `publish`, retry + DLQ automático |
| `@enxoval/auth` | 1.0.2 | Middleware JWT Bearer para Fastify |
| `@enxoval/observability` | 1.0.2 | Logger Pino — JSON estruturado com `service`, `cid`, `durationMs` |
| `@enxoval/testing` | 1.0.3 | Re-exports Vitest + `generate()` + `itCases()` + SQLite in-memory |

---

## Como Publicar Uma Nova Versão

```bash
# 1. Editar a versão no package.json do pacote que mudou
# 2. Commitar
git add enxoval/types/package.json
git commit -m "feat(types): ..."

# 3. Criar tag e push — pipeline roda automaticamente
git tag v1.0.28
git push origin main
git push origin v1.0.28
```

**Não usar `npm publish` diretamente** — o token npm está apenas no GitHub Actions.

### O que acontece após o push da tag

```
[job: publish]   → build + npm publish (pula versões já publicadas)
       │
[job: discover]  → lê package.json de cada repo via GitHub API
       │
[job: bump]      → para cada repo que usa @enxoval/*:
                     npm install @enxoval/...@nova-versao
                     git commit + gh pr create
```

Cada serviço recebe um PR pronto para merge em minutos.

---

## `asyncFn` — Padrão Obrigatório

Todo export de domínio usa `asyncFn` (ou `fn`). Isso garante input/output tipados em cada fronteira:

```ts
const Input = createSchema({ studentId: field.uuid() });

export const startJourney = asyncFn(Input, Journey, async (input) => {
  const res = await fetch(`${ODYSSEY_URL}/journeys`, {
    method: 'POST',
    body: JSON.stringify({ studentId: input.studentId }),
  });
  if (!res.ok) throw new NotFoundError('odyssey: journey not found');
  return Journey.parse(await res.json()); // .parse() explícito — necessário para TS inferir O
});
```

**Regra crítica:** sempre `Schema.parse(await res.json())` — nunca retornar `res.json()` direto (`any` impede inferência do tipo genérico).

Camadas onde `asyncFn`/`fn` é obrigatório: `src/adapters/`, `src/controllers/`, `src/logic/`, `src/model/`, `src/wire/`, `src/diplomat/http-client/`.

---

## Hierarquia `AppError`

`@enxoval/http` fornece erros tipados que mapeiam para status HTTP:

| Erro | Status |
|------|--------|
| `UnauthorizedError` | 401 |
| `NotFoundError` | 404 |
| `ConflictError` | 409 |
| `ValidationError` | 400 |
| `UnprocessableError` | 422 |

http-clients devem relançar erros downstream como `AppError` específico — nunca `Error` genérico (vira 500).

---

## `generate()` e `itCases()` — Testes

`@enxoval/testing` exporta dois helpers para property-based testing:

```ts
import { generate, itCases } from '@enxoval/testing';

// Gera um valor aleatório válido a partir de um schema
const journey = generate(Journey);
const journey = generate(Journey, { studentId: 'uuid-fixo' }); // com overrides

// Roda a função de teste 50–150× com inputs aleatórios, coleta todas as falhas
itCases('maps journeyId', EventRecordInput, (input) => {
  expect(toDbWire(input).journey_id).toBe(input.journeyId);
});
```

Ver padrões completos de uso em `CLAUDE.md`.

# @enxoval/* — Referência Completa

> **TL;DR** — Monorepo de bibliotecas compartilhadas do dune-lab. Publicado no npm via GitHub Actions ao criar uma tag `v*`. Cada serviço depende de um subset dessas libs.

---

## Pacotes

| Pacote | Versão | Arquivo |
|--------|--------|---------|
| [`@enxoval/types`](./types.md) | 1.0.27 | Schemas, `asyncFn`, erros, UUID |
| [`@enxoval/http`](./http.md) | 1.0.26 | Fastify wrapper, SSE, error mapping |
| [`@enxoval/db`](./db.md) | 1.0.3 | TypeORM helpers, data sources, migrations |
| [`@enxoval/messaging`](./messaging.md) | 1.0.2 | Kafka producer/consumer, DLQ, retry |
| [`@enxoval/auth`](./auth.md) | 1.0.2 | JWT middleware, contexto de usuário |
| [`@enxoval/observability`](./observability.md) | 1.0.2 | Pino logger, correlation IDs |
| [`@enxoval/testing`](./testing.md) | 1.0.4 | Vitest + `generate()` + `itCases()` |

---

## Como Publicar

```bash
# Editar versão no package.json do pacote que mudou, depois:
git add enxoval/<pacote>/package.json
git commit -m "feat(<pacote>): ..."
git tag v1.0.28
git push origin main && git push origin v1.0.28
```

GitHub Actions publica no npm e abre PRs de bump em todos os repos consumidores automaticamente. **Nunca usar `npm publish` diretamente.**

---

## Regras Gerais

- Todo export de domínio usa `asyncFn` ou `fn` de `@enxoval/types` — nunca `async function` ou `const x = async () =>` nas camadas `adapters/`, `controllers/`, `logic/`, `model/`, `wire/`, `diplomat/http-client/`.
- Erros entre serviços devem ser relançados como `AppError` específico (`UnauthorizedError`, `ConflictError`…) — nunca `Error` genérico (resulta em 500).
- `Schema.parse(await res.json())` sempre explícito — `res.json()` retorna `any` e quebra a inferência de tipo do `asyncFn`.

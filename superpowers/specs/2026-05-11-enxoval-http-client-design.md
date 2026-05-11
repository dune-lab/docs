# `@enxoval/http` — HTTP Client Transparente

> **Para agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Eliminar o boilerplate de HTTP client entre serviços da plataforma dune-lab — URL, token, parse e erro resolvidos automaticamente pelo `@enxoval/http`.

**Architecture:** JSON declara o routing (alias → service + method + path). Um arquivo `src/http.ts` por serviço registra os schemas de saída. A função `call(alias, opts)` resolve URL, injeta token, substitui path params, parseia a resposta e converte status em AppError — sem nenhum argumento extra no call site.

**Tech Stack:** TypeScript, AsyncLocalStorage (Node.js built-in), `@enxoval/http`, `@enxoval/auth`, `@enxoval/types`.

---

## 1. Shape do JSON de serviço

Cada serviço que faz chamadas HTTP a outros serviços declara os aliases na seção `http` do seu JSON de configuração (ex: `imperium.json`):

```json
{
  "http": {
    "getUser":      { "service": "atreides", "method": "GET",  "path": "/users/:userId" },
    "createUser":   { "service": "atreides", "method": "POST", "path": "/users" },
    "getStudent":   { "service": "persona",  "method": "GET",  "path": "/students/by-user/:userId" },
    "listStudents": { "service": "persona",  "method": "GET",  "path": "/students" },
    "startJourney": { "service": "odyssey",  "method": "POST", "path": "/journeys" },
    "listJourneys": { "service": "odyssey",  "method": "GET",  "path": "/journeys" }
  },
  "kafka_topics": {}
}
```

**Campos por alias:**
- `service` — nome em lowercase do serviço destino (ex: `"atreides"`). Usado para derivar a env var `ATREIDES_URL`.
- `method` — `GET`, `POST`, `PUT`, `PATCH` ou `DELETE`.
- `path` — path relativo à URL base do serviço. Params de path usam `:param`.

---

## 2. `defineHttpAliases` — registro de schemas

Arquivo único por serviço (`src/http.ts`) que registra o schema de saída de cada alias. Lê o JSON do serviço em disco e devolve um `call` tipado.

```typescript
// imperium/src/http.ts
import { defineHttpAliases } from '@enxoval/http';
import { field } from '@enxoval/types';
import { UserData } from './model/me';
import { Student } from './model/student';
import { Journey } from './model/journey';

export const { call } = defineHttpAliases({
  getUser:      UserData,
  createUser:   UserData,
  getStudent:   Student,
  listStudents: field.array(Student),
  startJourney: Journey,
  listJourneys: field.array(Journey),
});
```

**`defineHttpAliases(schemaMap)`**
- Lê `${SERVICE_NAME}.json` do `cwd` (mesmo padrão de `ensureTopics`)
- Valida que todo alias declarado no JSON tem schema correspondente no `schemaMap` (erro em startup se faltar)
- Retorna `{ call }` — função tipada onde o tipo de retorno é inferido do alias

---

## 3. Clientes HTTP — API do developer

```typescript
// imperium/src/diplomat/http-client/atreides.ts
import { call } from '../http';

export const getUser = ({ userId }: { userId: string }) =>
  call('getUser', { payload: { userId } })
// TypeScript infere: Promise<UserData>

export const createUser = (body: { name: string; email: string }) =>
  call('createUser', { payload: body })
// TypeScript infere: Promise<UserData>
```

**Assinatura do `call`:**

```typescript
call(alias: Alias, opts?: { payload?: Record<string, unknown>; headers?: Record<string, string> }): Promise<Output>
```

**Resolução do `payload`:**
1. Campos que coincidem com params de path (`:userId`) → substituídos na URL
2. `GET` / `DELETE` → campos restantes viram query string (`?key=value`)
3. `POST` / `PUT` / `PATCH` → campos restantes viram body JSON (`Content-Type: application/json` adicionado automaticamente)

---

## 4. Injeção de token — fluxo HTTP e Kafka

### Fluxo HTTP (request de usuário)

O `tokenStorage` vive em `@enxoval/http` (evita dependência circular). O middleware de auth em `@enxoval/auth` importa e seta:

```typescript
// @enxoval/http — exportado junto com defineHttpAliases
export const tokenStorage = new AsyncLocalStorage<string>();

// @enxoval/auth — setupAuth(), após jwt.verify():
import { tokenStorage } from '@enxoval/http';
tokenStorage.run(rawToken, () => next());
```

O `call` lê do storage internamente:

```typescript
const token = tokenStorage.getStore();
headers['Authorization'] = `Bearer ${token}`;
```

### Fluxo Kafka (sem request HTTP)

Quando `tokenStorage.getStore()` retorna `undefined` (consumer Kafka, cronjob, etc.), o `call` usa o service token:

```typescript
headers['X-Service-Token'] = process.env.JWT_SECRET;
```

### Validação nos serviços receptores

`@enxoval/auth` aceita ambos os mecanismos — adição ao `setupAuth()` existente:

```typescript
// aceita X-Service-Token como alternativa ao Bearer
const serviceToken = request.headers['x-service-token'];
if (serviceToken && serviceToken === process.env.JWT_SECRET) {
  return next(); // serviço interno confiável
}
// caso contrário, segue validação JWT normal
```

---

## 5. Resolução de URL

Convenção: `service: "atreides"` → `process.env.ATREIDES_URL`.

```typescript
const baseUrl = process.env[`${service.toUpperCase()}_URL`];
if (!baseUrl) throw new Error(`Missing env var: ${service.toUpperCase()}_URL`);
```

Em testes: não chega a executar (módulo mockado com `test.mock`). Em produção: setado automaticamente pelo forge (ver seção 7).

---

## 6. Tratamento de erros

O `call` converte automaticamente status HTTP em `AppError`:

| Status | AppError |
|--------|----------|
| 401 | `UnauthorizedError` |
| 404 | `NotFoundError` |
| 409 | `ConflictError` |
| 422 | `UnprocessableError` |
| 400 | `ValidationError` |
| 5xx | `Error` genérico |

Erros com body JSON do serviço remoto: mensagem extraída de `body.message` se presente.

---

## 7. Integração com o Forge

O `service-generator.ts` do forge lê a seção `http` do JSON gerado e extrai os `service` distintos. Para cada um, adiciona automaticamente:

- **GitHub Actions variable**: `ATREIDES_URL = http://atreides:PORT`
- **`docker-compose.yml`** (via PR no platform): env var `ATREIDES_URL: http://atreides:PORT` no serviço que está sendo criado

O valor da URL é derivado do `docker-compose.yml` existente no platform: busca o serviço pelo nome e usa a porta mapeada internamente.

O template base (`forge/src/templates/base/`) ganha suporte à seção `http` no JSON gerado — com aliases de exemplo comentados para o developer preencher.

---

## 8. Testes — zero mudança no padrão atual

Os testes de integração continuam mockando o módulo do client diretamente:

```typescript
test.mock('../../src/diplomat/http-client/atreides', () => ({
  getUser: test.fn().mockResolvedValue({ id: '123', name: 'João', email: 'j@j.com', ... }),
  createUser: test.fn().mockResolvedValue({ id: '456', name: 'Maria', ... }),
}));
```

Nenhuma env var necessária. O `call` nunca é executado nos testes — o mock substitui a função exportada pelo client.

---

## 9. Escopo de mudanças

| Pacote / Repo | O que muda |
|---------------|------------|
| `@enxoval/http` | Novo export: `defineHttpAliases`. Implementação interna do `call` (fetch, path params, token, erro). |
| `@enxoval/auth` | Middleware seta token em `AsyncLocalStorage`. Aceita `X-Service-Token` como auth interna. |
| `imperium` | `imperium.json` ganha seção `http`. Novo `src/http.ts`. Clientes reescritos com `call`. |
| `forge` | `service-generator.ts` lê `http.services` e seta env vars. Template base atualizado. |
| Outros serviços | Adotam o padrão quando fizerem chamadas HTTP (janus → atreides já pode migrar). |

# `@enxoval/http`

> **TL;DR** — Wrapper Fastify para registrar rotas HTTP, SSE e injetar requests em testes. Inclui logging automático de requests/responses, mapeamento de `AppError` → status HTTP e suporte a CORS.

---

## Versão atual: 1.0.27

---

## Rotas HTTP

```ts
import { get, getWith, getWithAuth, post, postOk, put, patch, del } from '@enxoval/http';
```

| Função | Método | Status padrão | Quando usar |
|--------|--------|---------------|-------------|
| `get(path, handler)` | GET | 200 | Sem parâmetros de rota |
| `getWith<TParams>(path, handler)` | GET | 200 | Com parâmetros de rota (ex: `/users/:id`) |
| `getWithAuth(path, handler)` | GET | 200 | Precisa do header `authorization` |
| `post<TBody>(path, handler)` | POST | **201** | Criação de recursos |
| `postOk<TBody>(path, handler)` | POST | **200** | POST sem criação (ex: login, ações) |
| `put<TBody>(path, handler)` | PUT | 200 | Atualização completa |
| `patch<TBody>(path, handler)` | PATCH | 200 | Atualização parcial |
| `del(path, handler)` | DELETE | 204 | Remoção |
| `html(path, handler)` | GET | 200 | Retorna HTML (`text/html`) |

### Exemplos

```ts
// GET sem parâmetros
get('/health', async () => ({ status: 'ok' }));

// GET com parâmetros de rota
getWith<{ id: string }>('/users/:id', async ({ id }) => getUser({ id }));

// GET com token no header
getWithAuth('/me', async (authorization) => getMe({ authorization }));

// POST cria recurso (201)
post('/users', async (body) => createUser(body));

// POST ação/login (200)
postOk('/auth/login', async (body) => login(body));
```

---

## SSE (Server-Sent Events)

```ts
import { sseRoute } from '@enxoval/http';

sseRoute<{ journeyId: string }, { token: string }>(
  '/journeys/:journeyId/stream',
  async (params, query, send, signal) => {
    signal.addEventListener('abort', cleanup);
    emitter.on('update', (event) => send({ type: event.name, data: event.payload }));
    await new Promise((resolve) => signal.addEventListener('abort', resolve));
  }
);
```

| Parâmetro do handler | Tipo | Descrição |
|---------------------|------|-----------|
| `params` | `TParams` | Parâmetros de rota |
| `query` | `TQuery` | Query string |
| `send` | `(data: object) => void` | Envia um evento SSE |
| `signal` | `AbortSignal` | Disparado quando o cliente desconecta |

Headers automáticos: `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `Connection: keep-alive`, `Access-Control-Allow-Origin: *`.

---

## `listen` / `close`

```ts
import { listen, close } from '@enxoval/http';

await listen(3000, '0.0.0.0', async () => {
  // setup: conectar Kafka, banco, etc.
  await connect();
  await AppDataSource.initialize();
});

await close(); // fecha o servidor
```

O callback de setup **não roda** quando `CI=true` (para testes de integração).

---

## `inject` (testes de integração)

```ts
import { inject } from '@enxoval/http';

const res = await inject({
  method: 'POST',
  url: '/users',
  body: { email: 'a@b.com', password: '123456' },
  headers: { authorization: `Bearer ${token}` },
});

expect(res.statusCode).toBe(201);
const user = JSON.parse(res.body);
```

---

## `addPreHandler`

```ts
import { addPreHandler } from '@enxoval/http';

// Usado internamente pelo @enxoval/auth para injetar o middleware JWT
addPreHandler((request, reply, done) => {
  // lógica de pre-handler
  done();
});
```

---

## Tratamento de erros automático

O Fastify do `@enxoval/http` tem um `setErrorHandler` global:

| Tipo de erro | Status | Corpo |
|---|---|---|
| `AppError` (+ subclasses) | 401 / 400 / 404 / 409 / 422 | `{ error: 'NotFoundError', message: '...' }` |
| `TypeError` (validação de schema) | 400 | `{ error: 'ValidationError', message: '...' }` |
| Qualquer outro `Error` | 500 | `{ error: 'InternalServerError', message: 'Internal server error' }` |

**Rotas silenciosas** (sem log): `/health`, `/contracts`, qualquer `OPTIONS`.

---

## Cliente HTTP transparente

Elimina o boilerplate de chamadas HTTP entre serviços: resolve URL, injeta token, substitui path params, converte erros HTTP em `AppError`.

### `defineHttpAliases(schemas)`

Lê `${SERVICE_NAME}.json` do cwd (campo `http`), valida os aliases e retorna uma função `call()` tipada. A leitura é **lazy** — ocorre na primeira chamada a `call()`, não no import.

```ts
// src/diplomat/http-client/index.ts
import { defineHttpAliases } from '@enxoval/http';
import { asyncFn, nullable, field } from '@enxoval/types';
import { User } from '../model/user';
import { GetUserIn } from '../wire/in/get-user';

const { call } = defineHttpAliases({
  getUser: User,
  listUsers: field.array(User),
  getProfile: nullable(User),
});

export const getUser = asyncFn(GetUserIn, User, (input) =>
  call('getUser', { payload: input }));
```

### Arquivo de configuração `${SERVICE_NAME}.json`

```json
{
  "http": {
    "getUser":    { "service": "atreides", "method": "GET",  "path": "/users/:userId" },
    "listUsers":  { "service": "atreides", "method": "GET",  "path": "/users" },
    "getProfile": { "service": "persona",  "method": "GET",  "path": "/profiles/:id", "nullable": true },
    "login":      { "service": "janus",    "method": "POST", "path": "/auth/login", "auth": false }
  },
  "kafka_topics": {}
}
```

| Campo | Descrição |
|-------|-----------|
| `service` | Nome do serviço — resolve `${SERVICE.toUpperCase()}_URL` do ambiente |
| `method` | `GET`, `POST`, `PUT`, `PATCH`, `DELETE` |
| `path` | Path com params (`:userId`) substituídos pelo payload |
| `nullable` | Se `true`, retorna `null` em 404 em vez de lançar `NotFoundError` |
| `auth` | Se `false`, não injeta token de autenticação |

### `call(alias, opts?)`

```ts
// GET/DELETE — payload vira query string
await call('listUsers', { payload: { role: 'admin' } });
// → GET http://atreides:3002/users?role=admin

// POST/PUT/PATCH — payload vira body JSON
await call('createUser', { payload: { name: 'Ana', email: 'a@b.com' } });
// → POST http://atreides:3002/users  body: { name, email }

// Path param substituído automaticamente
await call('getUser', { payload: { userId: '123', role: 'admin' } });
// → GET http://atreides:3002/users/123?role=admin
```

### Autenticação automática

| Contexto | Header injetado |
|----------|----------------|
| Dentro de um request HTTP (token no `tokenStorage`) | `Authorization: Bearer <token>` |
| Kafka/messaging (sem request context) | `X-Service-Token: <JWT_SECRET>` |
| `auth: false` no alias | Nenhum header |

### Mapeamento de erros

| Status HTTP | Exceção lançada |
|-------------|----------------|
| 401 | `UnauthorizedError` |
| 404 (não nullable) | `NotFoundError` |
| 404 (nullable) | `null` |
| 409 | `ConflictError` |
| 422 | `UnprocessableError` |
| 400 | `ValidationError` |

### `tokenStorage`

`AsyncLocalStorage<string>` exportado por `@enxoval/http`. Armazena o Bearer token da request atual. Alimentado automaticamente pelo `setupAuth` do `@enxoval/auth`; lido por `call()` para propagar auth a downstream.

```ts
import { tokenStorage } from '@enxoval/http';

// Em testes ou Kafka: injetar token manualmente
await tokenStorage.run('meu-jwt', async () => {
  await call('getUser', { payload: { userId: '1' } });
});
```

---

## Logs automáticos por request

Cada request gera dois logs via `@enxoval/observability`:

```json
{ "level": "info", "service": "odyssey", "cid": "abc:0", "method": "POST", "url": "/journeys", "msg": "http-server: request received" }
{ "level": "info", "service": "odyssey", "cid": "abc:0", "method": "POST", "url": "/journeys", "status": 201, "durationMs": 42, "msg": "http-server: response sent" }
```

O `cid` (correlation ID) é gerado via `@enxoval/observability` e propagado para downstream via header `x-cid`.

# `@enxoval/http`

> **TL;DR** — Wrapper Fastify para registrar rotas HTTP, SSE e injetar requests em testes. Inclui logging automático de requests/responses, mapeamento de `AppError` → status HTTP e suporte a CORS.

---

## Versão atual: 1.0.26

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

## Logs automáticos por request

Cada request gera dois logs via `@enxoval/observability`:

```json
{ "level": "info", "service": "odyssey", "cid": "abc:0", "method": "POST", "url": "/journeys", "msg": "http-server: request received" }
{ "level": "info", "service": "odyssey", "cid": "abc:0", "method": "POST", "url": "/journeys", "status": 201, "durationMs": 42, "msg": "http-server: response sent" }
```

O `cid` (correlation ID) é gerado via `@enxoval/observability` e propagado para downstream via header `x-cid`.

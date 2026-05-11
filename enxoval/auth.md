# `@enxoval/auth`

> **TL;DR** — Middleware JWT Bearer para Fastify (`setupAuth`), geração de tokens (`signToken`), e acesso ao usuário atual via `AsyncLocalStorage` (`getCurrentUser`). Usa `@enxoval/http`'s `addPreHandler` internamente.

---

## Versão atual: 1.0.3

---

## `setupAuth`

Instala o middleware JWT no Fastify. Deve ser chamado antes de registrar qualquer rota protegida.

```ts
import { setupAuth } from '@enxoval/auth';

// Antes de registrar rotas
setupAuth({
  exclude: ['/auth/login', '/auth/register', '/health'],
});
```

- Rotas em `exclude` não exigem token — `getCurrentUser()` retorna `null` nelas
- Para todas as outras rotas: exige header `Authorization: Bearer <token>`
- Se o token estiver ausente ou inválido: lança `UnauthorizedError` → status 401

### Variáveis de ambiente

| Variável | Descrição |
|----------|-----------|
| `JWT_SECRET` | Chave secreta para assinar/verificar tokens |
| `JWT_EXPIRES_IN` | Expiração (padrão: `1h`). Formato: `30s`, `15m`, `2h`, `7d` |

---

## `signToken`

Gera um JWT Bearer assinado com `JWT_SECRET`.

```ts
import { signToken } from '@enxoval/auth';

const token = signToken(user.id, user.role);
// → 'eyJhbGciOiJIUzI1NiJ9...'
```

Payload do token: `{ userId: string, role: string }` + campos padrão JWT (`iat`, `exp`).

---

## `decodeToken`

Decodifica um token sem verificar a assinatura. Útil para inspeção.

```ts
import { decodeToken } from '@enxoval/auth';

const payload = decodeToken(token);
// → { userId: 'uuid', role: 'student' } | null
```

Retorna `null` se o token for inválido ou não tiver os campos esperados.

---

## `getCurrentUser`

Retorna o usuário do token JWT para a request atual, via `AsyncLocalStorage`.

```ts
import { getCurrentUser } from '@enxoval/auth';
import type { AuthUser } from '@enxoval/auth';

const user = getCurrentUser();
// AuthUser | null

if (!user) throw new UnauthorizedError('Unauthorized');
```

```ts
type AuthUser = {
  userId: string;
  role: string;
  token: string;  // token Bearer completo (útil para repassar a downstream)
};
```

### Uso em SSE

Para rotas SSE, o `getCurrentUser()` funciona normalmente dentro do handler registrado com `sseRoute()` do `@enxoval/http`, pois o contexto AsyncLocalStorage é preservado.

```ts
sseRoute('/journeys/:journeyId/stream', async (params, query, send, signal) => {
  const user = getCurrentUser();
  if (!user) throw new UnauthorizedError('Unauthorized');
  // ...
});
```

---

## X-Service-Token (autenticação serviço-a-serviço)

Quando um serviço precisa chamar outro fora de um contexto HTTP (ex: consumer Kafka), não há Bearer token disponível no `tokenStorage`. Nesses casos, o `@enxoval/http` injeta o header `X-Service-Token: <JWT_SECRET>`.

O `setupAuth` reconhece esse header e autentica a chamada **sem exigir JWT**:

```
X-Service-Token: <valor de JWT_SECRET>
  └─► setupAuth verifica se o valor bate com process.env.JWT_SECRET
  └─► se bater: request passa como autenticada (getCurrentUser() retorna null)
  └─► se não bater: 401 Unauthorized
```

Isso garante que apenas serviços que conhecem o `JWT_SECRET` (configurado via secret compartilhado no Docker Compose / K8s) podem se comunicar sem um token de usuário.

---

## `tokenStorage` (propagação automática de token)

O `setupAuth` alimenta o `tokenStorage` do `@enxoval/http` com o Bearer token de cada request:

```ts
// Internamente, após verify() com sucesso:
tokenStorage.enterWith(token);
```

Isso permite que `call()` do `@enxoval/http` propague o token automaticamente para todas as chamadas downstream feitas durante a mesma request, sem precisar passar o token manualmente.

---

## Fluxo completo

```
1. janus emite POST /auth/login (rota excluída do setupAuth)
2. login(body) → verifica credenciais → signToken(userId, role)
3. Resposta: { token }

4. Cliente inclui: Authorization: Bearer <token>
5. setupAuth preHandler: verify(token, JWT_SECRET)
6. store.enterWith({ userId, role, token }) via AsyncLocalStorage
7. tokenStorage.enterWith(token) — disponível para @enxoval/http call()
8. Qualquer função chamada durante a request pode: getCurrentUser()

Fluxo Kafka (sem request context):
9. @enxoval/http call() detecta tokenStorage vazio
10. Injeta X-Service-Token: JWT_SECRET no header
11. Serviço destino: setupAuth valida X-Service-Token
```

# Autenticação

> **TL;DR** — Login: `imperium → janus → atreides`. Retorna JWT com `{ userId, role }`. Todos os serviços validam o token por conta própria via `@enxoval/auth`. O `imperium` é o único que aceita o token do browser.

---

## Fluxo de Login

```
POST /auth/login { email, password }
  └─► imperium
        └─► janus: verificar credenciais
              └─► atreides: buscar usuário por email, validar hash
              ← user { id, role }
        janus assina JWT { userId, role }
        ← { token }
```

O `atreides` valida o hash da senha. O `janus` assina o token. O `imperium` só encaminha.

---

## Payload do JWT

```ts
{ userId: string, role: "student" | "admin" }
```

O token não carrega nome, email nem outros dados — apenas identidade e papel.

---

## Validação nos serviços

Cada serviço chama `setupAuth()` de `@enxoval/auth` no startup. Isso registra um hook do Fastify que extrai o Bearer token e injeta `request.user` em todas as rotas protegidas.

Rotas públicas (ex: `/health`, SSE stream) são excluídas explicitamente no `app.ts` de cada serviço.

---

## Autenticação serviço-a-serviço (Kafka flows)

Quando um consumer Kafka precisa chamar outro serviço, não há Bearer token disponível (não existe request HTTP ativa). Nesses casos, o `@enxoval/http` injeta automaticamente o header `X-Service-Token: <JWT_SECRET>`.

O `setupAuth` de cada serviço reconhece esse header e autentica a chamada sem JWT de usuário — desde que o valor bata com o `JWT_SECRET` compartilhado no ambiente.

```
consumer Kafka → call('getUser', ...) → X-Service-Token: <JWT_SECRET>
  └─► atreides/setupAuth: valida X-Service-Token
  └─► request autenticada como chamada interna
```

Isso é transparente: o código do consumer não precisa fazer nada diferente. O `@enxoval/http` detecta que o `tokenStorage` está vazio e usa o `X-Service-Token` automaticamente.

---

## Caso especial: SSE

O `EventSource` do browser não suporta header `Authorization`. A rota de stream aceita o token via query param:

```
GET /journeys/:journeyId/stream?token=<jwt>
```

O `imperium` faz o `jwt.verify()` manualmente antes de abrir o proxy SSE.

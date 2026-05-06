# dune-lab

> Plataforma de aprendizado estudantil — microsserviços, Kafka, SSE e observabilidade.

---

## Serviços

| Serviço | Porta | Responsabilidade |
|---------|-------|-----------------|
| [arrakis](https://github.com/dune-lab/arrakis) | 5173 | Frontend SPA (React + Vite) |
| [imperium](https://github.com/dune-lab/imperium) | 3004 | BFF — ponto de entrada único do browser |
| [janus](https://github.com/dune-lab/janus) | 3003 | Auth — emite e assina JWTs |
| [atreides](https://github.com/dune-lab/atreides) | 3002 | Usuários — identidade e credenciais |
| [persona](https://github.com/dune-lab/persona) | 3000 | Estudantes — perfil ligado a um usuário |
| [odyssey](https://github.com/dune-lab/odyssey) | 3001 | Jornadas — máquina de estados + DLQ |
| [platform](https://github.com/dune-lab/platform) | — | Docker Compose + stack de observabilidade |
| [enxoval](https://github.com/dune-lab/enxoval) | — | Bibliotecas compartilhadas (`@enxoval/*`) |

---

## Documentação

| Arquivo | O que responde |
|---------|---------------|
| [architecture.md](./architecture.md) | Como os serviços se conectam, portas, camadas internas, Diplomat Pattern |
| [auth.md](./auth.md) | Como o login funciona, JWT, quem valida o token |
| [saga.md](./saga.md) | O fluxo Kafka interno do odyssey — 9 passos, idempotência, envelope de evento |
| [sse.md](./sse.md) | Atualizações em tempo real, EventEmitter in-process, jornadas travadas |
| [dlq.md](./dlq.md) | Dead Letter Queue (Harkonnen) — mensagens que falharam 3×, reprocess, dismiss |
| [observability.md](./observability.md) | Grafana, Loki, Pino, dashboards, alertas, LogQL |
| [running.md](./running.md) | Como subir tudo localmente com Docker Compose |
| [enxoval/README.md](./enxoval/README.md) | Visão geral dos pacotes `@enxoval/*` e como publicar |
| [enxoval/types.md](./enxoval/types.md) | `createSchema`, `field.*`, `asyncFn`/`fn`, UUID, `AppError` |
| [enxoval/http.md](./enxoval/http.md) | Rotas HTTP, SSE, `inject`, error handler |
| [enxoval/db.md](./enxoval/db.md) | `defineEntity`, `column.*`, data sources, migrations |
| [enxoval/messaging.md](./enxoval/messaging.md) | Kafka `publish`/`subscribe`, retry automático, DLQ |
| [enxoval/auth.md](./enxoval/auth.md) | `setupAuth`, `signToken`, `getCurrentUser` |
| [enxoval/observability.md](./enxoval/observability.md) | `logger`, `newCid`/`nextCid`, correlation IDs |
| [enxoval/testing.md](./enxoval/testing.md) | `generate()`, `itCases()`, `test.mock`, SQLite in-memory |
| [enxoval/quality.md](./enxoval/quality.md) | ESLint `base()`, `boundaries()`, Prettier |

---

## Em 30 segundos

1. Browser fala **só** com `imperium` (BFF).
2. Login: `imperium → janus → atreides`. Retorna JWT.
3. Iniciar jornada: `imperium → odyssey POST /journeys`.
4. odyssey dispara uma **Kafka saga interna** com 9 passos — cada consumer avança o estado e publica o próximo evento.
5. A jornada vai de `JOURNEY_INITIATED` até `JOURNEY_COMPLETED` autonomamente.
6. O browser recebe cada transição via **SSE** em tempo real.
7. Se um consumer falhar 3×, a mensagem vai para o **DLQ** (Harkonnen).

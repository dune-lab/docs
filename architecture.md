# Arquitetura

> **TL;DR** — O browser só fala com o `imperium` (BFF). Cada serviço tem seu próprio banco. Comunicação síncrona via HTTP, assíncrona via Kafka. Todo código de domínio segue o Diplomat Pattern (4 camadas).

---

## Fluxo HTTP

```
Browser (arrakis :5173)
  │  todos os requests
  ▼
imperium (BFF :3004)
  ├─► janus  :3003  — auth
  ├─► atreides :3002 — users
  ├─► persona :3000  — students
  └─► odyssey :3001  — journeys + SSE
```

O browser nunca chama janus, atreides, persona ou odyssey diretamente.

---

## Princípios

- **Banco isolado por serviço** — nenhum serviço acessa o banco do outro.
- **HTTP para comandos** — síncrono, tipado, validado por schema.
- **Kafka para eventos** — assíncrono, durável, at-least-once.
- **JWT em todo lugar** — cada serviço valida o token independentemente via `@enxoval/auth`.
- **Diplomat Pattern** — separação clara de camadas dentro de cada serviço.

---

## Diplomat Pattern (camadas internas)

Todos os serviços seguem a mesma estrutura:

```
src/
├── diplomat/
│   ├── http-server/    ← parse da request HTTP → chama controller
│   ├── http-client/    ← chamadas tipadas para outros serviços
│   └── consumer/       ← consumers Kafka (odyssey, atreides)
├── controllers/        ← orquestra a lógica de domínio
├── adapters/           ← transforma entre wire e model
├── model/              ← schemas de domínio (createSchema)
├── wire/
│   ├── in/             ← schemas de entrada (validação de request)
│   └── out/            ← schemas de saída (shape da response)
└── db/                 ← entidades TypeORM, queries, migrations
```

**Regra de direção:** dados fluem de fora para dentro (`wire-in → model`) e de dentro para fora (`model → wire-out`). Controllers nunca tocam no wire diretamente — usam adapters.

---

## Dois tipos de comunicação assíncrona

| Forma | Usado para | Garantia |
|-------|-----------|---------|
| Kafka (saga odyssey) | Avançar estado da jornada step a step | At-least-once |
| Kafka (atreides) | Broadcast de eventos de domínio | At-least-once |
| EventEmitter in-process | Push SSE para clientes conectados no mesmo processo | In-memory, síncrono |

Ver [saga.md](./saga.md) e [sse.md](./sse.md) para detalhes.

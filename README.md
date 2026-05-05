# dune-lab

> Adaptive learning platform built on microservices, event-driven architecture, and real-time observability.

---

## What is dune-lab?

dune-lab is a platform for managing and tracking student learning journeys. It is designed around a Kafka-driven saga pattern — once a journey starts, it progresses autonomously through a sequence of steps, each represented by a domain event.

The platform is split into small, independent services, each with its own database and a single responsibility. They communicate via HTTP for synchronous commands and Kafka for asynchronous events.

---

## Services

| Service | Port | Role |
|---------|------|------|
| [arrakis](https://github.com/dune-lab/arrakis) | 5173 | Frontend SPA — React + Vite |
| [imperium](https://github.com/dune-lab/imperium) | 3004 | BFF — single entry point for the browser |
| [janus](https://github.com/dune-lab/janus) | 3003 | Auth — issues and signs JWT tokens |
| [atreides](https://github.com/dune-lab/atreides) | 3002 | Users — identity, credentials, email verification |
| [persona](https://github.com/dune-lab/persona) | 3000 | Students — profiles linked to user accounts |
| [odyssey](https://github.com/dune-lab/odyssey) | 3001 | Journeys — saga state machine + DLQ |
| [platform](https://github.com/dune-lab/platform) | — | Docker Compose + observability stack |
| [enxoval](https://github.com/dune-lab/enxoval) | — | Shared libraries (`@enxoval/*`) |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Browser (arrakis)                             │
│                      port 5173                                   │
└──────────────────────────┬──────────────────────────────────────┘
                           │  all requests
                           ▼
            ┌──────────────────────────┐
            │     imperium (BFF)       │
            │       port 3004          │
            └────┬──────┬──────┬──────┘
                 │      │      │
         ┌───────┘      │      └────────────────┐
         ▼              ▼                        ▼
   ┌──────────┐   ┌──────────┐           ┌────────────────┐
   │  janus   │   │ atreides │           │    odyssey     │
   │  :3003   │   │  :3002   │           │    :3001       │
   │ issues   │   │  users   │           │  journeys +    │
   │  JWTs    │   │ Postgres │           │  Kafka saga    │
   └──────────┘   └──────────┘           │  Postgres      │
                                         │  DLQ           │
                                         └────────────────┘
                       ┌──────────┐
                       │ persona  │
                       │  :3000   │
                       │ students │
                       │ Postgres │
                       └──────────┘
```

### Principles

- **No shared databases** — each service owns its data
- **HTTP for commands** — synchronous, typed, contract-validated
- **Kafka for events** — async, durable, at-least-once delivery
- **JWT everywhere** — each service validates tokens independently
- **Diplomat Pattern** — clear layers: http-server → controller → http-client/db

---

## Authentication

```
POST /auth/login { email, password }
  └─► imperium → janus → atreides
        atreides validates password hash
        janus signs JWT { userId, role }
        ← { token }

All subsequent requests:
  Authorization: Bearer <token>
  Each service validates independently via @enxoval/auth
```

Token payload: `{ userId: string, role: "student" | "admin" }`

---

## Event-Driven Architecture

### Odyssey Saga

The learning journey is a Kafka saga internal to odyssey. Each step consumes one event and produces the next:

```
journeyInitiated
  → diagnosticTriggered
  → diagnosticCompleted
  → analysisStarted
  → analysisFinished
  → curriculumGenerated
  → contentDispatched
  → studentEngagementReceived
  → progressMilestoneReached
  → (journey complete)
```

**At-least-once delivery**: Kafka guarantees each message is processed at least once. Each step is designed to be idempotent.

### Atreides Events

| Event | Trigger | Available to |
|-------|---------|--------------|
| `userCreated` | User registers | Future consumers |
| `mailConfirmed` | Email verified | Future consumers |

---

## Resilience — Harkonnen DLQ

When a saga consumer fails 3 times, the message is sent to `student-journey-dlq`. A dedicated consumer persists it in the `harkonnen_messages` table.

```
                 ┌─────────────────────────────────────┐
                 │            Odyssey                   │
                 │                                      │
  Kafka          │  saga consumer                       │
  topic ────────►│    fails × 3                        │
                 │    └─► student-journey-dlq ─────────►│
                 │              harkonnen consumer       │
                 │              └─► harkonnen_messages   │
                 │                   (status: pending)   │
                 │                                      │
                 └─────────────────────────────────────┘
                         │ operator action
                         ▼
                 POST /harkonnen/reprocess
                   └─► republish to original topic
                   └─► saga retries
```

**DLQ statuses**: `pending` → `reprocessed` | `dismissed`

Operators access the DLQ via:
- **UI**: arrakis `/admin/dlq`
- **API**: `GET/POST /harkonnen` (odyssey or imperium)

---

## Real-Time Updates (SSE)

Journey progress is streamed to the browser via Server-Sent Events:

```
GET /journeys/:journeyId/stream
  ← event: { currentStep, status, events[] }
  ← event: { currentStep, status, events[] }
  ...
```

The connection is long-lived. The browser receives a push each time a saga step completes.

---

## Observability

### Structured Logging

Every service emits Pino JSON logs with a `service` field:

```json
{
  "level": "info",
  "time": "2026-05-05T19:47:06.276Z",
  "service": "odyssey",
  "cid": "abc:0",
  "method": "POST",
  "url": "/journeys",
  "status": 201,
  "durationMs": 42,
  "msg": "http-server: response sent"
}
```

| Field | Description |
|-------|-------------|
| `service` | Service name — Loki index label |
| `cid` | Correlation ID — traces a request across logs |
| `level` | `info`, `warn`, `error` |
| `durationMs` | Response time in milliseconds (HTTP responses) |
| `msg` | Human-readable event description |

### Grafana Dashboards (localhost:4001)

| Dashboard | Description |
|-----------|-------------|
| Platform Status | ONLINE/OFFLINE per service |
| Services | Request rate, error rate, p50 latency per service |
| HTTP Requests | All traffic across services |
| Kafka Messages | Producer/consumer activity |
| DLQ Monitor | Failed message ingestion + alert |

### Alerts

| Alert | Condition |
|-------|-----------|
| Consumer Crash | kafkajs `[Consumer] Crash` in logs |
| DLQ Spike | DLQ message rate > threshold |
| 5xx Spike | HTTP 5xx rate > threshold |
| Service Down | No logs from service for 3+ minutes |

### LogQL Queries

**All HTTP responses:**
```logql
{service=~"odyssey|imperium|atreides|persona|janus"}
  | json | msg="http-server: response sent"
```

**Slow requests (> 500ms):**
```logql
{service=~"odyssey|imperium"} | json | durationMs > 500
```

**DLQ ingestion:**
```logql
{service="odyssey"} | json | msg="harkonnen: DLQ message received"
```

**Service down:**
```logql
absent_over_time({service="odyssey"}[3m])
```

---

## Shared Libraries (`@enxoval/*`)

All services are built on a common set of internal packages:

| Package | Description |
|---------|-------------|
| `@enxoval/auth` | JWT Bearer middleware for Fastify |
| `@enxoval/db` | TypeORM wrapper — `defineEntity`, migrations, data source |
| `@enxoval/http` | Fastify wrapper — `listen`, `get`, `post`, `AppError` hierarchy |
| `@enxoval/messaging` | Kafka wrapper — `subscribe`, `publish`, `publishRaw`, retry + DLQ |
| `@enxoval/observability` | Pino logger — structured JSON, `service` label |
| `@enxoval/types` | Schema validation — `createSchema`, `field.*`, `asyncFn`, `fn` |

### `asyncFn` pattern

All domain-layer exports use `asyncFn` (or `fn`) from `@enxoval/types`. This enforces typed input/output at every boundary:

```ts
const Input = createSchema({ studentId: field.uuid(), token: field.string() });

export const startJourney = asyncFn(Input, Journey, async (input) => {
  const res = await fetch(`${process.env.ODYSSEY_URL}/journeys`, {
    method: 'POST',
    headers: { Authorization: `Bearer ${input.token}` },
    body: JSON.stringify({ studentId: input.studentId }),
  });
  if (!res.ok) throw new Error(`odyssey returned ${res.status}`);
  return Journey.parse(await res.json());
});
```

### `AppError` hierarchy

`@enxoval/http` provides typed errors that map to HTTP status codes:

| Error | Status |
|-------|--------|
| `UnauthorizedError` | 401 |
| `NotFoundError` | 404 |
| `ConflictError` | 409 |
| `ValidationError` | 400 |
| `UnprocessableError` | 422 |

---

## Diplomat Pattern

Every service follows the same internal layering:

```
src/
├── diplomat/
│   ├── http-server/    ← parse HTTP request, call controller
│   └── http-client/    ← typed HTTP calls to other services
│   └── consumer/       ← Kafka consumers (odyssey/atreides)
├── controllers/        ← orchestrate domain logic
├── adapters/           ← transform between wire and model formats
├── model/              ← domain schemas (createSchema)
├── wire/               ← I/O schemas (wire-in / wire-out)
└── db/                 ← TypeORM entities, queries, migrations
```

---

## Running the Full Platform

```bash
cd platform
docker compose up
```

**Rebuild after code changes:**

```bash
docker compose build <service>
docker compose up -d <service>
```

**Check logs:**

```bash
docker compose logs <service> -f
```

---

## Repository Index

| Repo | Description |
|------|-------------|
| [dune-lab/docs](https://github.com/dune-lab/docs) | This repo — platform documentation |
| [dune-lab/arrakis](https://github.com/dune-lab/arrakis) | Frontend SPA |
| [dune-lab/imperium](https://github.com/dune-lab/imperium) | BFF |
| [dune-lab/janus](https://github.com/dune-lab/janus) | Auth |
| [dune-lab/atreides](https://github.com/dune-lab/atreides) | Users |
| [dune-lab/persona](https://github.com/dune-lab/persona) | Students |
| [dune-lab/odyssey](https://github.com/dune-lab/odyssey) | Journeys + DLQ |
| [dune-lab/platform](https://github.com/dune-lab/platform) | Docker Compose + infra |
| [dune-lab/enxoval](https://github.com/dune-lab/enxoval) | Shared libraries |

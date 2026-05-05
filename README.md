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

### Overview

dune-lab uses two forms of messaging:

| Form | Used for | Guarantee |
|------|----------|-----------|
| **Kafka** (odyssey saga) | Async state advancement — each step triggers the next | At-least-once |
| **Kafka** (atreides) | Domain events broadcast to future consumers | At-least-once |
| **Node.js EventEmitter** (in-process) | Push saga state to SSE clients within the same process | In-memory, sync |

---

### Odyssey Saga — How It Works

The learning journey is a **self-advancing Kafka saga** internal to odyssey. No external service triggers any step — odyssey consumes its own events, advances state, and publishes the next event.

```
POST /journeys
  └─► INSERT journey (status: active, currentStep: JOURNEY_INITIATED)
  └─► INSERT journey_initiated
  └─► publish journeyInitiated { eventId, journeyId }
          │
          ▼
  [consumer: journeyInitiated]
    └─► INSERT diagnostic_triggered
    └─► UPDATE journey.currentStep = DIAGNOSTIC_TRIGGERED
    └─► emit SSE update
    └─► publish diagnosticTriggered { eventId, journeyId }
          │
          ▼
  [consumer: diagnosticTriggered]
    └─► INSERT diagnostic_completed
    └─► UPDATE journey.currentStep = DIAGNOSTIC_COMPLETED
    └─► emit SSE update
    └─► publish diagnosticCompleted { eventId, journeyId }
          │
         ...
          │
          ▼
  [consumer: progressMilestoneReached]
    └─► INSERT journey_completed
    └─► UPDATE journey.currentStep = JOURNEY_COMPLETED
    └─► UPDATE journey.status = completed
    └─► emit SSE update
    └─► (no further publish — saga ends)
```

---

### The Event Envelope

Every Kafka message in the saga carries a minimal payload:

```ts
{ eventId: UUID, journeyId: UUID }
```

`eventId` is the ID of the **previous** step's DB record. The consumer uses it to look up the previous step before inserting the next — guaranteeing the chain is never broken.

---

### Step Pattern — Idempotency by Design

Every saga controller follows the same structure:

```ts
export const journeyStarted = asyncFn(Event, async (event) => {
  // 1. Verify the previous step exists in DB
  const previous = await findById(event.eventId);             // journeyInitiated record
  if (!previous) throw new NotFoundError('...');

  // 2. Check if this step was already processed (idempotency guard)
  const existing = await diagnosticTriggeredDb.findById(event.eventId);
  if (existing) {
    await sideEffect(buildEvent({ journeyId: existing.journeyId, eventId: existing.id }));
    return;  // already done — just re-publish and exit
  }

  // 3. Insert the next step record
  const current = await diagnosticTriggeredDb.insert(buildEventRecord(event));

  // 4. Advance the journey state
  await journeyDb.updateStep(
    buildJourneyStepUpdate({ id: previous.journeyId, currentStep: 'DIAGNOSTIC_TRIGGERED' }),
  );

  // 5. Push update to SSE clients (in-process)
  journeyEventBus.emit(previous.journeyId, {
    id: previous.journeyId,
    currentStep: 'DIAGNOSTIC_TRIGGERED',
    status: 'active',
  });

  // 6. Publish next Kafka event
  await sideEffect(buildEvent({ journeyId: current.journeyId, eventId: current.id }));
});
```

**Why it's idempotent**: step 2 checks if the next DB record already exists. If Kafka delivers the message twice (at-least-once semantics), the second delivery is a no-op — it just re-publishes and returns. No duplicate inserts, no duplicate state transitions.

**`sideEffect` isolation**: the `publish()` call is wrapped in a separate `asyncFn` called `sideEffect`. This makes the outbound Kafka publish visually distinct from the state mutation. If the process crashes between insert and publish, the saga can be recovered via `POST /journeys/republish`.

---

### Saga Steps Reference

| Step | Consumes | Inserts | Next publish | Journey state |
|------|----------|---------|--------------|---------------|
| journeyStarted | `journeyInitiated` | `diagnostic_triggered` | `diagnosticTriggered` | `DIAGNOSTIC_TRIGGERED` |
| diagnosticTriggered | `diagnosticTriggered` | `diagnostic_completed` | `diagnosticCompleted` | `DIAGNOSTIC_COMPLETED` |
| diagnosticCompleted | `diagnosticCompleted` | `analysis_started` | `analysisStarted` | `ANALYSIS_STARTED` |
| analysisStarted | `analysisStarted` | `analysis_finished` | `analysisFinished` | `ANALYSIS_FINISHED` |
| analysisFinished | `analysisFinished` | `curriculum_generated` | `curriculumGenerated` | `CURRICULUM_GENERATED` |
| curriculumGenerated | `curriculumGenerated` | `content_dispatched` | `contentDispatched` | `CONTENT_DISPATCHED` |
| contentDispatched | `contentDispatched` | `student_engagement_received` | `studentEngagementReceived` | `STUDENT_ENGAGEMENT_RECEIVED` |
| studentEngagementReceived | `studentEngagementReceived` | `progress_milestone_reached` | `progressMilestoneReached` | `PROGRESS_MILESTONE_REACHED` |
| progressMilestoneReached | `progressMilestoneReached` | `journey_completed` | — | `JOURNEY_COMPLETED` / `completed` |

---

### Real-Time SSE — In-Process Event Bus

Each saga step calls `journeyEventBus.emit()` **after** writing to the database. This is a Node.js `EventEmitter` used as an in-process pub/sub:

```ts
// wire/journey-event-bus.ts
const bus = new EventEmitter();

export function emit(journeyId: string, update: JourneyUpdate): void {
  bus.emit(journeyId, update);
}

export function on(journeyId: string, cb: (update: JourneyUpdate) => void): void {
  bus.on(journeyId, cb);
}
```

The SSE route subscribes to this bus per `journeyId`:

```ts
// diplomat/http-server/journey-stream.ts
sseRoute('/journeys/:journeyId/stream', async ({ journeyId }, _query, send, signal) => {
  const listener = (update) => send(update);
  journeyEventBus.on(journeyId, listener);

  await new Promise<void>((resolve) => {
    signal.addEventListener('abort', () => {
      journeyEventBus.off(journeyId, listener);
      resolve();
    });
  });
});
```

When a saga step completes, the browser receives the new state immediately — no polling required.

> **Limitation**: the in-process bus only works when the browser and the saga consumer run in the same process instance. In a horizontally-scaled deployment, a Redis pub/sub or Kafka-backed SSE gateway would be needed.

---

### Stuck Journey Recovery — `POST /journeys/republish`

If odyssey crashes between a DB insert and the Kafka publish, the journey becomes stuck — state is in the DB but no Kafka message drives it forward.

`republishStuckJourneys` resolves this:

```
1. Load all active journeys
2. For each journey still at JOURNEY_INITIATED:
   - If journeyInitiated record exists → re-publish { eventId, journeyId }
   - If journeyInitiated record does NOT exist → create it, then publish
3. Return { republished: N }
```

This is safe to call at any time — the idempotency guard in each consumer ensures no duplicate state transitions even if the journey was not actually stuck.

---

### Kafka Consumer Groups

Each saga topic has its own consumer group, named `student-journey-<eventName>`. This means:

- Each step is independent — a slow consumer on `diagnosticCompleted` doesn't block `journeyInitiated`
- Each group has its own committed offset — safe to restart individual consumers

---

### Atreides Events

| Event (topic) | Trigger | Payload |
|---------------|---------|---------|
| `userCreated` | `POST /users` | `{ userId, email, role }` |
| `mailConfirmed` | `POST /users/confirm-email` | `{ userId, email }` |

These events are published to Kafka and available for any future consumer. No current service subscribes to them.

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

All services are built on a common set of internal packages, maintained in [dune-lab/enxoval](https://github.com/dune-lab/enxoval) and published to npm under the `@enxoval` scope.

| Package | Version | Description |
|---------|---------|-------------|
| `@enxoval/types` | 1.0.24 | Schema validation — `createSchema`, `field.*`, `asyncFn`, `fn`, branded UUID |
| `@enxoval/http` | 1.0.26 | Fastify wrapper — `listen`, `get`, `post`, `AppError` hierarchy |
| `@enxoval/db` | 1.0.3 | TypeORM wrapper — `defineEntity`, migrations, data source |
| `@enxoval/messaging` | 1.0.2 | Kafka wrapper — `subscribe`, `publish`, `publishRaw`, retry + DLQ |
| `@enxoval/auth` | 1.0.2 | JWT Bearer middleware for Fastify |
| `@enxoval/observability` | 1.0.2 | Pino logger — structured JSON, `service` label |

### Bump Flow — How `@enxoval/*` updates reach services

Versioning is tag-driven. Every push of a `v*` tag to `dune-lab/enxoval` triggers a three-job pipeline:

```
git tag v1.0.25
git push origin v1.0.25
        │
        ▼
[job: publish]
  Build all packages
  Publish to npm (skips already-published versions)
        │
        ▼
[job: discover]
  Scan all dune-lab/* repos for @enxoval/* in package.json
  Output: ["odyssey", "imperium", "atreides", "persona", "janus"]
        │
        ▼
[job: bump] (matrix — one job per repo, runs in parallel)
  For each repo:
    ├── npm install @enxoval/types@1.0.25 @enxoval/http@1.0.26 ...
    │   (only packages that already appear as a dependency)
    ├── git checkout -b chore/bump-enxoval-v1.0.25
    ├── git commit package.json package-lock.json
    └── gh pr create → "chore: bump @enxoval/* to v1.0.25"
```

**Result**: every service repo gets a ready-to-merge PR within minutes of the tag push.

```
dune-lab/odyssey  ← PR: chore/bump-enxoval-v1.0.25
dune-lab/imperium ← PR: chore/bump-enxoval-v1.0.25
dune-lab/atreides ← PR: chore/bump-enxoval-v1.0.25
dune-lab/persona  ← PR: chore/bump-enxoval-v1.0.25
dune-lab/janus    ← PR: chore/bump-enxoval-v1.0.25
```

**Key behaviors:**
- Only repos with `@enxoval/*` in `dependencies` or `devDependencies` are targeted — the discover job reads `package.json` via GitHub API before running any bump
- Only packages already listed in the repo's dependencies are bumped — a repo that doesn't use `@enxoval/messaging` won't have it added
- `package-lock.json` is always updated alongside `package.json` — `npm ci` in CI requires them to be in sync
- The workflow uses `DUNE_LAB_TOKEN` (a PAT with repo scope) to push branches and open PRs across repos

**How to publish a new version:**

```bash
# 1. Edit package.json version in the package(s) that changed
cd enxoval/types
# bump version field to 1.0.25

# 2. Commit
git add types/package.json
git commit -m "feat(types): add field.nullable helper"

# 3. Tag and push — this triggers the full pipeline
git tag v1.0.25
git push origin main
git push origin v1.0.25
```

The tag version does not need to match any individual package version — it's used only as a label for the bump PRs. npm publish uses each `package.json`'s own `version` field.

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

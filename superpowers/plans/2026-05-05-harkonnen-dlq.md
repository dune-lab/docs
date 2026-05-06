# Harkonnen DLQ Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a full DLQ admin system (harkonnen) that persists failed Kafka messages to Postgres and exposes an admin UI to reprocess per message, reprocess all by topic, edit payload before reprocessing, and dismiss messages.

**Architecture:** odyssey consumes `student-journey-dlq` directly via `kafka.consumer()` (bypassing `subscribe()` to avoid DLQ-of-DLQ loops), stores messages in `harkonnen_messages` table (JSONB payload), and exposes four REST endpoints. imperium proxies them with auth. arrakis renders `/admin/dlq`.

**Tech Stack:** TypeScript, asyncFn/fn (@enxoval/types), kafkajs (via @enxoval/messaging kafka instance), TypeORM (@enxoval/db), React + Tailwind (arrakis)

---

## File Map

**odyssey — create:**
- `src/db/migrations/1777200120000-create-harkonnen-messages-table.ts`
- `src/db/wire/harkonnen-message.ts`
- `src/model/harkonnen-message.ts`
- `src/adapters/harkonnen-message.ts`
- `src/db/harkonnen-message.ts`
- `src/diplomat/consumer/harkonnen.ts`
- `src/wire/in/harkonnen.ts`
- `src/wire/out/harkonnen.ts`
- `src/controllers/harkonnen.ts`
- `src/diplomat/http-server/harkonnen.ts`

**odyssey — modify:**
- `src/db/data-source.ts` — add `HarkonnenMessageSchema` to entities
- `src/server.ts` — call `setupHarkonnenConsumer()`
- `src/diplomat/http-server/index.ts` — call `registerHarkonnenRoutes()`

**imperium — create:**
- `src/model/dlq-message.ts`
- `src/wire/in/harkonnen.ts`
- `src/wire/out/harkonnen.ts`
- `src/controllers/harkonnen.ts`
- `src/diplomat/http-server/harkonnen.ts`

**imperium — modify:**
- `src/diplomat/http-client/odyssey.ts` — add `listDlq`, `reprocessDlqOne`, `reprocessDlqAllByTopic`, `dismissDlq`
- `src/app.ts` — register harkonnen routes

**arrakis — create:**
- `src/pages/admin/Dlq.tsx`

**arrakis — modify:**
- `src/api/imperium.ts` — add `DlqMessage` type + four API functions
- `src/router.tsx` — add `/admin/dlq` route
- `src/components/AppLayout.tsx` — add DLQ nav item

---

## Task 1: odyssey — DB migration

**Files:**
- Create: `odyssey/src/db/migrations/1777200120000-create-harkonnen-messages-table.ts`

- [ ] **Step 1: Write the migration file**

```ts
import { Migration, sql, MigrationRunner } from '@enxoval/db';

export class CreateHarkonnenMessagesTable1777200120000 extends Migration {
  name = 'CreateHarkonnenMessagesTable1777200120000';

  async up(runner: MigrationRunner): Promise<void> {
    await sql(
      runner,
      `CREATE TABLE "harkonnen_messages" (
        "id"              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
        "original_topic"  VARCHAR     NOT NULL,
        "name"            VARCHAR     NOT NULL,
        "payload"         JSONB       NOT NULL,
        "error"           TEXT        NOT NULL,
        "failed_at"       TIMESTAMPTZ NOT NULL,
        "status"          VARCHAR     NOT NULL DEFAULT 'pending',
        "reprocessed_at"  TIMESTAMPTZ,
        "created_at"      TIMESTAMPTZ NOT NULL DEFAULT now()
      )`,
    );
  }

  async down(runner: MigrationRunner): Promise<void> {
    await sql(runner, `DROP TABLE "harkonnen_messages"`);
  }
}
```

- [ ] **Step 2: Commit**

```bash
cd odyssey
git -c user.name="Joao Vitor" -c user.email="joaorabello99@gmail.com" \
  commit -m "feat(harkonnen): add harkonnen_messages migration"
```

---

## Task 2: odyssey — DbWire entity, model, adapter

**Files:**
- Create: `odyssey/src/db/wire/harkonnen-message.ts`
- Create: `odyssey/src/model/harkonnen-message.ts`
- Create: `odyssey/src/adapters/harkonnen-message.ts`

- [ ] **Step 1: Write DbWire entity**

`odyssey/src/db/wire/harkonnen-message.ts`:
```ts
import { defineEntity, column, SchemaDefinition } from '@enxoval/db';

export class HarkonnenMessageDbWire {
  id!: string;
  original_topic!: string;
  name!: string;
  payload!: object;
  error!: string;
  failed_at!: Date;
  status!: string;
  reprocessed_at!: Date | null;
  created_at!: Date;

  static parse(data: unknown): HarkonnenMessageDbWire {
    return Object.assign(new HarkonnenMessageDbWire(), data);
  }
}

export const HarkonnenMessageSchema: SchemaDefinition<HarkonnenMessageDbWire> = defineEntity(
  HarkonnenMessageDbWire,
  {
    tableName: 'harkonnen_messages',
    columns: {
      id:             column.primaryUuid(),
      original_topic: column.varchar(),
      name:           column.varchar(),
      payload:        { type: 'jsonb' },
      error:          { type: 'text' },
      failed_at:      { type: 'timestamptz' },
      status:         column.varchar(),
      reprocessed_at: { type: 'timestamptz', nullable: true },
      created_at:     column.createdAt(),
    },
  },
);
```

- [ ] **Step 2: Write model**

`odyssey/src/model/harkonnen-message.ts`:
```ts
import { createSchema, field } from '@enxoval/types';

export const HarkonnenMessage = createSchema({
  id:             field.uuid(),
  originalTopic:  field.string(),
  name:           field.string(),
  payload:        field.string(),
  error:          field.string(),
  failedAt:       field.string(),
  status:         field.string(),
  reprocessedAt:  field.nullable(field.string()),
  createdAt:      field.string(),
});

export const HarkonnenMessageInput = createSchema({
  originalTopic: field.string(),
  name:          field.string(),
  payload:       field.string(),
  error:         field.string(),
  failedAt:      field.string(),
});
```

- [ ] **Step 3: Write adapter**

`odyssey/src/adapters/harkonnen-message.ts`:
```ts
import { fn, asUUID } from '@enxoval/types';
import { HarkonnenMessage, HarkonnenMessageInput } from '../model/harkonnen-message';
import { HarkonnenMessageDbWire } from '../db/wire/harkonnen-message';

export const fromDbWire = fn(HarkonnenMessageDbWire, HarkonnenMessage, (wire) => ({
  id:             asUUID(wire.id),
  originalTopic:  wire.original_topic,
  name:           wire.name,
  payload:        JSON.stringify(wire.payload),
  error:          wire.error,
  failedAt:       wire.failed_at.toISOString(),
  status:         wire.status,
  reprocessedAt:  wire.reprocessed_at ? wire.reprocessed_at.toISOString() : null,
  createdAt:      wire.created_at.toISOString(),
}));

export const toDbWire = fn(HarkonnenMessageInput, HarkonnenMessageDbWire, (input) => {
  const row = new HarkonnenMessageDbWire();
  row.original_topic = input.originalTopic;
  row.name           = input.name;
  row.payload        = JSON.parse(input.payload) as object;
  row.error          = input.error;
  row.failed_at      = new Date(input.failedAt);
  row.status         = 'pending';
  return row;
});
```

- [ ] **Step 4: Commit**

```bash
git -c user.name="Joao Vitor" -c user.email="joaorabello99@gmail.com" \
  commit -m "feat(harkonnen): add DbWire entity, model and adapter"
```

---

## Task 3: odyssey — DB module

**Files:**
- Create: `odyssey/src/db/harkonnen-message.ts`

- [ ] **Step 1: Write DB module**

`odyssey/src/db/harkonnen-message.ts`:
```ts
import { asyncFn, type UUID } from '@enxoval/types';
import { AppDataSource } from './data-source';
import { HarkonnenMessageDbWire } from './wire/harkonnen-message';
import { HarkonnenMessage, HarkonnenMessageInput } from '../model/harkonnen-message';
import { fromDbWire, toDbWire } from '../adapters/harkonnen-message';

const repo = () => AppDataSource.getRepository(HarkonnenMessageDbWire);

export const insert = asyncFn(HarkonnenMessageInput, HarkonnenMessage, async (input) => {
  const row = await repo().save(toDbWire(input));
  return fromDbWire(row);
});

export async function findAll(): Promise<ReturnType<typeof HarkonnenMessage.parse>[]> {
  const rows = await repo().find({ order: { created_at: 'DESC' } });
  return rows.map(fromDbWire);
}

export async function findById(id: UUID): Promise<ReturnType<typeof HarkonnenMessage.parse> | null> {
  const row = await repo().findOne({ where: { id } });
  return row ? fromDbWire(row) : null;
}

export async function findPendingByTopic(
  topic: string,
): Promise<ReturnType<typeof HarkonnenMessage.parse>[]> {
  const rows = await repo().find({ where: { original_topic: topic, status: 'pending' } });
  return rows.map(fromDbWire);
}

export async function markReprocessed(id: UUID): Promise<void> {
  await repo().update({ id }, { status: 'reprocessed', reprocessed_at: new Date() });
}

export async function markDismissed(id: UUID): Promise<void> {
  await repo().update({ id }, { status: 'dismissed' });
}
```

- [ ] **Step 2: Commit**

```bash
git -c user.name="Joao Vitor" -c user.email="joaorabello99@gmail.com" \
  commit -m "feat(harkonnen): add DB module"
```

---

## Task 4: odyssey — DLQ consumer

**Files:**
- Create: `odyssey/src/diplomat/consumer/harkonnen.ts`

The consumer uses `kafka` directly (not `subscribe()`) to avoid triggering the retry/DLQ logic that `subscribe()` adds. `fromBeginning: true` ensures historical messages are loaded on first start.

- [ ] **Step 1: Write DLQ consumer**

`odyssey/src/diplomat/consumer/harkonnen.ts`:
```ts
import { kafka } from '@enxoval/messaging';
import { logger } from '@enxoval/observability';
import { insert } from '../../db/harkonnen-message';

const DLQ_TOPIC = 'student-journey-dlq';

export async function setupHarkonnenConsumer(): Promise<void> {
  const consumer = kafka.consumer({ groupId: 'odyssey-harkonnen-dlq' });
  await consumer.connect();
  await consumer.subscribe({ topic: DLQ_TOPIC, fromBeginning: true });
  await consumer.run({
    eachMessage: async ({ message }) => {
      const raw = message.value?.toString();
      if (!raw) return;
      const data = JSON.parse(raw) as {
        originalTopic: string;
        name: string;
        payload: unknown;
        error: string;
        failedAt: string;
      };
      logger.info({ topic: DLQ_TOPIC, name: data.name }, 'harkonnen: DLQ message received');
      await insert({
        originalTopic: data.originalTopic,
        name:          data.name,
        payload:       JSON.stringify(data.payload),
        error:         data.error,
        failedAt:      data.failedAt,
      });
    },
  });
}
```

- [ ] **Step 2: Commit**

```bash
git -c user.name="Joao Vitor" -c user.email="joaorabello99@gmail.com" \
  commit -m "feat(harkonnen): add DLQ consumer"
```

---

## Task 5: odyssey — Wire types and controllers

**Files:**
- Create: `odyssey/src/wire/in/harkonnen.ts`
- Create: `odyssey/src/wire/out/harkonnen.ts`
- Create: `odyssey/src/controllers/harkonnen.ts`

- [ ] **Step 1: Write wire in schemas**

`odyssey/src/wire/in/harkonnen.ts`:
```ts
import { createSchema, field } from '@enxoval/types';

export const ReprocessOneWireIn = createSchema({
  id:      field.uuid(),
  payload: field.string(),
});

export const ReprocessAllByTopicWireIn = createSchema({
  topic: field.string(),
});

export const DismissWireIn = createSchema({
  id: field.uuid(),
});

export const NoInput = createSchema({});
```

- [ ] **Step 2: Write wire out schemas**

`odyssey/src/wire/out/harkonnen.ts`:
```ts
import { createSchema, field } from '@enxoval/types';

export const ReprocessOneWireOut = createSchema({
  reprocessed: field.boolean(),
});

export const ReprocessAllWireOut = createSchema({
  reprocessed: field.number(),
});

export const DismissWireOut = createSchema({
  dismissed: field.boolean(),
});
```

- [ ] **Step 3: Write controllers**

`odyssey/src/controllers/harkonnen.ts`:
```ts
import { asyncFn, field, NotFoundError } from '@enxoval/types';
import { publishRaw } from '@enxoval/messaging';
import { NoInput, ReprocessOneWireIn, ReprocessAllByTopicWireIn, DismissWireIn } from '../wire/in/harkonnen';
import { ReprocessOneWireOut, ReprocessAllWireOut, DismissWireOut } from '../wire/out/harkonnen';
import { HarkonnenMessage } from '../model/harkonnen-message';
import * as db from '../db/harkonnen-message';

export const listDlqMessages = asyncFn(NoInput, field.array(HarkonnenMessage), async (_) => {
  return db.findAll();
});

export const reprocessDlqOne = asyncFn(ReprocessOneWireIn, ReprocessOneWireOut, async (input) => {
  const msg = await db.findById(input.id);
  if (!msg) throw new NotFoundError(`DLQ message ${input.id} not found`);
  const parsed = JSON.parse(input.payload) as unknown;
  await publishRaw(msg.originalTopic, parsed);
  await db.markReprocessed(input.id);
  return { reprocessed: true };
});

export const reprocessDlqAllByTopic = asyncFn(ReprocessAllByTopicWireIn, ReprocessAllWireOut, async (input) => {
  const messages = await db.findPendingByTopic(input.topic);
  await Promise.all(
    messages.map(async (msg) => {
      await publishRaw(msg.originalTopic, JSON.parse(msg.payload) as unknown);
      await db.markReprocessed(msg.id);
    }),
  );
  return { reprocessed: messages.length };
});

export const dismissDlqMessage = asyncFn(DismissWireIn, DismissWireOut, async (input) => {
  const msg = await db.findById(input.id);
  if (!msg) throw new NotFoundError(`DLQ message ${input.id} not found`);
  await db.markDismissed(input.id);
  return { dismissed: true };
});
```

- [ ] **Step 4: Commit**

```bash
git -c user.name="Joao Vitor" -c user.email="joaorabello99@gmail.com" \
  commit -m "feat(harkonnen): add wire types and controllers"
```

---

## Task 6: odyssey — HTTP routes

**Files:**
- Create: `odyssey/src/diplomat/http-server/harkonnen.ts`

- [ ] **Step 1: Write route registration**

`odyssey/src/diplomat/http-server/harkonnen.ts`:
```ts
import { get, post } from '@enxoval/http';
import {
  listDlqMessages,
  reprocessDlqOne,
  reprocessDlqAllByTopic,
  dismissDlqMessage,
} from '../../controllers/harkonnen';

export function registerHarkonnenRoutes(): void {
  get('/harkonnen', async () => listDlqMessages({}));
  post('/harkonnen/reprocess',     async (body) => reprocessDlqOne(body));
  post('/harkonnen/reprocess-all', async (body) => reprocessDlqAllByTopic(body));
  post('/harkonnen/dismiss',       async (body) => dismissDlqMessage(body));
}
```

- [ ] **Step 2: Commit**

```bash
git -c user.name="Joao Vitor" -c user.email="joaorabello99@gmail.com" \
  commit -m "feat(harkonnen): add HTTP routes"
```

---

## Task 7: odyssey — Wire everything up

**Files:**
- Modify: `odyssey/src/db/data-source.ts`
- Modify: `odyssey/src/server.ts`
- Modify: `odyssey/src/diplomat/http-server/index.ts`

- [ ] **Step 1: Add entity to data-source**

In `odyssey/src/db/data-source.ts`, add import and entity:

```ts
import { HarkonnenMessageSchema } from './wire/harkonnen-message';
```

Add `HarkonnenMessageSchema` to the `entities` array.

- [ ] **Step 2: Add consumer to server**

In `odyssey/src/server.ts`, add import and call:

```ts
import { setupHarkonnenConsumer } from './diplomat/consumer/harkonnen';
```

After `setupConsumers()` in the `listen` callback:
```ts
setupConsumers();
await setupHarkonnenConsumer();
```

- [ ] **Step 3: Add routes to http-server index**

In `odyssey/src/diplomat/http-server/index.ts`:
```ts
import { registerHarkonnenRoutes } from './harkonnen';
```

Add to `setupRoutes()`:
```ts
registerHarkonnenRoutes();
```

- [ ] **Step 4: Build and check for TypeScript errors**

```bash
cd odyssey && npx tsc --noEmit
```

Expected: no errors.

- [ ] **Step 5: Commit**

```bash
git -c user.name="Joao Vitor" -c user.email="joaorabello99@gmail.com" \
  commit -m "feat(harkonnen): wire odyssey data-source, server and routes"
```

---

## Task 8: imperium — model, wire, http-client, controllers, routes, app

**Files:**
- Create: `imperium/src/model/dlq-message.ts`
- Create: `imperium/src/wire/in/harkonnen.ts`
- Create: `imperium/src/wire/out/harkonnen.ts`
- Create: `imperium/src/controllers/harkonnen.ts`
- Create: `imperium/src/diplomat/http-server/harkonnen.ts`
- Modify: `imperium/src/diplomat/http-client/odyssey.ts`
- Modify: `imperium/src/app.ts`

- [ ] **Step 1: Write DlqMessage model**

`imperium/src/model/dlq-message.ts`:
```ts
import { createSchema, field } from '@enxoval/types';

export const DlqMessage = createSchema({
  id:             field.uuid(),
  originalTopic:  field.string(),
  name:           field.string(),
  payload:        field.string(),
  error:          field.string(),
  failedAt:       field.string(),
  status:         field.string(),
  reprocessedAt:  field.nullable(field.string()),
  createdAt:      field.string(),
});
```

- [ ] **Step 2: Write wire in schemas**

`imperium/src/wire/in/harkonnen.ts`:
```ts
import { createSchema, field } from '@enxoval/types';

export const ReprocessOneWireIn = createSchema({
  id:      field.uuid(),
  payload: field.string(),
});

export const ReprocessAllByTopicWireIn = createSchema({
  topic: field.string(),
});

export const DismissWireIn = createSchema({
  id: field.uuid(),
});
```

- [ ] **Step 3: Write wire out schemas**

`imperium/src/wire/out/harkonnen.ts`:
```ts
import { createSchema, field } from '@enxoval/types';

export const ReprocessOneWireOut = createSchema({
  reprocessed: field.boolean(),
});

export const ReprocessAllWireOut = createSchema({
  reprocessed: field.number(),
});

export const DismissWireOut = createSchema({
  dismissed: field.boolean(),
});
```

- [ ] **Step 4: Add http-client functions to odyssey.ts**

In `imperium/src/diplomat/http-client/odyssey.ts`, add these imports and exports:

```ts
import { DlqMessage } from '../../model/dlq-message';
import { ReprocessOneWireOut, ReprocessAllWireOut, DismissWireOut } from '../../wire/out/harkonnen';
```

Add after the existing exports:

```ts
const ReprocessOneClientInput = createSchema({
  token:   field.string(),
  id:      field.uuid(),
  payload: field.string(),
});

const ReprocessAllClientInput = createSchema({
  token: field.string(),
  topic: field.string(),
});

const DismissClientInput = createSchema({
  token: field.string(),
  id:    field.uuid(),
});

export const listDlq = asyncFn(TokenInput, field.array(DlqMessage), async (input) => {
  const res = await fetch(`${process.env.ODYSSEY_URL}/harkonnen`, {
    headers: { Authorization: `Bearer ${input.token}` },
  });
  if (!res.ok) throw new Error(`odyssey returned ${res.status}`);
  return field.array(DlqMessage).parse(await res.json());
});

export const reprocessDlqOne = asyncFn(ReprocessOneClientInput, ReprocessOneWireOut, async (input) => {
  const res = await fetch(`${process.env.ODYSSEY_URL}/harkonnen/reprocess`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${input.token}` },
    body: JSON.stringify({ id: input.id, payload: input.payload }),
  });
  if (!res.ok) throw new Error(`odyssey returned ${res.status}`);
  return ReprocessOneWireOut.parse(await res.json());
});

export const reprocessDlqAllByTopic = asyncFn(ReprocessAllClientInput, ReprocessAllWireOut, async (input) => {
  const res = await fetch(`${process.env.ODYSSEY_URL}/harkonnen/reprocess-all`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${input.token}` },
    body: JSON.stringify({ topic: input.topic }),
  });
  if (!res.ok) throw new Error(`odyssey returned ${res.status}`);
  return ReprocessAllWireOut.parse(await res.json());
});

export const dismissDlq = asyncFn(DismissClientInput, DismissWireOut, async (input) => {
  const res = await fetch(`${process.env.ODYSSEY_URL}/harkonnen/dismiss`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${input.token}` },
    body: JSON.stringify({ id: input.id }),
  });
  if (!res.ok) throw new Error(`odyssey returned ${res.status}`);
  return DismissWireOut.parse(await res.json());
});
```

- [ ] **Step 5: Write controllers**

`imperium/src/controllers/harkonnen.ts`:
```ts
import { asyncFn, field, UnauthorizedError } from '@enxoval/types';
import { getCurrentUser } from '@enxoval/auth';
import { NoInput } from '../wire/in/no-input';
import { ReprocessOneWireIn, ReprocessAllByTopicWireIn, DismissWireIn } from '../wire/in/harkonnen';
import { ReprocessOneWireOut, ReprocessAllWireOut, DismissWireOut } from '../wire/out/harkonnen';
import { DlqMessage } from '../model/dlq-message';
import {
  listDlq as odysseyList,
  reprocessDlqOne as odysseyReprocessOne,
  reprocessDlqAllByTopic as odysseyReprocessAll,
  dismissDlq as odysseyDismiss,
} from '../diplomat/http-client/odyssey';

export const listDlqMessages = asyncFn(NoInput, field.array(DlqMessage), async (_) => {
  const auth = getCurrentUser();
  if (!auth) throw new UnauthorizedError('Unauthorized');
  return odysseyList({ token: auth.token });
});

export const reprocessDlqOne = asyncFn(ReprocessOneWireIn, ReprocessOneWireOut, async (input) => {
  const auth = getCurrentUser();
  if (!auth) throw new UnauthorizedError('Unauthorized');
  return odysseyReprocessOne({ token: auth.token, id: input.id, payload: input.payload });
});

export const reprocessDlqAllByTopic = asyncFn(ReprocessAllByTopicWireIn, ReprocessAllWireOut, async (input) => {
  const auth = getCurrentUser();
  if (!auth) throw new UnauthorizedError('Unauthorized');
  return odysseyReprocessAll({ token: auth.token, topic: input.topic });
});

export const dismissDlqMessage = asyncFn(DismissWireIn, DismissWireOut, async (input) => {
  const auth = getCurrentUser();
  if (!auth) throw new UnauthorizedError('Unauthorized');
  return odysseyDismiss({ token: auth.token, id: input.id });
});
```

- [ ] **Step 6: Write HTTP routes**

`imperium/src/diplomat/http-server/harkonnen.ts`:
```ts
import { get, post } from '@enxoval/http';
import {
  listDlqMessages,
  reprocessDlqOne,
  reprocessDlqAllByTopic,
  dismissDlqMessage,
} from '../../controllers/harkonnen';

export function registerHarkonnenRoutes(): void {
  get('/harkonnen', async () => listDlqMessages({}));
  post('/harkonnen/reprocess',     async (body) => reprocessDlqOne(body));
  post('/harkonnen/reprocess-all', async (body) => reprocessDlqAllByTopic(body));
  post('/harkonnen/dismiss',       async (body) => dismissDlqMessage(body));
}
```

- [ ] **Step 7: Register routes in app.ts**

In `imperium/src/app.ts`, add import and call:

```ts
import { registerHarkonnenRoutes } from './diplomat/http-server/harkonnen';
```

Add `registerHarkonnenRoutes()` at the end of `buildApp()`.

- [ ] **Step 8: Build and check**

```bash
cd imperium && npx tsc --noEmit
```

Expected: no errors.

- [ ] **Step 9: Commit**

```bash
git -c user.name="Joao Vitor" -c user.email="joaorabello99@gmail.com" \
  commit -m "feat(harkonnen): add imperium DLQ proxy (model, controllers, routes)"
```

---

## Task 9: arrakis — API client

**Files:**
- Modify: `arrakis/src/api/imperium.ts`

- [ ] **Step 1: Add DlqMessage type and API functions**

In `arrakis/src/api/imperium.ts`, add:

```ts
export type DlqMessage = {
  id: string;
  originalTopic: string;
  name: string;
  payload: string;
  error: string;
  failedAt: string;
  status: string;
  reprocessedAt: string | null;
  createdAt: string;
};

export function listDlq(token: string) {
  return request<DlqMessage[]>('/harkonnen', { headers: authHeaders(token) });
}

export function reprocessDlqOne(id: string, payload: string, token: string) {
  return request<{ reprocessed: boolean }>('/harkonnen/reprocess', {
    method: 'POST',
    headers: authHeaders(token),
    body: JSON.stringify({ id, payload }),
  });
}

export function reprocessDlqAllByTopic(topic: string, token: string) {
  return request<{ reprocessed: number }>('/harkonnen/reprocess-all', {
    method: 'POST',
    headers: authHeaders(token),
    body: JSON.stringify({ topic }),
  });
}

export function dismissDlq(id: string, token: string) {
  return request<{ dismissed: boolean }>('/harkonnen/dismiss', {
    method: 'POST',
    headers: authHeaders(token),
    body: JSON.stringify({ id }),
  });
}
```

- [ ] **Step 2: Commit**

```bash
cd arrakis && git -c user.name="Joao Vitor" -c user.email="joaorabello99@gmail.com" \
  commit -m "feat(harkonnen): add DLQ API client functions"
```

---

## Task 10: arrakis — DLQ admin page

**Files:**
- Create: `arrakis/src/pages/admin/Dlq.tsx`
- Modify: `arrakis/src/router.tsx`
- Modify: `arrakis/src/components/AppLayout.tsx`

- [ ] **Step 1: Write Dlq.tsx**

`arrakis/src/pages/admin/Dlq.tsx`:
```tsx
import { useEffect, useState } from 'react';
import { useAuth } from '../../context/AuthContext';
import {
  listDlq,
  reprocessDlqOne,
  reprocessDlqAllByTopic,
  dismissDlq,
  type DlqMessage,
} from '../../api/imperium';
import { Button } from '../../components/ui/Button';

type Filter = 'all' | 'pending' | 'reprocessed' | 'dismissed';

const FILTERS: { key: Filter; label: string }[] = [
  { key: 'all',          label: 'Todas' },
  { key: 'pending',      label: 'Pendentes' },
  { key: 'reprocessed',  label: 'Reprocessadas' },
  { key: 'dismissed',    label: 'Descartadas' },
];

const STATUS_STYLE: Record<string, { background: string; color: string }> = {
  pending:     { background: 'rgba(248,113,113,0.1)', color: '#f87171' },
  reprocessed: { background: 'rgba(61,220,132,0.1)',  color: '#3ddc84' },
  dismissed:   { background: 'rgba(85,85,85,0.1)',    color: '#555' },
};

function formatTime(iso: string): string {
  return new Date(iso).toLocaleString('pt-BR', {
    day: '2-digit', month: '2-digit', hour: '2-digit', minute: '2-digit',
  });
}

function TopicBadge({ topic }: { topic: string }) {
  const short = topic.split('.').pop() ?? topic;
  return (
    <span
      className="text-xs px-2 py-0.5 rounded font-mono"
      style={{ background: '#1a1a1a', color: '#888', border: '1px solid #222' }}
    >
      {short}
    </span>
  );
}

export function Dlq() {
  const { token } = useAuth();
  const [messages, setMessages] = useState<DlqMessage[]>([]);
  const [filter, setFilter] = useState<Filter>('pending');
  const [expanded, setExpanded] = useState<string | null>(null);
  const [editedPayloads, setEditedPayloads] = useState<Record<string, string>>({});
  const [loading, setLoading] = useState<Record<string, boolean>>({});
  const [error, setError] = useState('');

  useEffect(() => {
    if (!token) return;
    listDlq(token).then(setMessages).catch(err => setError(err.message));
  }, [token]);

  const visible = filter === 'all'
    ? messages
    : messages.filter(m => m.status === filter);

  function countFor(f: Filter) {
    if (f === 'all') return messages.length;
    return messages.filter(m => m.status === f).length;
  }

  function getPayload(msg: DlqMessage): string {
    if (editedPayloads[msg.id] !== undefined) return editedPayloads[msg.id];
    try {
      return JSON.stringify(JSON.parse(msg.payload), null, 2);
    } catch {
      return msg.payload;
    }
  }

  async function handleReprocessOne(msg: DlqMessage) {
    if (!token) return;
    setLoading(l => ({ ...l, [msg.id]: true }));
    setError('');
    try {
      const payload = editedPayloads[msg.id] ?? msg.payload;
      await reprocessDlqOne(msg.id, payload, token);
      setMessages(prev => prev.map(m => m.id === msg.id ? { ...m, status: 'reprocessed' } : m));
      setExpanded(null);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Erro ao reprocessar');
    } finally {
      setLoading(l => ({ ...l, [msg.id]: false }));
    }
  }

  async function handleDismiss(msg: DlqMessage) {
    if (!token) return;
    setLoading(l => ({ ...l, [`dismiss-${msg.id}`]: true }));
    setError('');
    try {
      await dismissDlq(msg.id, token);
      setMessages(prev => prev.map(m => m.id === msg.id ? { ...m, status: 'dismissed' } : m));
      setExpanded(null);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Erro ao descartar');
    } finally {
      setLoading(l => ({ ...l, [`dismiss-${msg.id}`]: false }));
    }
  }

  async function handleReprocessAllByTopic(topic: string) {
    if (!token) return;
    setLoading(l => ({ ...l, [`all-${topic}`]: true }));
    setError('');
    try {
      const { reprocessed } = await reprocessDlqAllByTopic(topic, token);
      setMessages(prev =>
        prev.map(m =>
          m.originalTopic === topic && m.status === 'pending'
            ? { ...m, status: 'reprocessed' }
            : m,
        ),
      );
      setError(`${reprocessed} mensagem(ns) reprocessada(s) do tópico ${topic}.`);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Erro ao reprocessar tópico');
    } finally {
      setLoading(l => ({ ...l, [`all-${topic}`]: false }));
    }
  }

  const topics = [...new Set(messages.filter(m => m.status === 'pending').map(m => m.originalTopic))];

  return (
    <div className="p-8">
      <div className="max-w-3xl">
        <div className="flex justify-between items-center mb-6">
          <div>
            <h1 className="text-xl font-semibold text-white">Harkonnen DLQ</h1>
            <p className="text-xs mt-0.5" style={{ color: '#444' }}>Mensagens Kafka que falharam após 3 tentativas</p>
          </div>
          {topics.length > 0 && (
            <div className="flex gap-2 flex-wrap justify-end">
              {topics.map(topic => (
                <Button
                  key={topic}
                  variant="secondary"
                  loading={!!loading[`all-${topic}`]}
                  onClick={() => handleReprocessAllByTopic(topic)}
                >
                  Reprocessar tudo: {topic.split('.').pop()}
                </Button>
              ))}
            </div>
          )}
        </div>

        {error && (
          <p
            className="text-xs mb-4"
            style={{ color: error.includes('reprocessada') ? '#3ddc84' : '#f87171' }}
          >
            {error}
          </p>
        )}

        <div className="flex gap-1 mb-4 flex-wrap">
          {FILTERS.map(({ key, label }) => {
            const count = countFor(key);
            const active = filter === key;
            return (
              <button
                key={key}
                onClick={() => setFilter(key)}
                className="px-3 py-1.5 rounded-lg text-xs font-medium transition-colors"
                style={{
                  background: active ? 'rgba(170,59,255,0.15)' : '#111',
                  color: active ? '#aa3bff' : '#555',
                  border: `1px solid ${active ? '#aa3bff' : '#1e1e1e'}`,
                  cursor: 'pointer',
                }}
              >
                {label} ({count})
              </button>
            );
          })}
        </div>

        <div className="flex flex-col gap-2">
          {visible.map(msg => {
            const isOpen = expanded === msg.id;
            const badgeStyle = STATUS_STYLE[msg.status] ?? STATUS_STYLE['pending'];
            const isPending = msg.status === 'pending';

            return (
              <div
                key={msg.id}
                className="rounded-lg"
                style={{
                  background: '#111',
                  border: `1px solid ${isOpen ? '#333' : '#1e1e1e'}`,
                }}
              >
                <div
                  className="px-4 py-3 cursor-pointer"
                  onClick={() => setExpanded(isOpen ? null : msg.id)}
                >
                  <div className="flex justify-between items-start">
                    <div className="flex-1 min-w-0 mr-4">
                      <div className="flex items-center gap-2 flex-wrap">
                        <p className="text-sm font-medium text-white">{msg.name}</p>
                        <TopicBadge topic={msg.originalTopic} />
                      </div>
                      <p className="text-xs mt-1 truncate" style={{ color: '#f87171' }}>{msg.error}</p>
                      <p className="text-xs mt-0.5" style={{ color: '#444' }}>
                        Falhou em {formatTime(msg.failedAt)}
                      </p>
                    </div>
                    <div className="flex items-center gap-2 shrink-0">
                      <span className="text-xs font-medium px-2 py-0.5 rounded-full" style={badgeStyle}>
                        {msg.status}
                      </span>
                      <span className="text-xs" style={{ color: '#333' }}>{isOpen ? '▲' : '▼'}</span>
                    </div>
                  </div>
                </div>

                {isOpen && (
                  <div
                    className="px-4 pb-4"
                    style={{ borderTop: '1px solid #1a1a1a' }}
                    onClick={e => e.stopPropagation()}
                  >
                    <p className="text-xs mt-3 mb-1.5 font-medium" style={{ color: '#666' }}>Payload</p>
                    <textarea
                      className="w-full font-mono text-xs rounded-md p-3 resize-y"
                      rows={8}
                      style={{
                        background: '#0d0d0d',
                        border: '1px solid #222',
                        color: '#c4c4c4',
                        outline: 'none',
                      }}
                      value={getPayload(msg)}
                      onChange={e => setEditedPayloads(prev => ({ ...prev, [msg.id]: e.target.value }))}
                      readOnly={!isPending}
                    />
                    {isPending && (
                      <div className="flex gap-2 mt-3">
                        <Button
                          variant="primary"
                          loading={!!loading[msg.id]}
                          onClick={() => handleReprocessOne(msg)}
                        >
                          Reprocessar
                        </Button>
                        <Button
                          variant="secondary"
                          loading={!!loading[`dismiss-${msg.id}`]}
                          onClick={() => handleDismiss(msg)}
                        >
                          Descartar
                        </Button>
                      </div>
                    )}
                  </div>
                )}
              </div>
            );
          })}

          {visible.length === 0 && !error && (
            <p className="text-sm py-12 text-center" style={{ color: '#444' }}>
              Nenhuma mensagem encontrada.
            </p>
          )}
        </div>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Add route to router.tsx**

In `arrakis/src/router.tsx`:
- Add import: `import { Dlq } from './pages/admin/Dlq';`
- Add route inside the protected `<Route>` block:
  ```tsx
  <Route path="/admin/dlq" element={<Dlq />} />
  ```

- [ ] **Step 3: Add nav item to AppLayout.tsx**

In `arrakis/src/components/AppLayout.tsx`, add to `NAV_ITEMS`:

```tsx
const DLQ_ICON = (
  <svg width="15" height="15" viewBox="0 0 15 15" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round" aria-hidden>
    <rect x="1" y="1" width="13" height="13" rx="2" />
    <path d="M5 5l5 5M10 5l-5 5" />
  </svg>
);
```

Add after the Journeys item:
```ts
{ to: '/admin/dlq', label: 'DLQ', icon: DLQ_ICON, adminOnly: true },
```

- [ ] **Step 4: Build arrakis**

```bash
cd arrakis && npx tsc --noEmit
```

Expected: no errors.

- [ ] **Step 5: Commit**

```bash
git -c user.name="Joao Vitor" -c user.email="joaorabello99@gmail.com" \
  commit -m "feat(harkonnen): add DLQ admin page, route and nav item"
```

---

## Self-Review

**Spec coverage:**
- ✅ Reprocess all by topic — `POST /harkonnen/reprocess-all` + "Reprocessar tudo: {topic}" button per pending topic
- ✅ Reprocess per message — `POST /harkonnen/reprocess` + per-card Reprocess button
- ✅ Edit payload before reprocess — editable textarea, stored in local state, sent in POST body
- ✅ Dismiss message — `POST /harkonnen/dismiss` + Dismiss button
- ✅ Persisted to Postgres — `harkonnen_messages` table with JSONB payload
- ✅ Status transitions: pending → reprocessed / dismissed
- ✅ DLQ consumer avoids DLQ-of-DLQ by using `kafka.consumer()` directly
- ✅ Dune-themed name: harkonnen

**Type consistency check:**
- `HarkonnenMessage.payload` is `field.string()` throughout (JSON.stringify in fromDbWire, JSON.parse in toDbWire and controllers) ✅
- `DlqMessage` in imperium mirrors `HarkonnenMessage` in odyssey ✅
- `DlqMessage` in arrakis type mirrors API response ✅
- `ReprocessOneWireIn.id` is `field.uuid()` in both odyssey and imperium ✅
- `NoInput` reused from `src/wire/in/no-input.ts` in imperium (already exists) ✅

**Placeholder scan:** None found.

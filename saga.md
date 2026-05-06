# Saga Kafka (odyssey)

> **TL;DR** — A jornada estudantil é uma Kafka saga **interna ao odyssey**. Nenhum serviço externo aciona os passos. Cada consumer avança o estado, publica o próximo evento e é idempotente (segundo delivery é no-op).

---

## Fluxo Completo

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
    └─► emit SSE
    └─► publish diagnosticTriggered { eventId, journeyId }
          │
         ...
          │
          ▼
  [consumer: progressMilestoneReached]
    └─► INSERT journey_completed
    └─► UPDATE journey.currentStep = JOURNEY_COMPLETED
    └─► UPDATE journey.status = completed
    └─► emit SSE
    └─► (fim da saga)
```

---

## Envelope de Evento

Cada mensagem Kafka carrega apenas:

```ts
{ eventId: UUID, journeyId: UUID }
```

`eventId` é o ID do registro do **passo anterior** no banco. O consumer usa para verificar se a cadeia não foi quebrada antes de inserir o próximo.

---

## Padrão de Idempotência

Todo controller segue o mesmo esqueleto:

```ts
// 1. Verificar passo anterior
const previous = await findById(event.eventId);
if (!previous) throw new NotFoundError('...');

// 2. Guard de idempotência — já processado?
const existing = await thisStepDb.findById(event.eventId);
if (existing) {
  await publish(buildEvent({ journeyId: existing.journeyId, eventId: existing.id }));
  return; // no-op — só re-publica
}

// 3. Inserir registro do passo atual
const current = await thisStepDb.insert(buildEventRecord(event));

// 4. Avançar estado da jornada
await journeyDb.updateStep(buildJourneyStepUpdate({ id: event.journeyId, currentStep: 'NEXT_STEP' }));

// 5. Emitir SSE in-process
journeyEventBus.emit(event.journeyId, { ... });

// 6. Publicar próximo evento Kafka
await sideEffect(buildEvent({ journeyId: current.journeyId, eventId: current.id }));
```

**Por que funciona com at-least-once:** o passo 2 detecta se o registro já existe antes de inserir. Se o Kafka entregou duas vezes, o segundo delivery só re-publica e retorna.

---

## Tabela de Passos

| Passo | Consome | Insere | Publica | Estado da jornada |
|-------|---------|--------|---------|------------------|
| journeyStarted | `journeyInitiated` | `diagnostic_triggered` | `diagnosticTriggered` | `DIAGNOSTIC_TRIGGERED` |
| diagnosticTriggered | `diagnosticTriggered` | `diagnostic_completed` | `diagnosticCompleted` | `DIAGNOSTIC_COMPLETED` |
| diagnosticCompleted | `diagnostic_completed` | `analysis_started` | `analysisStarted` | `ANALYSIS_STARTED` |
| analysisStarted | `analysis_started` | `analysis_finished` | `analysisFinished` | `ANALYSIS_FINISHED` |
| analysisFinished | `analysis_finished` | `curriculum_generated` | `curriculumGenerated` | `CURRICULUM_GENERATED` |
| curriculumGenerated | `curriculum_generated` | `content_dispatched` | `contentDispatched` | `CONTENT_DISPATCHED` |
| contentDispatched | `content_dispatched` | `student_engagement_received` | `studentEngagementReceived` | `STUDENT_ENGAGEMENT_RECEIVED` |
| studentEngagementReceived | `student_engagement_received` | `progress_milestone_reached` | `progressMilestoneReached` | `PROGRESS_MILESTONE_REACHED` |
| progressMilestoneReached | `progress_milestone_reached` | `journey_completed` | — | `JOURNEY_COMPLETED` / `completed` |

---

## Consumer Groups

Cada tópico tem seu próprio consumer group: `student-journey-<eventName>`. Isso significa que um consumer lento não bloqueia os demais — cada grupo tem seu offset independente.

---

## Eventos do atreides (fora da saga)

| Tópico | Gatilho | Payload |
|--------|---------|---------|
| `userCreated` | `POST /users` | `{ userId, email, role }` |
| `mailConfirmed` | `POST /users/confirm-email` | `{ userId, email }` |

Nenhum serviço consome esses eventos ainda — estão disponíveis para consumidores futuros.

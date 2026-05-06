# SSE — Atualizações em Tempo Real

> **TL;DR** — Cada passo da saga emite um evento num `EventEmitter` in-process. A rota `GET /journeys/:journeyId/stream` escuta esse bus e envia cada transição para o browser via SSE. Se a jornada travar, `POST /journeys/republish` re-publica o último evento Kafka.

---

## Como Funciona

Após cada passo da saga, o controller chama:

```ts
journeyEventBus.emit(journeyId, { id, currentStep, status });
```

O `journeyEventBus` é um `EventEmitter` singleton definido em `src/wire/journey-event-bus.ts`. A rota SSE escuta por `journeyId`:

```ts
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

Quando um passo completa, o browser recebe a nova state imediatamente — sem polling.

---

## Rota no Browser (arrakis)

```ts
// hooks/useJourneyStream.ts
const es = new EventSource(
  `${IMPERIUM_URL}/journeys/${journeyId}/stream?token=${token}`
);
es.onmessage = (e) => setJourney(JSON.parse(e.data));
```

O token vai como query param porque `EventSource` não aceita headers customizados.

---

## Limitação

O bus in-process só funciona quando o browser e o consumer Kafka rodam na mesma instância do odyssey. Em deploy horizontal com múltiplas réplicas, seria necessário um Redis pub/sub ou Kafka-backed SSE gateway.

---

## Jornadas Travadas — `POST /journeys/republish`

Se o odyssey cair entre o INSERT no banco e o publish no Kafka, a jornada fica travada — o estado está salvo mas nenhum consumer avança.

`republishStuckJourneys`:

```
1. Carrega todas as jornadas ativas
2. Para cada jornada ainda em JOURNEY_INITIATED:
   - Se journey_initiated existe → re-publica { eventId, journeyId }
   - Se não existe → cria o registro, então publica
3. Retorna { republished: N }
```

É seguro chamar a qualquer momento — o guard de idempotência em cada consumer garante que não haverá duplicate inserts.

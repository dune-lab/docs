# `@enxoval/messaging`

> **TL;DR** — Wrapper KafkaJS para publicar e consumir mensagens. Retry automático (3×) com backoff exponencial, DLQ automática no tópico `student-journey-dlq`. Consumer usa groupId derivado do nome do evento.

---

## Versão atual: 1.0.2

---

## `publish`

Publica uma mensagem em um tópico Kafka derivado do `name`.

```ts
import { publish } from '@enxoval/messaging';

await publish('diagnosticTriggered', {
  eventId: crypto.randomUUID(),
  journeyId,
  studentId,
  cid: request.cid,
});
```

- O tópico é derivado do `name` via `getKafkaTopic()` (geralmente `student-journey-<name>` ou configurado via env var `KAFKA_TOPIC_<NAME>`)
- `cid` é propagado: se presente no payload, o producer aplica `nextCid()` antes de publicar
- Log automático: `producer: message published` com `topic`, `name`, `eventId`

---

## `publishRaw`

Publica em um tópico arbitrário sem derivação de nome. Usado internamente para DLQ.

```ts
import { publishRaw } from '@enxoval/messaging';

await publishRaw('student-journey-dlq', {
  originalTopic: 'student-journey-diagnosticTriggered',
  name: 'diagnosticTriggered',
  payload: { ... },
  error: 'Timeout after 3 retries',
  failedAt: new Date().toISOString(),
});
```

---

## `subscribe`

Registra um consumer Kafka para o tópico derivado de `name`.

```ts
import { subscribe } from '@enxoval/messaging';

subscribe('diagnosticTriggered', async (message) => {
  await processDiagnostic(message);
});
```

### Comportamento interno

1. `groupId` = `student-journey-<name>`
2. `fromBeginning: false` — lê apenas mensagens novas
3. Retry automático: 3 tentativas com backoff exponencial (500ms, 1s, 2s)
4. Se falhar após 3× → publica no `student-journey-dlq` e loga `consumer: message failed after retries, sending to DLQ`
5. Logs automáticos: `consumer: subscribed` e `consumer: message received`

### Tipagem do handler

```ts
type MessageHandler<T = unknown> = (message: T) => Promise<void> | void;
```

---

## `connect` / `disconnect`

Conecta/desconecta o producer Kafka. Deve ser chamado no startup/shutdown do serviço.

```ts
import { connect, disconnect } from '@enxoval/messaging';

// No setup do listen()
await connect();

// No shutdown
await disconnect();
```

---

## `ensureTopics`

Cria os tópicos no Kafka se ainda não existirem. Normalmente chamado uma vez no startup.

```ts
import { ensureTopics } from '@enxoval/messaging';

await ensureTopics(['student-journey-diagnosticTriggered', 'student-journey-dlq']);
```

---

## `kafka` (instância raw)

Acesso direto ao cliente KafkaJS para casos que precisam de `kafka.consumer()` diretamente (ex: consumer Harkonnen DLQ — para evitar DLQ-do-DLQ).

```ts
import { kafka } from '@enxoval/messaging';

const consumer = kafka.consumer({ groupId: 'student-journey-harkonnen-dlq' });
await consumer.connect();
await consumer.subscribe({ topic: 'student-journey-dlq', fromBeginning: true });
await consumer.run({
  eachMessage: async ({ message }) => {
    // processa sem retry automático
  },
});
```

---

## DLQ — Dead Letter Queue

O consumer Harkonnen persiste mensagens da DLQ em banco e expõe endpoints para reprocessamento. Ver [DLQ](../dlq.md) para detalhes completos.

| Tópico | Descrição |
|--------|-----------|
| `student-journey-dlq` | Mensagens que falharam após 3 tentativas |

```json
{
  "originalTopic": "student-journey-diagnosticTriggered",
  "name": "diagnosticTriggered",
  "payload": { "eventId": "...", "journeyId": "...", "studentId": "..." },
  "error": "timeout",
  "failedAt": "2026-05-05T12:00:00.000Z"
}
```

---

## Variáveis de ambiente

| Variável | Descrição |
|----------|-----------|
| `KAFKA_BROKERS` | Lista de brokers, ex: `kafka:9092` |
| `KAFKA_CLIENT_ID` | Identificador do cliente, ex: `odyssey` |
| `KAFKA_TOPIC_<NAME>` | Override do tópico para o evento `<name>` |

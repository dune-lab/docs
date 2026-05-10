# `@enxoval/messaging`

> **TL;DR** — Wrapper KafkaJS para publicar e consumir mensagens. Retry automático (3×) com backoff exponencial, DLQ automática no tópico `${SERVICE_NAME}-dlq` (criado automaticamente por `ensureTopics`). Consumer usa groupId derivado do nome do serviço e do evento.

---

## Versão atual: 1.0.4

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

- O tópico é derivado do `name` via `getKafkaTopic()` — configurado no `{SERVICE_NAME}.json` do serviço
- `cid` é propagado: se presente no payload, o producer aplica `nextCid()` antes de publicar
- Log automático: `producer: message published` com `topic`, `name`, `eventId`

---

## `publishRaw`

Publica em um tópico arbitrário sem derivação de nome. Usado internamente para DLQ.

```ts
import { publishRaw } from '@enxoval/messaging';

await publishRaw(`${serviceName}-dlq`, {
  originalTopic: 'odyssey-diagnosticTriggered',
  name: 'diagnosticTriggered',
  payload: { ... },
  error: 'Timeout after 3 retries',
  failedAt: new Date().toISOString(),
});
```

---

## `consume`

Registra um consumer Kafka para o tópico derivado de `name`.

```ts
import { consume } from '@enxoval/messaging';

consume('diagnosticTriggered', async (message) => {
  await processDiagnostic(message);
});
```

### Comportamento interno

1. `groupId` = `${SERVICE_NAME}-<name>` (ex: `odyssey-diagnosticTriggered`)
2. `fromBeginning: false` — lê apenas mensagens novas
3. Retry automático: 3 tentativas com backoff exponencial (500ms, 1s, 2s)
4. Se falhar após 3× → publica no `${SERVICE_NAME}-dlq` e loga `consumer: message failed after retries, sending to DLQ`
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

Lê o `{SERVICE_NAME}.json` do diretório de trabalho, cria os tópicos declarados e **automaticamente cria o tópico DLQ** `${SERVICE_NAME}-dlq`. Chamado uma vez no startup.

```ts
import { ensureTopics } from '@enxoval/messaging';

await ensureTopics();
```

- Lê `${SERVICE_NAME}.json` (ou `student-journey.json` se `SERVICE_NAME` não estiver definido)
- Cria todos os tópicos declarados em `kafka_topics` que ainda não existem no broker
- Cria `${SERVICE_NAME}-dlq` automaticamente — **não precisa declarar no JSON do serviço**
- Se o arquivo não existir, ignora silenciosamente (sem erro)

---

## `kafka` (instância raw)

Acesso direto ao cliente KafkaJS para casos que precisam de `kafka.consumer()` diretamente (ex: consumer Harkonnen DLQ — para evitar DLQ-do-DLQ).

```ts
import { kafka } from '@enxoval/messaging';

const serviceName = process.env.SERVICE_NAME || 'odyssey';
const consumer = kafka.consumer({ groupId: `${serviceName}-harkonnen-dlq` });
await consumer.connect();
await consumer.subscribe({ topic: `${serviceName}-dlq`, fromBeginning: true });
await consumer.run({
  eachMessage: async ({ message }) => {
    // processa sem retry automático
  },
});
```

---

## Schemas — `HarkonnenMessage` / `HarkonnenMessageInput`

Os schemas da DLQ são centralizados em `@enxoval/messaging` — **não defina localmente**.

```ts
import {
  HarkonnenMessage,
  HarkonnenMessageInput,
  HARKONNEN_STATUSES,
} from '@enxoval/messaging';
```

| Export | Descrição |
|--------|-----------|
| `HarkonnenMessage` | Schema completo da mensagem DLQ (persistida no banco) |
| `HarkonnenMessageInput` | Schema de entrada ao receber mensagem da DLQ |
| `HARKONNEN_STATUSES` | `['pending', 'reprocessed', 'dismissed']` |

### Campos de `HarkonnenMessage`

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | `uuid` | Identificador único |
| `originalTopic` | `string` | Tópico onde a mensagem falhou |
| `name` | `string` | Nome do evento |
| `payload` | `string` | JSON serializado do payload original |
| `error` | `string` | Mensagem de erro |
| `failedAt` | `string` | ISO string do momento da falha |
| `status` | `'pending' \| 'reprocessed' \| 'dismissed'` | Estado atual |
| `reprocessedAt` | `string \| null` | ISO string do reprocessamento |
| `createdAt` | `string` | ISO string de criação |

### Campos de `HarkonnenMessageInput`

| Campo | Tipo |
|-------|------|
| `originalTopic` | `string` |
| `name` | `string` |
| `payload` | `string` |
| `error` | `string` |
| `failedAt` | `string` |

---

## DLQ — Dead Letter Queue

O consumer Harkonnen persiste mensagens da DLQ em banco e expõe endpoints para reprocessamento. Ver [DLQ](../dlq.md) para detalhes completos.

| Tópico | Criado por | Descrição |
|--------|-----------|-----------|
| `${SERVICE_NAME}-dlq` | `ensureTopics()` automaticamente | Mensagens que falharam após 3 tentativas |

O tópico DLQ **não precisa ser declarado** no `{SERVICE_NAME}.json` do serviço — `ensureTopics()` o cria automaticamente com base na variável `SERVICE_NAME`.

---

## Variáveis de ambiente

| Variável | Obrigatória | Descrição |
|----------|-------------|-----------|
| `KAFKA_BROKERS` | sim | Lista de brokers, ex: `kafka:9092` |
| `SERVICE_NAME` | sim | Nome do serviço (ex: `odyssey`). Determina o clientId Kafka, o groupId dos consumers, o arquivo de config e o tópico DLQ |
| `KAFKA_TOPIC_<NAME>` | não | Override do tópico para o evento `<name>` |

# DLQ — Dead Letter Queue (Harkonnen)

> **TL;DR** — Se um consumer Kafka falhar 3×, a mensagem vai para `${SERVICE_NAME}-dlq`. O consumer Harkonnen persiste ela na tabela `harkonnen_messages`. Operadores podem reprocessar (re-publicar no tópico original) ou dar dismiss via API ou UI.

---

## Fluxo

```
consume('diagnosticTriggered', handler)
  falha × 3
  └─► @enxoval/messaging → publishRaw '${SERVICE_NAME}-dlq'
            │
            ▼
  consumer harkonnen-dlq (kafka.consumer() direto)
    └─► INSERT harkonnen_messages { id, originalTopic, name, payload, error, failedAt, status: 'pending' }
```

O consumer Harkonnen usa `kafka.consumer()` direto (sem o helper `consume()`) para evitar criar um DLQ-do-DLQ.

---

## Tópico DLQ

O tópico `${SERVICE_NAME}-dlq` é criado **automaticamente** por `ensureTopics()` no startup do serviço — **não é necessário declará-lo** no `{SERVICE_NAME}.json`.

---

## Schemas

Os schemas da DLQ estão centralizados em `@enxoval/messaging`:

```ts
import {
  HarkonnenMessage,
  HarkonnenMessageInput,
  HARKONNEN_STATUSES,
} from '@enxoval/messaging';
```

Ver [messaging.md](enxoval/messaging.md#schemas--harkonnenm essage--harkonnenm essageinput) para a definição completa dos campos.

---

## Reprocess

```
POST /harkonnen/reprocess { id, payload? }
  └─► busca harkonnen_messages por id
  └─► republica no tópico original com payload (editado ou original)
  └─► UPDATE status = 'reprocessed', reprocessedAt = now()
```

O `payload` pode ser editado antes do reprocess para corrigir dados corrompidos.

---

## Endpoints

| Método | Rota | Descrição |
|--------|------|-----------|
| `GET` | `/harkonnen` | Lista todas as mensagens DLQ |
| `POST` | `/harkonnen/reprocess` | Reprocessa uma mensagem pelo id |
| `POST` | `/harkonnen/reprocess-all` | Reprocessa todas as `pending` de um tópico |
| `POST` | `/harkonnen/dismiss` | Marca como `dismissed` (sem reprocessar) |

Todos disponíveis no odyssey diretamente e via imperium.

---

## Status

| Status | Significado |
|--------|-------------|
| `pending` | Aguardando ação do operador |
| `reprocessed` | Re-publicado no tópico original |
| `dismissed` | Descartado — operador reconheceu o erro |

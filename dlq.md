# DLQ — Dead Letter Queue (Harkonnen)

> **TL;DR** — Se um consumer Kafka falhar 3×, a mensagem vai para `student-journey-dlq`. O consumer Harkonnen persiste ela na tabela `harkonnen_messages`. Operadores podem reprocessar (re-publicar no tópico original) ou dar dismiss via API ou UI.

---

## Fluxo

```
consumer qualquer (ex: diagnosticTriggered)
  falha × 3
  └─► @enxoval/messaging → publish student-journey-dlq
            │
            ▼
  consumer harkonnen-dlq
    └─► INSERT harkonnen_messages { id, topic, payload, status: pending }
```

O consumer Harkonnen usa `kafka.consumer()` direto (sem o helper `subscribe()`) para evitar criar um DLQ-do-DLQ.

---

## Reprocess

```
POST /harkonnen/reprocess { id, payload? }
  └─► busca harkonnen_messages por id
  └─► republica no tópico original com payload (editado ou original)
  └─► UPDATE status = reprocessed
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

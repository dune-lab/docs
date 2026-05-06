# `@enxoval/observability`

> **TL;DR** — Logger Pino estruturado com campo `service` fixo. Correlation ID (cid) gerado por request e propagado entre serviços via header `x-cid`. Em produção emite JSON; em dev usa `pino-pretty`.

---

## Versão atual: 1.0.2

---

## `logger`

Instância Pino pré-configurada. Usar em todo log da aplicação.

```ts
import { logger } from '@enxoval/observability';

logger.info({ cid, journeyId }, 'logic: journey started');
logger.warn({ cid, statusCode: 404 }, 'http-client: journey not found');
logger.error({ err, cid }, 'consumer: processing failed');
```

### Configuração

| Campo | Valor |
|-------|-------|
| `service` | `process.env.SERVICE_NAME ?? 'unknown'` |
| `level` | `process.env.LOG_LEVEL ?? 'info'` |
| `timestamp` | ISO 8601 (`2026-05-05T12:00:00.000Z`) |
| Formato | JSON em produção; pretty-print com cores em dev (`NODE_ENV !== 'production'`) |

### Schema do log

```json
{
  "level": "info",
  "time": "2026-05-05T12:00:00.000Z",
  "service": "odyssey",
  "cid": "e002088d:1",
  "method": "POST",
  "url": "/journeys",
  "status": 201,
  "durationMs": 42,
  "msg": "http-server: response sent"
}
```

---

## `newCid` / `nextCid`

Gera e propaga Correlation IDs entre serviços.

```ts
import { newCid, nextCid } from '@enxoval/observability';

// Gera um novo cid para uma request de entrada (sem cid upstream)
const cid = newCid();   // → 'e002088d:0'

// Incrementa o counter de um cid existente (chamada downstream)
const downstream = nextCid('e002088d:0');  // → 'e002088d:1'
const further    = nextCid('e002088d:1');  // → 'e002088d:2'
```

### Formato

`<prefixo>:<counter>` onde:
- `<prefixo>` = primeiros 8 chars de um UUID (`randomUUID().split('-')[0]`)
- `<counter>` = inteiro incrementado a cada hop entre serviços

### Propagação automática

O `@enxoval/http` lida com a propagação automaticamente:
- Request de entrada: lê `x-cid` do header → `nextCid()` se presente, `newCid()` se ausente
- Response: escreve o cid no header `x-cid` de saída

Para propagar manualmente em http-clients:

```ts
import { getCurrentUser } from '@enxoval/auth';

// O token já carrega o cid original — propagar via header x-cid
const user = getCurrentUser();
const headers = {
  authorization: `Bearer ${user?.token}`,
  'x-cid': cid,
};
```

---

## Variáveis de ambiente

| Variável | Descrição |
|----------|-----------|
| `SERVICE_NAME` | Nome do serviço que aparece no campo `service` de todo log |
| `LOG_LEVEL` | `debug`, `info`, `warn`, `error` (padrão: `info`) |
| `NODE_ENV` | `production` → JSON puro; qualquer outro → pino-pretty |

---

## Dois formatos de log no odyssey

| Origem | Campo msg | Campo level | Filtro LogQL |
|--------|-----------|-------------|-------------|
| Pino (app) | `msg` | `"info"` (lowercase) | `\| msg="..."` |
| kafkajs | `message` | `"ERROR"` (uppercase) | `\| message=~"..."` |

Em queries LogQL, sempre verificar qual campo usar para o serviço em questão.

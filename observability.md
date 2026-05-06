# Observabilidade

> **TL;DR** — Todos os serviços emitem JSON estruturado via Pino com campo `service`. Promtail coleta via Docker socket, indexa no Loki com `service` e `level` como labels. Grafana em `localhost:4001` exibe dashboards e alertas.

---

## Logs Estruturados (Pino)

Cada request HTTP emite dois logs:

```json
{ "level": "info", "service": "odyssey", "cid": "abc:0", "method": "POST", "url": "/journeys", "msg": "http-server: request received" }
{ "level": "info", "service": "odyssey", "cid": "abc:0", "method": "POST", "url": "/journeys", "status": 201, "durationMs": 42, "msg": "http-server: response sent" }
```

| Campo | Descrição |
|-------|-----------|
| `service` | Nome do serviço — label indexado no Loki |
| `cid` | Correlation ID — rastreia uma request em todos os logs |
| `level` | `info`, `warn`, `error` |
| `durationMs` | Tempo de resposta em ms (presente no "response sent") |
| `msg` | Descrição do evento |

`/health` e `OPTIONS` são suprimidos na origem.

---

## Dashboards Grafana (`localhost:4001`)

| Dashboard | O que mostra |
|-----------|-------------|
| Platform Status | Semáforo ONLINE/OFFLINE por serviço |
| Services | Taxa de requests, taxa de erro, latência p50 por serviço |
| HTTP Requests | Todo o tráfego HTTP |
| Kafka Messages | Atividade de producer/consumer |
| DLQ Monitor | Ingestão de mensagens falhadas + alerta |

---

## Alertas

| Alerta | Condição |
|--------|----------|
| Consumer Crash | `[Consumer] Crash` nos logs do kafkajs |
| DLQ Spike | Taxa de mensagens DLQ acima do threshold |
| 5xx Spike | Taxa de HTTP 5xx acima do threshold |
| Service Down | Sem logs de um serviço por 3+ minutos |

Para receber notificações: editar `platform/grafana/provisioning/alerting/contact-points.yml` com a URL do webhook.

---

## Queries LogQL Úteis

**Todos os requests HTTP:**
```logql
{service=~"odyssey|atreides|janus|imperium|persona"}
  | json | msg="http-server: response sent"
  | line_format "[{{.service}}] {{.method}} {{.url}} → {{.status}} ({{.durationMs}}ms)"
```

**Requests lentos (> 500ms):**
```logql
{service=~"odyssey|imperium"} | json | msg="http-server: response sent" | durationMs > 500
```

**Erros 4xx/5xx:**
```logql
{service=~"odyssey|atreides|janus|imperium|persona"}
  | json | statusCode >= 400
  | line_format "[{{.service}}] {{.method}} {{.url}} → {{.statusCode}} | {{.error}}: {{.message}}"
```

**Consumer crashes (kafkajs):**
```logql
{service=~"odyssey|atreides"} | json | message=~"\\[Consumer\\] Crash.*"
```

**DLQ — mensagens que falharam:**
```logql
{service="odyssey"} | json | msg="consumer: message failed after retries, sending to DLQ"
```

**Serviço sem logs (down detection):**
```logql
absent_over_time({service="odyssey"}[3m])
```

---

## Dois formatos de log no odyssey

| Origem | Campo msg | Exemplo |
|--------|-----------|---------|
| Pino (app) | `msg` | `"msg":"http-server: response sent"` |
| kafkajs | `message` | `"message":"[Consumer] Crash..."` |

Filtros em logs de app usam `msg=`. Filtros em logs do kafkajs usam `message=`.

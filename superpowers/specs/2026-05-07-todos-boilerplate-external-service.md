# TODOs — Boilerplate e Serviço de Integração Externa

> Documento de requisitos para os dois próximos itens de desenvolvimento da plataforma Dune Lab.

---

## TODO 1 — Boilerplate de Novo Repositório

### Objetivo

Padronizar a criação de novos serviços. Ao criar um novo repositório, ele já deve nascer com a estrutura e dependências corretas, sem precisar copiar manualmente de outro serviço.

### O que o boilerplate deve incluir

Baseado nos serviços existentes (`atreides`, `odyssey`, `persona`), o template deve conter:

**Estrutura de pastas:**
```
src/
  adapters/       # adaptadores externos (HTTP clients, brokers)
  controllers/    # handlers de entrada (rotas HTTP, consumers Kafka)
  db/
    migrations/   # arquivos de migração
  logic/          # regras de negócio
  model/          # tipos e entidades de domínio
  wire/           # wiring de dependências (injeção)
  app.ts
  server.ts
```

**Dependências pré-instaladas (`@enxoval/*`):**
- `@enxoval/http` — servidor HTTP
- `@enxoval/db` — ORM/migrations (PostgreSQL)
- `@enxoval/messaging` — Kafka producer/consumer
- `@enxoval/auth` — autenticação JWT
- `@enxoval/observability` — logs e métricas
- `@enxoval/types` — tipos compartilhados

**DevDependencies:**
- `@enxoval/quality` — eslint + prettier
- `@enxoval/testing` — vitest config
- `typescript`, `tsx`, `nodemon`, `pino-pretty`, `reflect-metadata`

**Arquivos de config:**
- `tsconfig.json`
- `eslint.config.ts`
- `nodemon.json`
- `package.json` com todos os scripts padrão (`dev`, `build`, `start`, `lint`, `test`, `migration:*`)
- `Dockerfile`
- `docker-compose.yml` (para desenvolvimento local isolado)
- `.env.example`

### Resultado esperado

Um repositório template (`enxoval/boilerplate` ou similar) que pode ser clonado como base para qualquer novo serviço, eliminando setup manual.

---

## TODO 2 — Serviço Intermediário para Integrações Externas

### Objetivo

Isolar toda a camada de comunicação com serviços terceiros em um único serviço dedicado. Qualquer outro serviço da plataforma que precise chamar uma API externa deve **publicar uma mensagem Kafka** em vez de fazer a chamada HTTP diretamente.

### Problema que resolve

Hoje, se um serviço externo muda de URL, método ou contrato, é preciso alterar todos os serviços da plataforma que o chamam. Com este serviço intermediário:
- **Mudança de URL/método** → altera só a `request_config` no banco, nada mais
- **Falha do serviço externo** → a mensagem vai para a DLQ, pode ser monitorada e reenviada sem impacto na aplicação

### Fluxo de funcionamento

```
[Serviço interno]
      │
      │ publica mensagem Kafka
      ▼
[Tópico Kafka: external-requests]
      │
      │ consome
      ▼
[Serviço intermediário]
      │
      ├─ 1. busca configuração pelo `alias` na tabela `request_config`
      ├─ 2. monta a requisição (method + path + payload + headers)
      ├─ 3. executa a chamada HTTP para o serviço externo
      ├─ 4. salva o registro na tabela `request`
      │
      ├─ [sucesso] → fluxo encerrado
      └─ [falha]   → publica na DLQ para monitoramento e retry
```

### Banco de dados

**Tabela `request_config`** — catálogo de endpoints externos

| Coluna   | Tipo    | Restrições        | Exemplo                  |
|----------|---------|-------------------|--------------------------|
| `id`     | UUID    | NOT NULL, PK      | `550e8400-...`           |
| `alias`  | VARCHAR | NOT NULL, UNIQUE  | `externalServicePath1`   |
| `path`   | VARCHAR | NOT NULL          | `/api/v1/emit`           |
| `method` | VARCHAR | NOT NULL          | `POST`                   |

> O campo `alias` é a chave de lookup — é o que os serviços internos usam para identificar o destino sem conhecer a URL real.

**Tabela `request`** — log de todas as chamadas realizadas

| Coluna      | Tipo    | Restrições   | Exemplo              |
|-------------|---------|--------------|----------------------|
| `id`        | UUID    | NOT NULL, PK | `550e8400-...`       |
| `config_id` | UUID    | NOT NULL, FK | ref. `request_config`|
| `payload`   | JSONB   | NOT NULL     | `{"key": "value"}`   |
| `header`    | JSONB   | NOT NULL     | `{"Authorization": "Bearer ..."}` |

### Mensagem Kafka consumida

**Tópico:** `external-requests` (sugestão)

```json
{
  "alias": "externalServicePath1",
  "payload": { ... },
  "header": { "Authorization": "Bearer ..." }
}
```

### DLQ (Dead Letter Queue)

Se a chamada HTTP ao serviço externo falhar (timeout, 5xx, erro de rede), o serviço deve:
1. Publicar a mensagem original no tópico DLQ (ex: `external-requests.dlq`)
2. Registrar o erro nos logs (via `@enxoval/observability`)
3. **Não** relançar exceção para o consumer — a mensagem deve ir para DLQ sem bloquear o processamento das próximas

O Grafana já provisionado na plataforma pode ser usado para monitorar o volume de mensagens na DLQ.

### Onde o serviço deve ser cadastrado

Ao criar o serviço, ele deve ser adicionado ao `docker-compose.yml` da platform com:
- Banco de dados PostgreSQL próprio
- Variáveis de ambiente: `KAFKA_BROKER`, `DB_*`, `EXTERNAL_SERVICE_BASE_URL` (a URL base do serviço externo)

### Nome sugerido para o serviço

Seguindo o padrão da plataforma (universo Dune): **`herald`** — o mensageiro que entrega recados entre mundos.

---

## Dependências entre os TODOs

```
TODO 1 (boilerplate) ──► TODO 2 (serviço externo usa o boilerplate como base)
```

O boilerplate deve ser concluído primeiro para que o novo serviço de integração externa já nasça com a estrutura padrão.

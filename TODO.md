# TODO

Lista de iniciativas planejadas para a plataforma Dune Lab.

---

## 1. Boilerplate de Novo Repositório

**Status:** Pendente

### O que é

Um repositório template que serve de ponto de partida para qualquer novo serviço da plataforma. Ao criar um novo repo, basta clonar o boilerplate — a estrutura e as dependências já estão prontas.

### Por que fazer

Hoje criar um novo serviço exige copiar manualmente arquivos de outro serviço e ajustar configurações. Isso é lento, propenso a erro e gera inconsistência entre serviços.

### O que deve ter

**Estrutura de pastas:**
```
src/
  adapters/     # adaptadores para dependências externas (HTTP clients, brokers)
  controllers/  # entradas da aplicação (rotas HTTP, consumers Kafka)
  db/
    migrations/ # arquivos de migração de banco
  logic/        # regras de negócio
  model/        # tipos e entidades de domínio
  wire/         # injeção de dependências
  app.ts
  server.ts
```

**Dependências pré-instaladas:**
| Pacote                   | Finalidade                        |
|--------------------------|-----------------------------------|
| `@enxoval/http`          | Servidor HTTP                     |
| `@enxoval/db`            | ORM e migrations (PostgreSQL)     |
| `@enxoval/messaging`     | Kafka producer e consumer         |
| `@enxoval/auth`          | Autenticação JWT                  |
| `@enxoval/observability` | Logs e métricas                   |
| `@enxoval/types`         | Tipos compartilhados              |
| `@enxoval/quality`       | ESLint + Prettier (dev)           |
| `@enxoval/testing`       | Configuração Vitest (dev)         |

**Arquivos de configuração prontos:**
- `tsconfig.json`
- `eslint.config.ts`
- `nodemon.json`
- `package.json` com todos os scripts padrão (`dev`, `build`, `start`, `lint`, `test`, `migration:*`)
- `Dockerfile`
- `docker-compose.yml`
- `.env.example`

---

## 2. Serviço Intermediário para Integrações Externas

**Status:** Pendente  
**Depende de:** TODO 1 (Boilerplate)

### O que é

Um serviço dedicado exclusivamente a se comunicar com APIs de terceiros. Nenhum outro serviço da plataforma chama uma API externa diretamente — todos publicam uma mensagem Kafka e este serviço faz a chamada.

### Por que fazer

Sem este serviço, cada integração com terceiro fica espalhada pela plataforma. Se o serviço externo muda de URL ou de contrato, é preciso alterar múltiplos lugares.

Com este serviço:
- **Mudança de URL ou método** → altera só uma linha no banco de dados, em um único lugar
- **Serviço externo fora do ar** → a mensagem vai para a DLQ; pode ser monitorada e reenviada sem impacto no restante da plataforma

### Como funciona

```
[Qualquer serviço interno]
         │
         │  publica mensagem Kafka
         ▼
  [Tópico: external-requests]
         │
         │  consome
         ▼
  [Serviço intermediário]
         │
         ├─ 1. recebe: { alias, payload, header }
         ├─ 2. busca a configuração do endpoint pelo alias no banco
         ├─ 3. monta e executa a requisição HTTP para o serviço externo
         ├─ 4. salva o registro da chamada no banco
         │
         ├─ [sucesso] ──► encerrado
         └─ [falha]   ──► publica na DLQ + loga o erro
```

### Mensagem Kafka recebida

**Tópico:** `external-requests`

```json
{
  "alias": "externalServicePath1",
  "payload": { },
  "header": { "Authorization": "Bearer ..." }
}
```

O campo `alias` é a chave que identifica qual serviço externo chamar e como. O serviço interno não precisa saber a URL real — só o alias.

### Banco de dados

**Tabela `request_config`** — catálogo de endpoints externos

| Coluna   | Tipo    | Restrições        | Exemplo                |
|----------|---------|-------------------|------------------------|
| `id`     | UUID    | PK, NOT NULL      | `550e8400-...`         |
| `alias`  | VARCHAR | NOT NULL, UNIQUE  | `externalServicePath1` |
| `path`   | VARCHAR | NOT NULL          | `/api/v1/emit`         |
| `method` | VARCHAR | NOT NULL          | `POST`                 |

> Cada linha representa um endpoint de um serviço externo. Para redirecionar uma integração para uma nova URL, basta atualizar o `path` desta tabela.

---

**Tabela `request`** — log de todas as chamadas realizadas

| Coluna      | Tipo  | Restrições        | Exemplo                     |
|-------------|-------|-------------------|-----------------------------|
| `id`        | UUID  | PK, NOT NULL      | `550e8400-...`              |
| `config_id` | UUID  | FK, NOT NULL      | referência à `request_config` |
| `payload`   | JSONB | NOT NULL          | `{ "key": "value" }`        |
| `header`    | JSONB | NOT NULL          | `{ "Authorization": "..." }`|

> Registra cada chamada feita, com os dados enviados. Útil para auditoria e debug.

### DLQ (Dead Letter Queue)

Se a chamada HTTP falhar (timeout, erro 5xx, erro de rede):

1. A mensagem original é publicada no tópico `external-requests.dlq`
2. O erro é registrado nos logs
3. O consumer **não** é bloqueado — o próximo item da fila é processado normalmente

O volume de mensagens na DLQ pode ser monitorado no Grafana já provisionado na plataforma.

### Nome do serviço

**`herald`** — seguindo o padrão de nomes do universo Dune. O herald é o mensageiro que entrega recados entre mundos.

---

## Ordem de execução

```
TODO 1 — Boilerplate
    └─► TODO 2 — Herald (serviço intermediário)
```

O boilerplate deve ser concluído primeiro para que o `herald` já nasça com a estrutura padrão da plataforma.

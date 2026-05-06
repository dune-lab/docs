# Rodando Localmente

> **TL;DR** — `cd platform && docker compose up`. Todos os serviços sobem juntos. Para desenvolvimento, rode o serviço que você está mudando fora do Docker com `npm run dev` e ajuste a env var correspondente no `docker-compose.yml`.

---

## Subir tudo

```bash
cd platform
docker compose up
```

Após subir, os serviços ficam disponíveis em:

| Serviço | URL |
|---------|-----|
| arrakis (frontend) | http://localhost:5173 |
| imperium (BFF) | http://localhost:3004 |
| janus | http://localhost:3003 |
| atreides | http://localhost:3002 |
| persona | http://localhost:3000 |
| odyssey | http://localhost:3001 |
| Grafana | http://localhost:4001 |
| Kafka (broker) | localhost:9092 |

---

## Rebuild após mudança de código

```bash
docker compose build <serviço>
docker compose up -d <serviço>
```

Ou rebuild completo:

```bash
docker compose build --no-cache
docker compose up -d
```

---

## Rodar um serviço fora do Docker (dev com hot reload)

```bash
cd odyssey
cp .env.example .env   # configurar variáveis locais
npm install
npm run dev
```

No `platform/docker-compose.yml`, apontar o `ODYSSEY_URL` do imperium para `http://host.docker.internal:3001`.

---

## Verificar logs

```bash
docker compose logs <serviço> -f
docker compose logs odyssey atreides -f  # múltiplos
```

---

## Migrations

```bash
# Dentro de um serviço (ex: odyssey)
npm run migration:run     # aplicar pendentes
npm run migration:revert  # reverter a última
```

As migrations rodam automaticamente no startup do serviço em produção.

---

## Kafka fora do ar

Se o Kafka não estiver `(healthy)`, rotas que publicam eventos ficam aguardando indefinidamente.

```bash
docker compose ps kafka        # verificar status
docker compose up kafka -d     # subir se estiver down
```

---

## Colima no macOS

O Docker no macOS usa Colima para port-forwarding. **Nunca matar os processos `ssh` do Colima** — isso derruba o Docker daemon.

Se `Cannot connect to the Docker daemon`:
```bash
colima stop && colima start
```

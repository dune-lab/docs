# Design: Loop Completo — Onboarding, SSE e Admin

**Data:** 2026-05-05
**Status:** Aprovado

---

## Contexto

O dune-lab é uma plataforma de jornada estudantil com dois perfis de usuário: **estudantes** e **admins**. A plataforma está em fase de desenvolvimento. Os principais problemas identificados:

**Estudante:**
- Não sabe onde está na jornada nem o que fazer a seguir
- Não vê valor/contexto nas etapas
- Precisa recarregar a página para ver progresso atualizado

**Admin:**
- Não sabe quais alunos estão travados ou inativos
- Não tem visão agregada do progresso da turma
- Não consegue agir sobre alunos problemáticos

---

## Escopo

Três features independentes, implementadas em sequência:

1. **Onboarding guiado** — wizard pós-registro para o estudante
2. **SSE (Server-Sent Events)** — progresso em tempo real no dashboard
3. **Admin: filtro + inatividade** — visibilidade operacional para o admin

---

## Feature 1 — Onboarding Guiado

### Problema
Após o registro, o aluno sem perfil de estudante vê apenas um card genérico com "Complete seu perfil de aluno". Não há contexto, orientação ou motivação.

### Design

Wizard de 3 passos na rota `/onboarding`, substituindo o fluxo atual de link simples.

**Passo 1 — Boas-vindas**
- Exibe o nome do usuário (já cadastrado)
- Texto de boas-vindas explicando o que é a plataforma
- Botão "Continuar"

**Passo 2 — Área de interesse**
- Lista de seleção: Exatas, Humanas, Tecnologia, Saúde, Outros
- Seleção armazenada apenas em estado local (não persiste no backend — `StudentInput` não tem campo `interest`)
- Usada para personalizar o texto do passo 3 (ex: "Sua jornada de Tecnologia está prestes a começar")
- Botão "Continuar"

**Passo 3 — Pronto para começar**
- Explica em linguagem humana o que vai acontecer na jornada (as 10 etapas)
- Botão "Iniciar minha jornada" — chama `POST /journeys` via imperium e redireciona para `/`

**Guarda-chuvas:**
- Se o aluno já tem perfil e jornada ativa → redireciona direto para `/`
- Se o aluno já tem perfil mas sem jornada → vai para passo 3 diretamente

### Arquivos

| Arquivo | Ação |
|---|---|
| `arrakis/src/pages/onboarding/index.tsx` | Novo — controla estado do wizard (step atual) |
| `arrakis/src/pages/onboarding/Step1Welcome.tsx` | Novo |
| `arrakis/src/pages/onboarding/Step2Interest.tsx` | Novo |
| `arrakis/src/pages/onboarding/Step3Launch.tsx` | Novo |
| `arrakis/src/pages/onboarding/CreateStudent.tsx` | Remover (substituído pelo wizard) |
| `arrakis/src/pages/onboarding/StartJourney.tsx` | Remover (substituído pelo wizard) |
| `arrakis/src/router.tsx` | Substituir rotas `/onboarding/student` e `/onboarding/journey` por uma única `/onboarding` |
| `arrakis/src/pages/Dashboard.tsx` | Atualizar link de onboarding para `/onboarding` |

### Backend
Sem mudanças — usa `POST /students` (persona) e `POST /journeys` (odyssey via imperium), ambos já existentes.

---

## Feature 2 — SSE (Server-Sent Events)

### Problema
O dashboard carrega as jornadas uma vez. Quando um evento Kafka é processado e a jornada avança, o aluno precisa recarregar a página para ver.

### Design

O aluno abre o dashboard e vê a barra de progresso avançar em tempo real, sem refresh.

### Arquitetura

```
Kafka → odyssey consumer → salva no banco → emite no journeyEventBus
                                                        ↓
                          cliente SSE ← odyssey GET /journeys/:id/stream
                                ↑
                imperium GET /journeys/:id/stream?token=...
                                ↑
                arrakis EventSource → atualiza estado React
```

**Camada 1 — `@enxoval/http`**

Nova função exportada `sse(ParamsSchema, handler)`:
- Configura headers: `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `Connection: keep-alive`
- Expõe `send(data: object)` que serializa como `data: <json>\n\n` via `reply.raw`
- Handler recebe `(params, send, signal)` — `signal` é `AbortSignal` que dispara quando o cliente desconecta

**Camada 2 — `odyssey`**

Dois novos arquivos:

`src/wire/journey-event-bus.ts`
- `EventEmitter` singleton em memória
- API: `emit(journeyId, journey)`, `on(journeyId, cb)`, `off(journeyId, cb)`

`src/diplomat/http-server/journey-stream.ts`
- Rota `GET /journeys/:journeyId/stream` usando `sse`
- Registra listener no bus para o `journeyId`
- Envia `journey` atualizado a cada emissão
- Remove listener quando `signal` dispara (cliente desconectou)

Cada controller Kafka existente que atualiza uma jornada ganha uma linha:
```ts
journeyEventBus.emit(journey.id, journey);
```

**Camada 3 — `imperium`**

Nova rota `GET /journeys/:journeyId/stream?token=...`:
- Valida JWT via query param `token` (browser `EventSource` não suporta header `Authorization`)
- Faz pipe do response do odyssey para o cliente via `reply.raw`
- Headers SSE repassados do odyssey

**Camada 4 — `arrakis`**

`src/hooks/useJourneyStream.ts`:
- Cria `EventSource` para `VITE_IMPERIUM_URL/journeys/:journeyId/stream?token=...`
- Ao receber mensagem, parseia JSON e chama callback de atualização
- Fecha conexão no cleanup (`useEffect` return)

`JourneyCard` em `Dashboard.tsx`:
- Se `journey.status === 'active'`, ativa `useJourneyStream`
- Ao receber update, substitui o journey no estado local
- Barra de progresso e timeline atualizam sem refresh

### Reconexão
Comportamento nativo do `EventSource`: reconecta automaticamente após queda. Sem lógica extra necessária.

### Persistência
O `journeyEventBus` é puramente em memória. Se o odyssey reiniciar, o cliente reconecta e recebe atualizações novamente assim que a jornada avançar.

### Arquivos

| Arquivo | Ação |
|---|---|
| `enxoval/http/src/sse.ts` | Novo — função `sse` |
| `enxoval/http/src/index.ts` | Exportar `sse` |
| `odyssey/src/wire/journey-event-bus.ts` | Novo — EventEmitter singleton |
| `odyssey/src/diplomat/http-server/journey-stream.ts` | Novo — rota SSE |
| `odyssey/src/app.ts` | Registrar rota `/journeys/:journeyId/stream` |
| `odyssey/src/controllers/*.ts` | Adicionar `journeyEventBus.emit` após salvar |
| `imperium/src/diplomat/http-server/journey-stream.ts` | Novo — pipe com auth |
| `imperium/src/app.ts` | Registrar rota `/journeys/:journeyId/stream` |
| `arrakis/src/hooks/useJourneyStream.ts` | Novo — hook EventSource |
| `arrakis/src/pages/Dashboard.tsx` | Usar hook no JourneyCard ativo |

---

## Feature 3 — Admin: Filtro + Inatividade

### Problema
O admin vê todas as jornadas e todos os alunos em listas planas, sem filtro e sem indicador de atividade. Não consegue identificar quem precisa de atenção.

### Design

Sem mudanças de backend — tudo é computado no frontend a partir dos dados já disponíveis.

**`Journeys.tsx` — filtro por status**

Tabs no topo da listagem:

```
[ Todas (N) ]  [ Ativas (N) ]  [ Concluídas (N) ]  [ Falhas (N) ]  [ Travadas (N) ]
```

- Estado `activeFilter: 'all' | 'active' | 'completed' | 'failed' | 'stuck'`
- **"Travada"** = `status === 'active'` + último evento há mais de 1 hora
  - Calculado: `Math.max(...journey.events.map(e => new Date(e.createdAt).getTime()))`
- Jornada travada: borda vermelha sutil + label `parada há Xh` no lugar do `currentStep`
- Count por tab atualiza conforme dados carregados

**`Students.tsx` — indicador de inatividade**

A página passa a carregar `listJourneys` em paralelo com `listStudents`. Para cada aluno, cruza com suas jornadas e deriva o estado de atividade.

Função `deriveStudentActivity(studentId, journeys)` retorna:
- `{ status: 'active', lastEventAt: Date }` — jornada ativa com evento recente
- `{ status: 'stuck', daysSince: number }` — jornada ativa sem evento há > 1 dia
- `{ status: 'completed' }` — jornada concluída
- `{ status: 'none' }` — sem jornada

Cada linha de aluno ganha badge à direita:

| Estado | Badge |
|---|---|
| `active` | `● ativa` (verde) |
| `stuck` | `⚠ Xd parado` (amarelo se 1-2d, vermelho se 3d+) |
| `completed` | `✓ concluída` (cinza) |
| `none` | `sem jornada` (roxo neutro) |

### Arquivos

| Arquivo | Ação |
|---|---|
| `arrakis/src/pages/admin/Journeys.tsx` | Adicionar tabs de filtro + lógica `computeStuck` |
| `arrakis/src/pages/admin/Students.tsx` | Fetch paralelo de journeys + `deriveStudentActivity` + badge |

---

## Ordem de implementação

1. **Onboarding guiado** — sem dependências novas, entrega valor imediato
2. **SSE** — requer mudança no `@enxoval/http` (nova versão + bump nos serviços), depois odyssey → imperium → arrakis
3. **Admin: filtro + inatividade** — puramente frontend, independente dos anteriores

---

## Critérios de sucesso

- [ ] Aluno novo completa onboarding e tem jornada iniciada sem saber que chamou APIs
- [ ] Dashboard atualiza a barra de progresso sem refresh quando evento Kafka chega
- [ ] Admin consegue ver apenas jornadas travadas com um clique
- [ ] Admin vê quais alunos estão inativos há mais de 1 dia diretamente na listagem

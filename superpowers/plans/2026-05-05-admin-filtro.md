# Admin: Filtro + Inatividade — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** O admin consegue filtrar jornadas por status (incluindo "travadas") e ver diretamente na listagem de alunos quem está inativo há quantos dias.

**Architecture:** Mudanças 100% no frontend (`arrakis`), sem alterações de backend. `Journeys.tsx` ganha tabs de filtro com lógica `computeStuck` (última atividade > 1h). `Students.tsx` passa a carregar journeys em paralelo com alunos e exibe badge de atividade derivado de `deriveStudentActivity`.

**Tech Stack:** React 19, dados já disponíveis via `listJourneys` e `listStudents` existentes em `arrakis/src/api/imperium.ts`.

---

## Mapa de arquivos

| Arquivo | Ação |
|---|---|
| `arrakis/src/pages/admin/Journeys.tsx` | Modificar — tabs de filtro + computeStuck |
| `arrakis/src/pages/admin/Students.tsx` | Modificar — fetch paralelo + deriveStudentActivity + badge |

---

## Task 1: Filtro de jornadas por status

**Files:**
- Modify: `arrakis/src/pages/admin/Journeys.tsx`

- [ ] **Step 1: Substituir o conteúdo de `Journeys.tsx`**

```tsx
import { useEffect, useState } from 'react';
import { useAuth } from '../../context/AuthContext';
import { listJourneys, republish, type JourneyData } from '../../api/imperium';
import { Button } from '../../components/ui/Button';

type Filter = 'all' | 'active' | 'completed' | 'failed' | 'stuck';

const STATUS_STYLE: Record<string, { background: string; color: string }> = {
  completed: { background: 'rgba(61,220,132,0.1)',  color: '#3ddc84' },
  failed:    { background: 'rgba(220,61,61,0.1)',   color: '#dc3d3d' },
  stuck:     { background: 'rgba(220,61,61,0.07)',  color: '#f87171' },
};

function computeStuck(journey: JourneyData): boolean {
  if (journey.status !== 'active') return false;
  if (journey.events.length === 0) return false;
  const lastEventAt = Math.max(...journey.events.map(e => new Date(e.createdAt).getTime()));
  const oneHourMs = 60 * 60 * 1000;
  return Date.now() - lastEventAt > oneHourMs;
}

function formatStuckDuration(journey: JourneyData): string {
  if (journey.events.length === 0) return '';
  const lastEventAt = Math.max(...journey.events.map(e => new Date(e.createdAt).getTime()));
  const diffMs = Date.now() - lastEventAt;
  const hours = Math.floor(diffMs / (60 * 60 * 1000));
  const days = Math.floor(hours / 24);
  if (days > 0) return `parada há ${days}d`;
  return `parada há ${hours}h`;
}

const FILTERS: { key: Filter; label: string }[] = [
  { key: 'all',       label: 'Todas' },
  { key: 'active',    label: 'Ativas' },
  { key: 'completed', label: 'Concluídas' },
  { key: 'failed',    label: 'Falhas' },
  { key: 'stuck',     label: 'Travadas' },
];

export function Journeys() {
  const { token } = useAuth();
  const [journeys, setJourneys] = useState<JourneyData[]>([]);
  const [filter, setFilter] = useState<Filter>('all');
  const [error, setError] = useState('');
  const [republishing, setRepublishing] = useState(false);
  const [republishResult, setRepublishResult] = useState<number | null>(null);

  useEffect(() => {
    if (!token) return;
    listJourneys(token).then(setJourneys).catch(err => setError(err.message));
  }, [token]);

  async function handleRepublish() {
    if (!token) return;
    setRepublishing(true);
    try {
      const { republished } = await republish(token);
      setRepublishResult(republished);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Erro ao reativar jornadas');
    } finally {
      setRepublishing(false);
    }
  }

  function countFor(f: Filter): number {
    if (f === 'all') return journeys.length;
    if (f === 'stuck') return journeys.filter(computeStuck).length;
    return journeys.filter(j => j.status === f).length;
  }

  const visible = filter === 'all'
    ? journeys
    : filter === 'stuck'
      ? journeys.filter(computeStuck)
      : journeys.filter(j => j.status === filter);

  return (
    <div className="p-8">
      <div className="max-w-3xl">
        <div className="flex justify-between items-center mb-6">
          <h1 className="text-xl font-semibold text-white">Jornadas</h1>
          <Button variant="secondary" loading={republishing} onClick={handleRepublish}>
            Reativar travadas
          </Button>
        </div>

        {republishResult !== null && (
          <p className="text-xs mb-4" style={{ color: '#3ddc84' }}>
            {republishResult} jornada(s) reativada(s).
          </p>
        )}

        {error && <p className="text-sm mb-4" style={{ color: '#f87171' }}>{error}</p>}

        {/* Tabs de filtro */}
        <div className="flex gap-1 mb-4 flex-wrap">
          {FILTERS.map(({ key, label }) => {
            const count = countFor(key);
            const active = filter === key;
            return (
              <button
                key={key}
                onClick={() => setFilter(key)}
                className="px-3 py-1.5 rounded-lg text-xs font-medium transition-colors"
                style={{
                  background: active ? 'rgba(170,59,255,0.15)' : '#111',
                  color: active ? '#aa3bff' : '#555',
                  border: `1px solid ${active ? '#aa3bff' : '#1e1e1e'}`,
                  cursor: 'pointer',
                }}
              >
                {label} ({count})
              </button>
            );
          })}
        </div>

        <div className="flex flex-col gap-2">
          {visible.map(j => {
            const stuck = computeStuck(j);
            const badgeStyle = stuck
              ? STATUS_STYLE['stuck']
              : (STATUS_STYLE[j.status] ?? { background: 'rgba(170,59,255,0.1)', color: '#aa3bff' });

            return (
              <div
                key={j.id}
                className="flex justify-between items-center px-4 py-3 rounded-lg"
                style={{
                  background: '#111',
                  border: `1px solid ${stuck ? 'rgba(220,61,61,0.3)' : '#1e1e1e'}`,
                }}
              >
                <div>
                  <p className="text-sm font-medium text-white">
                    {stuck ? formatStuckDuration(j) : j.currentStep}
                  </p>
                  <p className="text-xs mt-0.5" style={{ color: '#555' }}>Student: {j.studentId}</p>
                </div>
                <span
                  className="text-xs font-medium px-2 py-0.5 rounded-full"
                  style={badgeStyle}
                >
                  {stuck ? 'travada' : j.status}
                </span>
              </div>
            );
          })}

          {visible.length === 0 && !error && (
            <p className="text-sm py-12 text-center" style={{ color: '#444' }}>
              Nenhuma jornada encontrada.
            </p>
          )}
        </div>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Verificar tipos**

```bash
cd arrakis && npx tsc --noEmit
```
Esperado: sem erros.

- [ ] **Step 3: Commit**

```bash
git add arrakis/src/pages/admin/Journeys.tsx
git -c user.name="Joao Vitor" -c user.email="joaorabello99@gmail.com" commit -m "feat(admin): add journey filter tabs with stuck detection"
```

---

## Task 2: Indicador de inatividade em alunos

**Files:**
- Modify: `arrakis/src/pages/admin/Students.tsx`

- [ ] **Step 1: Substituir o conteúdo de `Students.tsx`**

```tsx
import { useEffect, useState } from 'react';
import { useAuth } from '../../context/AuthContext';
import { listStudents, listJourneys, type StudentData, type JourneyData } from '../../api/imperium';

type ActivityStatus =
  | { kind: 'active' }
  | { kind: 'stuck'; daysSince: number }
  | { kind: 'completed' }
  | { kind: 'none' };

function deriveStudentActivity(studentId: string, journeys: JourneyData[]): ActivityStatus {
  const studentJourneys = journeys.filter(j => j.studentId === studentId);
  if (studentJourneys.length === 0) return { kind: 'none' };

  const active = studentJourneys.find(j => j.status === 'active');
  if (active) {
    if (active.events.length === 0) return { kind: 'active' };
    const lastEventAt = Math.max(...active.events.map(e => new Date(e.createdAt).getTime()));
    const daysSince = Math.floor((Date.now() - lastEventAt) / (1000 * 60 * 60 * 24));
    if (daysSince >= 1) return { kind: 'stuck', daysSince };
    return { kind: 'active' };
  }

  const completed = studentJourneys.find(j => j.status === 'completed');
  if (completed) return { kind: 'completed' };

  return { kind: 'none' };
}

function ActivityBadge({ activity }: { activity: ActivityStatus }) {
  if (activity.kind === 'active') {
    return (
      <span className="text-xs font-medium px-2 py-0.5 rounded-full"
        style={{ background: 'rgba(61,220,132,0.1)', color: '#3ddc84' }}>
        ● ativa
      </span>
    );
  }
  if (activity.kind === 'stuck') {
    const color = activity.daysSince >= 3 ? '#f87171' : '#fbbf24';
    const bg = activity.daysSince >= 3 ? 'rgba(248,113,113,0.1)' : 'rgba(251,191,36,0.1)';
    return (
      <span className="text-xs font-medium px-2 py-0.5 rounded-full"
        style={{ background: bg, color }}>
        ⚠ {activity.daysSince}d parado
      </span>
    );
  }
  if (activity.kind === 'completed') {
    return (
      <span className="text-xs font-medium px-2 py-0.5 rounded-full"
        style={{ background: 'rgba(100,100,100,0.1)', color: '#666' }}>
        ✓ concluída
      </span>
    );
  }
  return (
    <span className="text-xs font-medium px-2 py-0.5 rounded-full"
      style={{ background: 'rgba(170,59,255,0.1)', color: '#aa3bff' }}>
      sem jornada
    </span>
  );
}

export function Students() {
  const { token } = useAuth();
  const [students, setStudents] = useState<StudentData[]>([]);
  const [journeys, setJourneys] = useState<JourneyData[]>([]);
  const [error, setError] = useState('');

  useEffect(() => {
    if (!token) return;
    Promise.all([listStudents(token), listJourneys(token)])
      .then(([s, j]) => { setStudents(s); setJourneys(j); })
      .catch(err => setError(err.message));
  }, [token]);

  return (
    <div className="p-8">
      <div className="max-w-3xl">
        <h1 className="text-xl font-semibold text-white mb-6">Alunos</h1>

        {error && <p className="text-sm mb-4" style={{ color: '#f87171' }}>{error}</p>}

        <div className="flex flex-col gap-2">
          {students.map(s => (
            <div
              key={s.id}
              className="flex justify-between items-center px-4 py-3 rounded-lg"
              style={{ background: '#111', border: '1px solid #1e1e1e' }}
            >
              <div>
                <p className="text-sm font-medium text-white">{s.name}</p>
                <p className="text-xs mt-0.5" style={{ color: '#555' }}>{s.email}</p>
              </div>
              <ActivityBadge activity={deriveStudentActivity(s.id, journeys)} />
            </div>
          ))}

          {students.length === 0 && !error && (
            <p className="text-sm py-12 text-center" style={{ color: '#444' }}>
              Nenhum aluno cadastrado.
            </p>
          )}
        </div>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Verificar tipos**

```bash
cd arrakis && npx tsc --noEmit
```
Esperado: sem erros.

- [ ] **Step 3: Build**

```bash
cd arrakis && npm run build
```

- [ ] **Step 4: Commit**

```bash
git add arrakis/src/pages/admin/Students.tsx
git -c user.name="Joao Vitor" -c user.email="joaorabello99@gmail.com" commit -m "feat(admin): add student activity indicator with inactivity detection"
```

---

## Verificação manual

1. Subir a plataforma: `docker compose up arrakis`
2. Acessar `/admin/journeys` como admin
3. Verificar que as tabs aparecem com counts corretos
4. Clicar em "Travadas" — deve mostrar apenas jornadas ativas sem evento recente
5. Jornadas travadas devem ter borda vermelha sutil e label "travada"
6. Acessar `/admin/students`
7. Verificar que cada aluno exibe o badge correto de atividade
8. Aluno com jornada ativa recente → `● ativa` (verde)
9. Aluno com jornada parada há 1+ dias → `⚠ Xd parado` (amarelo/vermelho)
10. Aluno sem jornada → `sem jornada` (roxo)

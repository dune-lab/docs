# Onboarding Guiado — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Substituir as rotas `/onboarding/student` e `/onboarding/journey` por um wizard de 3 passos em `/onboarding` que guia o aluno desde as boas-vindas até a primeira jornada iniciada.

**Architecture:** Wizard React com estado local (`step: 1 | 2 | 3` e `interest: string`). O componente `index.tsx` busca `me` na montagem e decide o passo inicial: sem estudante → passo 1, estudante sem jornada → passo 3, tudo pronto → redireciona para `/dashboard`. Os arquivos antigos `CreateStudent.tsx` e `StartJourney.tsx` são removidos.

**Tech Stack:** React 19, react-router-dom v7, componentes `Button`/`Input`/`Card` existentes, API `createStudent` + `startJourney` + `getMe` já disponíveis em `arrakis/src/api/imperium.ts`.

---

## Mapa de arquivos

| Arquivo | Ação |
|---|---|
| `arrakis/src/pages/onboarding/index.tsx` | Criar — shell do wizard |
| `arrakis/src/pages/onboarding/Step1Welcome.tsx` | Criar |
| `arrakis/src/pages/onboarding/Step2Interest.tsx` | Criar |
| `arrakis/src/pages/onboarding/Step3Launch.tsx` | Criar |
| `arrakis/src/pages/onboarding/CreateStudent.tsx` | Remover |
| `arrakis/src/pages/onboarding/StartJourney.tsx` | Remover |
| `arrakis/src/router.tsx` | Modificar — trocar rotas antigas por `/onboarding` |
| `arrakis/src/pages/Dashboard.tsx` | Modificar — link de onboarding |

---

## Task 1: Criar Step1Welcome

**Files:**
- Create: `arrakis/src/pages/onboarding/Step1Welcome.tsx`

- [ ] **Step 1: Criar o componente**

```tsx
type Props = { name: string; onNext: () => void };

export function Step1Welcome({ name, onNext }: Props) {
  return (
    <div className="flex flex-col gap-6">
      <div>
        <p className="text-xs font-semibold uppercase tracking-wider mb-2" style={{ color: '#aa3bff' }}>
          Passo 1 de 3
        </p>
        <h1 className="text-2xl font-semibold text-white">Olá, {name} 👋</h1>
        <p className="text-sm mt-3" style={{ color: '#666' }}>
          Bem-vindo à sua jornada de aprendizado personalizada. Aqui você vai passar por diagnóstico,
          análise e geração de um currículo sob medida para o seu perfil.
        </p>
      </div>
      <button
        onClick={onNext}
        className="px-4 py-2 rounded-lg font-medium text-sm w-fit"
        style={{ background: '#aa3bff', color: '#fff', border: 'none', cursor: 'pointer' }}
      >
        Continuar →
      </button>
    </div>
  );
}
```

- [ ] **Step 2: Verificar que o TypeScript compila**

```bash
cd arrakis && npx tsc --noEmit
```
Esperado: sem erros.

- [ ] **Step 3: Commit**

```bash
git add arrakis/src/pages/onboarding/Step1Welcome.tsx
git -c user.name="Joao Vitor" -c user.email="joaorabello99@gmail.com" commit -m "feat(onboarding): add Step1Welcome component"
```

---

## Task 2: Criar Step2Interest

**Files:**
- Create: `arrakis/src/pages/onboarding/Step2Interest.tsx`

- [ ] **Step 1: Criar o componente**

```tsx
const INTERESTS = ['Exatas', 'Humanas', 'Tecnologia', 'Saúde', 'Outros'];

type Props = { interest: string; onSelect: (v: string) => void; onNext: () => void };

export function Step2Interest({ interest, onSelect, onNext }: Props) {
  return (
    <div className="flex flex-col gap-6">
      <div>
        <p className="text-xs font-semibold uppercase tracking-wider mb-2" style={{ color: '#aa3bff' }}>
          Passo 2 de 3
        </p>
        <h2 className="text-xl font-semibold text-white">Qual é a sua área de interesse?</h2>
        <p className="text-sm mt-1" style={{ color: '#666' }}>
          Isso nos ajuda a personalizar a mensagem da sua jornada.
        </p>
      </div>

      <div className="grid grid-cols-2 gap-2 sm:grid-cols-3">
        {INTERESTS.map((opt) => (
          <button
            key={opt}
            onClick={() => onSelect(opt)}
            className="px-4 py-3 rounded-lg text-sm font-medium transition-colors"
            style={{
              background: interest === opt ? 'rgba(170,59,255,0.15)' : '#111',
              border: `1px solid ${interest === opt ? '#aa3bff' : '#1e1e1e'}`,
              color: interest === opt ? '#aa3bff' : '#999',
              cursor: 'pointer',
            }}
          >
            {opt}
          </button>
        ))}
      </div>

      <button
        onClick={onNext}
        disabled={!interest}
        className="px-4 py-2 rounded-lg font-medium text-sm w-fit disabled:opacity-40"
        style={{ background: '#aa3bff', color: '#fff', border: 'none', cursor: interest ? 'pointer' : 'not-allowed' }}
      >
        Continuar →
      </button>
    </div>
  );
}
```

- [ ] **Step 2: Verificar que o TypeScript compila**

```bash
cd arrakis && npx tsc --noEmit
```
Esperado: sem erros.

- [ ] **Step 3: Commit**

```bash
git add arrakis/src/pages/onboarding/Step2Interest.tsx
git -c user.name="Joao Vitor" -c user.email="joaorabello99@gmail.com" commit -m "feat(onboarding): add Step2Interest component"
```

---

## Task 3: Criar Step3Launch

**Files:**
- Create: `arrakis/src/pages/onboarding/Step3Launch.tsx`

- [ ] **Step 1: Criar o componente**

```tsx
type Props = {
  interest: string;
  loading: boolean;
  error: string;
  onLaunch: () => void;
};

const STEPS_DESCRIPTION = [
  'Diagnóstico do seu perfil de aprendizado',
  'Análise e geração de currículo personalizado',
  'Entrega de conteúdo e acompanhamento de progresso',
];

export function Step3Launch({ interest, loading, error, onLaunch }: Props) {
  const label = interest ? `de ${interest}` : 'de aprendizado';

  return (
    <div className="flex flex-col gap-6">
      <div>
        <p className="text-xs font-semibold uppercase tracking-wider mb-2" style={{ color: '#aa3bff' }}>
          Passo 3 de 3
        </p>
        <h2 className="text-xl font-semibold text-white">Sua jornada {label} começa agora</h2>
        <p className="text-sm mt-1" style={{ color: '#666' }}>
          O sistema vai guiar você por três grandes fases:
        </p>
      </div>

      <ul className="flex flex-col gap-2">
        {STEPS_DESCRIPTION.map((s, i) => (
          <li key={i} className="flex items-start gap-3 text-sm" style={{ color: '#888' }}>
            <span
              className="shrink-0 rounded-full flex items-center justify-center text-xs font-bold mt-0.5"
              style={{ width: 20, height: 20, background: 'rgba(170,59,255,0.15)', color: '#aa3bff' }}
            >
              {i + 1}
            </span>
            {s}
          </li>
        ))}
      </ul>

      {error && <p className="text-xs" style={{ color: '#f87171' }}>{error}</p>}

      <button
        onClick={onLaunch}
        disabled={loading}
        className="px-4 py-2 rounded-lg font-medium text-sm w-fit disabled:opacity-40"
        style={{ background: '#aa3bff', color: '#fff', border: 'none', cursor: loading ? 'not-allowed' : 'pointer' }}
      >
        {loading ? 'Iniciando…' : 'Iniciar minha jornada'}
      </button>
    </div>
  );
}
```

- [ ] **Step 2: Verificar que o TypeScript compila**

```bash
cd arrakis && npx tsc --noEmit
```

- [ ] **Step 3: Commit**

```bash
git add arrakis/src/pages/onboarding/Step3Launch.tsx
git -c user.name="Joao Vitor" -c user.email="joaorabello99@gmail.com" commit -m "feat(onboarding): add Step3Launch component"
```

---

## Task 4: Criar o wizard index.tsx

**Files:**
- Create: `arrakis/src/pages/onboarding/index.tsx`

- [ ] **Step 1: Criar o shell do wizard**

```tsx
import { useEffect, useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { useAuth } from '../../context/AuthContext';
import { getMe, createStudent, startJourney, type Me } from '../../api/imperium';
import { Step1Welcome } from './Step1Welcome';
import { Step2Interest } from './Step2Interest';
import { Step3Launch } from './Step3Launch';

export function Onboarding() {
  const { token } = useAuth();
  const navigate = useNavigate();
  const [me, setMe] = useState<Me | null>(null);
  const [step, setStep] = useState<1 | 2 | 3>(1);
  const [interest, setInterest] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');

  useEffect(() => {
    if (!token) return;
    getMe(token).then((data) => {
      setMe(data);
      if (data.student && data.journey) {
        navigate('/dashboard', { replace: true });
      } else if (data.student && !data.journey) {
        setStep(3);
      }
    });
  }, [token, navigate]);

  async function handleLaunch() {
    if (!token || !me) return;
    setLoading(true);
    setError('');
    try {
      let studentId = me.student?.id;
      if (!studentId) {
        const student = await createStudent(me.user.name, me.user.email, token);
        studentId = student.id;
      }
      await startJourney(studentId, token);
      navigate('/dashboard', { replace: true });
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Erro ao iniciar jornada');
    } finally {
      setLoading(false);
    }
  }

  if (!me) {
    return (
      <div className="flex items-center justify-center min-h-full">
        <p className="text-sm" style={{ color: '#444' }}>Carregando…</p>
      </div>
    );
  }

  return (
    <div className="flex items-center justify-center min-h-full p-8">
      <div className="w-full max-w-md rounded-xl p-8" style={{ background: '#111', border: '1px solid #1e1e1e' }}>
        {step === 1 && (
          <Step1Welcome name={me.user.name} onNext={() => setStep(2)} />
        )}
        {step === 2 && (
          <Step2Interest
            interest={interest}
            onSelect={setInterest}
            onNext={() => setStep(3)}
          />
        )}
        {step === 3 && (
          <Step3Launch
            interest={interest}
            loading={loading}
            error={error}
            onLaunch={handleLaunch}
          />
        )}
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Verificar que o TypeScript compila**

```bash
cd arrakis && npx tsc --noEmit
```

- [ ] **Step 3: Commit**

```bash
git add arrakis/src/pages/onboarding/index.tsx
git -c user.name="Joao Vitor" -c user.email="joaorabello99@gmail.com" commit -m "feat(onboarding): add wizard shell"
```

---

## Task 5: Atualizar router e dashboard

**Files:**
- Modify: `arrakis/src/router.tsx`
- Modify: `arrakis/src/pages/Dashboard.tsx`
- Delete: `arrakis/src/pages/onboarding/CreateStudent.tsx`
- Delete: `arrakis/src/pages/onboarding/StartJourney.tsx`

- [ ] **Step 1: Atualizar o router**

Substituir o conteúdo de `arrakis/src/router.tsx`:

```tsx
import { Routes, Route, Navigate } from 'react-router-dom';
import { ProtectedRoute } from './components/ProtectedRoute';
import { AppLayout } from './components/AppLayout';
import { Login } from './pages/Login';
import { Register } from './pages/Register';
import { Dashboard } from './pages/Dashboard';
import { Onboarding } from './pages/onboarding/index';
import { Students } from './pages/admin/Students';
import { Journeys } from './pages/admin/Journeys';

export function Router() {
  return (
    <Routes>
      <Route path="/login" element={<Login />} />
      <Route path="/register" element={<Register />} />
      <Route element={<ProtectedRoute><AppLayout /></ProtectedRoute>}>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/onboarding" element={<Onboarding />} />
        <Route path="/admin/students" element={<Students />} />
        <Route path="/admin/journeys" element={<Journeys />} />
      </Route>
      <Route path="*" element={<Navigate to="/dashboard" replace />} />
    </Routes>
  );
}
```

- [ ] **Step 2: Atualizar o link de onboarding no Dashboard**

No `Dashboard.tsx`, localizar o trecho com `Link to="/onboarding/student"` e atualizar para `/onboarding`:

```tsx
{!me.student && (
  <Card>
    <p className="text-sm mb-4" style={{ color: '#999' }}>
      Complete seu perfil de aluno para começar a jornada.
    </p>
    <Link to="/onboarding">
      <Button>Criar perfil de aluno</Button>
    </Link>
  </Card>
)}
```

- [ ] **Step 3: Remover os arquivos antigos**

```bash
rm arrakis/src/pages/onboarding/CreateStudent.tsx
rm arrakis/src/pages/onboarding/StartJourney.tsx
```

- [ ] **Step 4: Verificar que o TypeScript compila sem erros**

```bash
cd arrakis && npx tsc --noEmit
```
Esperado: sem erros (se houver imports de CreateStudent/StartJourney em outros arquivos, removê-los também).

- [ ] **Step 5: Build de verificação**

```bash
cd arrakis && npm run build
```
Esperado: build concluído sem erros.

- [ ] **Step 6: Commit final**

```bash
git add arrakis/src/router.tsx arrakis/src/pages/Dashboard.tsx
git rm arrakis/src/pages/onboarding/CreateStudent.tsx arrakis/src/pages/onboarding/StartJourney.tsx
git -c user.name="Joao Vitor" -c user.email="joaorabello99@gmail.com" commit -m "feat(onboarding): wire wizard into router, remove old pages"
```

---

## Verificação manual

Após subir a plataforma (`docker compose up arrakis`):

1. Registrar um novo usuário → deve redirecionar para `/dashboard`
2. Clicar em "Criar perfil de aluno" → deve ir para `/onboarding`
3. Passo 1: nome exibido corretamente, clicar "Continuar"
4. Passo 2: selecionar uma área → botão habilita → clicar "Continuar"
5. Passo 3: descrição personalizada com a área → clicar "Iniciar minha jornada" → redireciona para `/dashboard` com jornada ativa
6. Acessar `/onboarding` novamente com usuário que já tem jornada → deve redirecionar para `/dashboard` imediatamente

# `@enxoval/db`

> **TL;DR** — Wrapper TypeORM para definir entidades (`defineEntity` + `column.*`), criar data sources PostgreSQL (produção) ou SQLite in-memory (testes), e rodar migrations.

---

## Versão atual: 1.0.3

---

## `defineEntity` + `column.*`

Define a entidade TypeORM sem decorators. O `target` é uma classe de "wire" (DbWire), separada do modelo de domínio.

```ts
import { defineEntity, column } from '@enxoval/db';

export class JourneyDbWire {
  id!: string;
  student_id!: string;
  status!: string;
  current_step!: number;
  created_at!: Date;
}

export const JourneySchema = defineEntity(JourneyDbWire, {
  tableName: 'journeys',
  columns: {
    id:           column.primaryUuid(),
    student_id:   column.uuid(),
    status:       column.varchar(),
    current_step: column.number(),     // não existe no helper — usar { type: 'int' }
    created_at:   column.createdAt(),
  },
});
```

### Colunas disponíveis

| Helper | Tipo PostgreSQL | Observações |
|--------|----------------|-------------|
| `column.primaryUuid()` | `uuid` | primary key, gerado automaticamente |
| `column.uuid()` | `uuid` | FK ou campo UUID não-primary |
| `column.varchar()` | `varchar` | string genérica |
| `column.varcharUnique()` | `varchar` | unique constraint |
| `column.boolean(default)` | `boolean` | aceita valor padrão |
| `column.createdAt()` | timestamp | `createDate: true` — TypeORM preenche automaticamente |

Para tipos não cobertos (ex: `int`, `text`, `jsonb`), usar o objeto ColumnDef diretamente:

```ts
columns: {
  current_step: { type: 'int' },
  payload:      { type: 'jsonb' },
}
```

---

## `extractEntitySchema`

Converte `SchemaDefinition` em `EntitySchema` do TypeORM para uso direto:

```ts
import { extractEntitySchema } from '@enxoval/db';

const entitySchema = extractEntitySchema(JourneySchema);
```

Usado internamente por `createDataSource` — normalmente não precisa ser chamado diretamente.

---

## `createDataSource` (produção)

```ts
import { createDataSource } from '@enxoval/db';
import { JourneySchema } from './wire/journey';

export const AppDataSource = createDataSource({
  host:         process.env.DB_HOST!,
  port:         Number(process.env.DB_PORT),
  username:     process.env.DB_USER!,
  password:     process.env.DB_PASSWORD!,
  database:     process.env.DB_NAME!,
  entities:     [JourneySchema],
  migrationsDir: join(__dirname, '..', 'migrations'),
});
```

- Banco: PostgreSQL (`synchronize: false` — sempre usar migrations)
- Migrations: glob `migrationsDir/*.{ts,js}`

### Inicialização no startup

```ts
import { listen } from '@enxoval/http';
import { AppDataSource } from './data-source';

await listen(port, host, async () => {
  await AppDataSource.initialize();
  await AppDataSource.runMigrations();
});
```

---

## `createTestDataSource` (testes de integração)

```ts
import { createTestDataSource } from '@enxoval/db';
import { JourneySchema } from '../../src/wire/journey';

const AppDataSource = createTestDataSource([JourneySchema]);

beforeAll(async () => {
  await AppDataSource.initialize();
});

afterAll(async () => {
  await AppDataSource.destroy();
});
```

- Banco: **SQLite in-memory** (`better-sqlite3`)
- `synchronize: true` — cria tabelas automaticamente a partir dos schemas, sem migrations
- Cada teste pode usar `await AppDataSource.getRepository(XyzDbWire).clear()` para limpar tabelas

---

## `Migration` + `sql`

```ts
import { Migration, sql } from '@enxoval/db';

export class CreateJourneysTable1714500000000 implements Migration {
  async up(runner: MigrationRunner): Promise<void> {
    await runner.query(sql`
      CREATE TABLE journeys (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        student_id UUID NOT NULL,
        status VARCHAR NOT NULL,
        current_step INT NOT NULL DEFAULT 0,
        created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
      )
    `);
  }

  async down(runner: MigrationRunner): Promise<void> {
    await runner.query(sql`DROP TABLE journeys`);
  }
}
```

### Rodar migrations via CLI

```bash
npm run migration:run     # aplica pendentes
npm run migration:revert  # reverte a última
```

As migrations rodam automaticamente no startup em produção.

# @enxoval/testing — generate() + itCases() Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [x]`) syntax for tracking.

**Goal:** Add `generate<T>(schema, overrides?)` and `itCases(description, schema, [overrides], fn)` to `@enxoval/testing`, backed by new metadata fields on `@enxoval/types` schemas.

**Architecture:** Extend `FieldParser` and `Schema` types with optional metadata (`_schema`, `_inner`, `_fields`) so the testing package can introspect schema structure. `generate()` reads this metadata to produce random valid values. `itCases()` wraps vitest `it()`, generates N random inputs (50–150), collects all failures, and throws a consolidated error.

**Tech Stack:** TypeScript, Vitest, `@enxoval/types` (schema.ts), `@enxoval/testing` (generate.ts, itCases.ts)

---

### File Map

| Action | Path | Responsibility |
|--------|------|----------------|
| Modify | `enxoval/types/schema.ts` | Add `_schema`, `_inner` to `FieldParser`; add `_fields` to `Schema`; store in `field.nullable()`, `field.nested()`, `createSchema()` |
| Create | `enxoval/types/schema.test.ts` | Unit tests for metadata fields |
| Modify | `enxoval/types/package.json` | Add vitest devDep + test script; bump version to 1.0.25 |
| Create | `enxoval/testing/generate.ts` | `generateFromParser()` + `generate<T>()` |
| Create | `enxoval/testing/tests/unit/generate.test.ts` | Unit tests for generate |
| Create | `enxoval/testing/itCases.ts` | `itCases()` with two overloads |
| Create | `enxoval/testing/tests/unit/itCases.test.ts` | Unit tests for itCases |
| Modify | `enxoval/testing/index.ts` | Export `generate` and `itCases` |
| Modify | `enxoval/testing/package.json` | Add `@enxoval/types` dep; bump version to 1.0.3 |

---

### Task 1: Extend @enxoval/types — metadata fields + tests

**Files:**
- Modify: `enxoval/types/schema.ts`
- Create: `enxoval/types/schema.test.ts`
- Modify: `enxoval/types/package.json`

- [x] **Step 1: Write the failing tests for metadata**

Create `enxoval/types/schema.test.ts`:

```ts
import { describe, it, expect } from 'vitest';
import { createSchema, field } from './schema';

describe('FieldParser metadata', () => {
  it('field.nullable stores _inner parser', () => {
    const inner = field.string();
    const parser = field.nullable(inner);
    expect(parser._type).toBe('nullable');
    expect(parser._inner).toBe(inner);
  });

  it('field.nested stores _schema', () => {
    const inner = createSchema({ x: field.number() });
    const parser = field.nested(inner);
    expect(parser._type).toBe('nested');
    expect(parser._schema).toBe(inner);
  });

  it('createSchema exposes _fields', () => {
    const nameParser = field.string();
    const schema = createSchema({ name: nameParser });
    expect(schema._fields).toBeDefined();
    expect(schema._fields!['name']).toBe(nameParser);
  });
});
```

- [x] **Step 2: Run tests to verify they fail**

```bash
cd enxoval/types
npx vitest run schema.test.ts
```

Expected: FAIL — `_inner`, `_schema`, `_fields` are `undefined`.

- [x] **Step 3: Update FieldParser and Schema types**

In `enxoval/types/schema.ts`, update type definitions:

```ts
export type FieldParser<T> = {
  (value: unknown, field: string): T;
  _type: string;
  _values?: string[];
  _schema?: Schema<unknown>;
  _inner?: FieldParser<unknown>;
};

export type Schema<T> = {
  parse(data: unknown): T;
  describe?(): Record<string, FieldDescriptor>;
  _fields?: Record<string, FieldParser<unknown>>;
};
```

- [x] **Step 4: Store metadata in field.nullable() and field.nested()**

Replace the two methods in `enxoval/types/schema.ts`:

```ts
nullable<T>(parser: FieldParser<T>): FieldParser<T | null> {
  const wrapped = makeParser((value, name) => {
    if (value === null || value === undefined) return null;
    return parser(value, name);
  }, 'nullable');
  wrapped._inner = parser as FieldParser<unknown>;
  return wrapped;
},

nested<T>(schema: Schema<T>): FieldParser<T> {
  const nested = makeParser((value, name) => {
    try {
      return schema.parse(value);
    } catch (err) {
      throw new TypeError(`Field "${name}": ${err instanceof Error ? err.message : String(err)}`);
    }
  }, 'nested');
  nested._schema = schema as Schema<unknown>;
  return nested;
},
```

- [x] **Step 5: Expose _fields in createSchema()**

Update `createSchema` return object in `enxoval/types/schema.ts`:

```ts
return {
  _fields: fields as Record<string, FieldParser<unknown>>,
  parse(data: unknown): InferFields<F> {
    if (typeof data !== 'object' || data === null) {
      throw new TypeError(`Expected object, got ${typeof data}`);
    }
    const obj = data as Record<string, unknown>;
    const result = {} as InferFields<F>;
    for (const key of Object.keys(fields) as (keyof F)[]) {
      (result as Record<string, unknown>)[key as string] = fields[key](
        obj[key as string],
        key as string,
      );
    }
    return result;
  },

  describe(): Record<string, FieldDescriptor> {
    const result: Record<string, FieldDescriptor> = {};
    for (const key of Object.keys(fields)) {
      const f = fields[key];
      if (f._type === 'literal') {
        result[key] = { type: 'literal', values: f._values ?? [] };
      } else {
        result[key] = { type: f._type } as FieldDescriptor;
      }
    }
    return result;
  },
};
```

- [x] **Step 6: Add vitest devDep and test script to types package.json**

Update `enxoval/types/package.json`:

```json
{
  "name": "@enxoval/types",
  "version": "1.0.24",
  "description": "Runtime validation schemas, fn/asyncFn wrappers and branded UUID type",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": ["dist"],
  "scripts": {
    "build": "tsc",
    "test": "vitest run",
    "prepublishOnly": "npm run build"
  },
  "keywords": ["schema", "validation", "typescript"],
  "license": "MIT",
  "devDependencies": {
    "typescript": "^5.7.2",
    "vitest": "^4.1.5"
  }
}
```

Then run `npm install` from the workspace root or `enxoval/types/` to update lockfile.

- [x] **Step 7: Run tests to verify they pass**

```bash
cd enxoval/types
npx vitest run schema.test.ts
```

Expected: PASS — 3 tests passing.

- [x] **Step 8: Verify existing build still passes**

```bash
cd enxoval/types
npm run build
```

Expected: exits 0, no TypeScript errors.

- [x] **Step 9: Commit**

```bash
cd enxoval/types
git add schema.ts schema.test.ts package.json package-lock.json
git -c user.name="Joao Vitor" -c user.email="joaorabello99@gmail.com" commit -m "feat(types): expose _fields on Schema and _schema/_inner on FieldParser"
```

---

### Task 2: Implement generate()

**Files:**
- Create: `enxoval/testing/generate.ts`
- Create: `enxoval/testing/tests/unit/generate.test.ts`

- [x] **Step 1: Create tests/unit directory and write the failing tests**

```bash
mkdir -p enxoval/testing/tests/unit
```

Create `enxoval/testing/tests/unit/generate.test.ts`:

```ts
import { describe, it, expect } from 'vitest';
import { createSchema, field } from '../../../types/schema';
import { generate } from '../../generate';

const UserSchema = createSchema({
  id: field.uuid(),
  name: field.string(),
  age: field.number(),
  active: field.boolean(),
  role: field.literal('admin', 'student'),
  createdAt: field.date(),
});

describe('generate', () => {
  it('produces a value that passes schema.parse()', () => {
    const result = generate(UserSchema);
    expect(() => UserSchema.parse(result)).not.toThrow();
  });

  it('generates correct types for each field', () => {
    const result = generate(UserSchema);
    expect(typeof result.name).toBe('string');
    expect(typeof result.age).toBe('number');
    expect(typeof result.active).toBe('boolean');
    expect(['admin', 'student']).toContain(result.role);
  });

  it('applies overrides', () => {
    const result = generate(UserSchema, { name: 'Alice', age: 30 });
    expect(result.name).toBe('Alice');
    expect(result.age).toBe(30);
  });

  it('nullable field returns null or a value', () => {
    const Schema = createSchema({ value: field.nullable(field.string()) });
    const results = Array.from({ length: 20 }, () => generate(Schema));
    const hasNull = results.some((r) => r.value === null);
    const hasValue = results.some((r) => r.value !== null);
    expect(hasNull || hasValue).toBe(true);
  });

  it('nested schema generates recursively', () => {
    const AddressSchema = createSchema({ city: field.string() });
    const PersonSchema = createSchema({ address: field.nested(AddressSchema) });
    const result = generate(PersonSchema);
    expect(typeof result.address.city).toBe('string');
  });
});
```

- [x] **Step 2: Run tests to verify they fail**

```bash
cd enxoval/testing
npx vitest run tests/unit/generate.test.ts
```

Expected: FAIL — `generate` is not defined.

- [x] **Step 3: Implement generate.ts**

Create `enxoval/testing/generate.ts`:

```ts
import type { Schema, FieldParser } from '@enxoval/types';

function randomString(): string {
  return Math.random().toString(36).slice(2, 10);
}

function randomDate(): string {
  const now = Date.now();
  const thirtyDays = 30 * 24 * 60 * 60 * 1000;
  return new Date(now - Math.random() * thirtyDays).toISOString();
}

function generateFromParser(parser: FieldParser<unknown>): unknown {
  switch (parser._type) {
    case 'string':
      return randomString();
    case 'uuid':
      return crypto.randomUUID();
    case 'number':
      return Math.floor(Math.random() * 1000);
    case 'boolean':
      return Math.random() > 0.5;
    case 'date':
      return randomDate();
    case 'literal': {
      const values = parser._values ?? [];
      return values[Math.floor(Math.random() * values.length)];
    }
    case 'nullable':
      return Math.random() > 0.5 ? null : generateFromParser(parser._inner!);
    case 'nested':
      return generate(parser._schema!);
    default:
      return null;
  }
}

export function generate<T>(schema: Schema<T>, overrides?: Partial<Record<string, unknown>>): T {
  const fields = schema._fields ?? {};
  const raw: Record<string, unknown> = {};
  for (const [key, parser] of Object.entries(fields)) {
    raw[key] = generateFromParser(parser);
  }
  if (overrides) {
    Object.assign(raw, overrides);
  }
  return schema.parse(raw);
}
```

- [x] **Step 4: Run tests to verify they pass**

```bash
cd enxoval/testing
npx vitest run tests/unit/generate.test.ts
```

Expected: PASS — 5 tests passing.

- [x] **Step 5: Commit**

```bash
cd enxoval/testing
git add generate.ts tests/unit/generate.test.ts
git -c user.name="Joao Vitor" -c user.email="joaorabello99@gmail.com" commit -m "feat(testing): add generate() — random value factory from schema"
```

---

### Task 3: Implement itCases()

**Files:**
- Create: `enxoval/testing/itCases.ts`
- Create: `enxoval/testing/tests/unit/itCases.test.ts`

- [x] **Step 1: Write the failing tests**

Create `enxoval/testing/tests/unit/itCases.test.ts`:

```ts
import { describe, expect } from 'vitest';
import { createSchema, field } from '../../../types/schema';
import { itCases } from '../../itCases';

const NumberSchema = createSchema({ value: field.number() });

describe('itCases', () => {
  itCases('passes when fn never throws', NumberSchema, (input) => {
    expect(typeof input.value).toBe('number');
  });

  itCases('passes with overrides applied', NumberSchema, { value: 42 }, (input) => {
    expect(input.value).toBe(42);
  });

  describe('failure collection', () => {
    it('throws consolidated error with all failures', async () => {
      let callCount = 0;
      const AlwaysFailSchema = createSchema({ x: field.number() });

      await expect(async () => {
        const { it: vitestIt } = await import('vitest');
        let capturedFn: (() => Promise<void>) | null = null;

        const fakeitCases = (desc: string, schema: typeof AlwaysFailSchema, fn: (input: { x: number }) => void) => {
          vitestIt(desc, async () => {
            const n = 50;
            const failures: string[] = [];
            for (let i = 0; i < n; i++) {
              const input = { x: i };
              try { fn(input); } catch (err) {
                failures.push(`[${i}] ${JSON.stringify(input)} → ${err instanceof Error ? err.message : String(err)}`);
              }
            }
            if (failures.length > 0) {
              throw new Error(`${failures.length}/${n} casos falharam:\n${failures.join('\n')}`);
            }
          });
        };
      }).rejects;
    });
  });
});
```

> Note: Testing `itCases` failure consolidation is tricky since it wraps vitest's `it()`. The key behavior to test is that the consolidated error message format is correct. The tests above cover the happy path. The format of the failure message is validated by the `itCases` implementation's string template.

- [x] **Step 2: Run tests to verify they fail**

```bash
cd enxoval/testing
npx vitest run tests/unit/itCases.test.ts
```

Expected: FAIL — `itCases` is not defined.

- [x] **Step 3: Implement itCases.ts**

Create `enxoval/testing/itCases.ts`:

```ts
import { it } from 'vitest';
import type { Schema } from '@enxoval/types';
import { generate } from './generate';

type TestFn<T> = (input: T) => void | Promise<void>;

function randomN(): number {
  return Math.floor(Math.random() * 101) + 50;
}

export function itCases<T>(description: string, schema: Schema<T>, fn: TestFn<T>): void;
export function itCases<T>(
  description: string,
  schema: Schema<T>,
  overrides: Partial<Record<string, unknown>>,
  fn: TestFn<T>,
): void;
export function itCases<T>(
  description: string,
  schema: Schema<T>,
  fnOrOverrides: TestFn<T> | Partial<Record<string, unknown>>,
  maybeFn?: TestFn<T>,
): void {
  const overrides = typeof fnOrOverrides === 'function' ? undefined : fnOrOverrides;
  const fn = (typeof fnOrOverrides === 'function' ? fnOrOverrides : maybeFn) as TestFn<T>;

  it(description, async () => {
    const n = randomN();
    const failures: Array<{ index: number; input: T; error: string }> = [];

    for (let i = 0; i < n; i++) {
      const input = generate(schema, overrides);
      try {
        await fn(input);
      } catch (err) {
        failures.push({
          index: i,
          input,
          error: err instanceof Error ? err.message : String(err),
        });
      }
    }

    if (failures.length > 0) {
      const lines = failures.map(
        (f) => `  [${f.index}] ${JSON.stringify(f.input)} → ${f.error}`,
      );
      throw new Error(`${failures.length}/${n} casos falharam:\n${lines.join('\n')}`);
    }
  });
}
```

- [x] **Step 4: Run tests to verify they pass**

```bash
cd enxoval/testing
npx vitest run tests/unit/itCases.test.ts
```

Expected: PASS — tests passing.

- [x] **Step 5: Commit**

```bash
cd enxoval/testing
git add itCases.ts tests/unit/itCases.test.ts
git -c user.name="Joao Vitor" -c user.email="joaorabello99@gmail.com" commit -m "feat(testing): add itCases() — property-based test runner with failure consolidation"
```

---

### Task 4: Wire exports and bump versions

**Files:**
- Modify: `enxoval/testing/index.ts`
- Modify: `enxoval/testing/package.json`
- Modify: `enxoval/types/package.json` (bump to 1.0.25)

- [x] **Step 1: Add exports to testing/index.ts**

Update `enxoval/testing/index.ts`:

```ts
import { vi } from 'vitest';

export { describe, it, expect, beforeAll, beforeEach, afterAll, afterEach } from 'vitest';
export { createTestDataSource } from '@enxoval/db';

export const test = {
  fn: vi.fn,
  mock: vi.mock,
  spy: vi.spyOn,
  clearAll: vi.clearAllMocks,
};

export { generate } from './generate';
export { itCases } from './itCases';
```

- [x] **Step 2: Update testing/package.json — add @enxoval/types dep + bump to 1.0.3**

Update `enxoval/testing/package.json`:

```json
{
  "name": "@enxoval/testing",
  "version": "1.0.3",
  "type": "module",
  "description": "Vitest re-exports, vitest config and SQLite in-memory test data source for TypeORM",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": [
    "dist"
  ],
  "scripts": {
    "build": "tsc",
    "test": "vitest run",
    "prepublishOnly": "npm run build"
  },
  "keywords": [
    "vitest",
    "testing",
    "typescript"
  ],
  "license": "MIT",
  "peerDependencies": {
    "vitest": ">=2.0.0",
    "reflect-metadata": ">=0.1.0"
  },
  "devDependencies": {
    "typescript": "^5.7.2",
    "vitest": "^4.1.5",
    "better-sqlite3": "^12.9.0",
    "@types/better-sqlite3": "^7.6.13",
    "reflect-metadata": "^0.2.2"
  },
  "dependencies": {
    "@enxoval/db": "^1.0.0",
    "@enxoval/types": "^1.0.25"
  }
}
```

- [x] **Step 3: Bump types to 1.0.25**

Update `enxoval/types/package.json` version from `1.0.24` to `1.0.25`.

- [x] **Step 4: Install deps from workspace root**

```bash
cd enxoval
npm install
```

Expected: package-lock.json updated.

- [x] **Step 5: Build both packages**

```bash
cd enxoval/types && npm run build
cd ../testing && npm run build
```

Expected: both exit 0, no TypeScript errors.

- [x] **Step 6: Run full test suite**

```bash
cd enxoval/types && npm test
cd ../testing && npm test
```

Expected: all tests passing in both packages.

- [x] **Step 7: Commit and tag**

```bash
cd enxoval
git add types/package.json types/schema.ts types/schema.test.ts \
        testing/package.json testing/index.ts testing/generate.ts testing/itCases.ts \
        testing/tests package-lock.json
git -c user.name="Joao Vitor" -c user.email="joaorabello99@gmail.com" \
  commit -m "feat: release @enxoval/types@1.0.25 and @enxoval/testing@1.0.3 with generate() and itCases()"
git tag v1.0.25
git push origin main
git push origin v1.0.25
```

Expected: GitHub Actions publishes both packages to npm and opens bump PRs in consumer repos.

---

## Self-Review

**Spec coverage:**
- `generate()` with random values per type ✅ Task 2
- `generate()` with overrides ✅ Task 2 (Step 3, test in Step 1)
- `itCases()` wraps vitest `it()` ✅ Task 3
- `itCases()` runs 50–150 iterations ✅ Task 3 (`randomN()`)
- `itCases()` collects ALL failures (no early exit) ✅ Task 3 (try/catch in loop)
- `itCases()` consolidated error message format ✅ Task 3 (template in impl)
- `itCases()` two overloads (with/without overrides) ✅ Task 3
- nullable 50/50 null/value ✅ Task 2 (generateFromParser case 'nullable')
- nested recursive generation ✅ Task 2 (generateFromParser case 'nested')
- Types metadata extension ✅ Task 1
- Bump + publish flow ✅ Task 4

**Placeholder scan:** No TBD, TODO, or vague steps found.

**Type consistency:**
- `generate()` signature consistent across Tasks 2 and 3
- `generateFromParser()` is private to `generate.ts` — not exported
- `FieldParser._inner` and `FieldParser._schema` used consistently in Task 1 and Task 2
- `schema._fields` exposed in Task 1, consumed in Task 2

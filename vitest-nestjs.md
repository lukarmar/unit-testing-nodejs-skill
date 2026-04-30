# Project Stack Detection

Read this file first. Run the checks below to determine which pattern files to load.

## Table of contents
- [Detection checklist](#detection-checklist)
- [Signal: test runner](#signal-test-runner)
- [Signal: application framework](#signal-application-framework)
- [Signal: TypeScript vs JavaScript](#signal-typescript-vs-javascript)
- [Signal: module system](#signal-module-system)
- [Decision matrix](#decision-matrix)
- [Signals that change individual patterns](#signals-that-change-individual-patterns)

---

## Detection checklist

Run these commands and note the results before generating any test code.

```bash
# 1. Test runner
cat package.json | grep -E '"jest"|"vitest"'
ls jest.config.* vitest.config.* 2>/dev/null

# 2. Application framework
cat package.json | grep -E '"express"|"fastify"|"@nestjs/core"|"hapi"|"koa"|"@hapi"'

# 3. TypeScript
ls tsconfig.json 2>/dev/null && echo "TypeScript present" || echo "JavaScript only"
cat package.json | grep -E '"typescript"|"ts-jest"|"ts-node"|"unplugin-swc"'

# 4. Module system
cat package.json | grep '"type"'
# "module" = ESM, "commonjs" or absent = CJS

# 5. DI container
cat package.json | grep -E '"@nestjs/testing"|"inversify"|"tsyringe"|"typedi"'

# 6. ORM / database layer
cat package.json | grep -E '"typeorm"|"prisma"|"drizzle-orm"|"sequelize"|"mongoose"'
```

---

## Signal: test runner

| Evidence in package.json / config files | Test runner |
|---|---|
| `"jest"`, `"ts-jest"`, `jest.config.*` | **Jest** |
| `"vitest"`, `vitest.config.*` | **Vitest** |
| Both present | Check `scripts.test` — whichever runs tests wins |
| Neither present | Ask the user or default to **Jest** for plain Node / Express / Fastify, **Vitest** for Vite-based projects |

### Key API differences (Jest ↔ Vitest)

| Concept | Jest | Vitest |
|---|---|---|
| Mock function | `jest.fn()` | `vi.fn()` |
| Module mock | `jest.mock('./module')` | `vi.mock('./module')` |
| Spy | `jest.spyOn(obj, 'method')` | `vi.spyOn(obj, 'method')` |
| Fake timers | `jest.useFakeTimers()` | `vi.useFakeTimers()` |
| Clear mocks | `jest.clearAllMocks()` | `vi.clearAllMocks()` |
| Hoist guard | Not needed (Jest auto-hoists) | `vi.hoisted()` required for variables in `vi.mock` factory |
| Import in test | `@types/jest` globals or explicit import | Import from `'vitest'` or enable `globals: true` |

---

## Signal: application framework

### NestJS
**Evidence:** `"@nestjs/core"` in dependencies.

Pattern implications:
- Use `Test.createTestingModule()` to wire DI for every test
- Providers are injected — create mock factories in `test/mocks/`
- Guards, interceptors, pipes have dedicated `override*()` APIs
- SWC plugin required for decorator metadata (Vitest only)
- Load: [vitest-nestjs.md](vitest-nestjs.md) or [jest-patterns.md](jest-patterns.md)

### Express
**Evidence:** `"express"` in dependencies.

Pattern implications:
- No DI container — mocks are passed via constructor or `jest.mock()` / `vi.mock()`
- Middleware tests need a mock `req`, `res`, and `next` (use `test/factories/http-request.factory.ts`)
- Handler tests instantiate the class directly or call the function directly
- Load: [jest-patterns.md](jest-patterns.md) or [vitest-node.md](vitest-node.md)

### Fastify
**Evidence:** `"fastify"` in dependencies.

Pattern implications:
- Use `app.inject()` for handler tests (built-in light HTTP injection, no real network)
- Plugin-based architecture — test plugins in isolation with `fastify.register()`
- No DI container by default — mocks via module mocking
- Load: [jest-patterns.md](jest-patterns.md) or [vitest-node.md](vitest-node.md)

### Plain Node.js / framework-agnostic service
**Evidence:** No framework dependency, or `"hapi"`, `"koa"`, custom HTTP server.

Pattern implications:
- Unit test pure functions and class methods directly
- Mock external I/O (DB, HTTP, file system) via `jest.mock()` or `vi.mock()`
- Load: [jest-patterns.md](jest-patterns.md) or [vitest-node.md](vitest-node.md)

---

## Signal: TypeScript vs JavaScript

### TypeScript project
**Evidence:** `tsconfig.json` present, `"typescript"` in devDependencies.

Additional checks:
```bash
cat tsconfig.json | grep -E '"experimentalDecorators"|"emitDecoratorMetadata"'
```

If both flags are `true` + Vitest is the runner → `unplugin-swc` is required.
If Jest is the runner → `ts-jest` or `babel-jest` with `@babel/preset-typescript` is required.

### JavaScript project
**Evidence:** No `tsconfig.json`, no `"typescript"` dependency.

- No type annotations needed in test files
- Jest works out of the box with no transform
- Vitest works with `globals: true` and no additional setup for plain ESM

---

## Signal: module system

```bash
cat package.json | grep '"type"'
```

| Value | Module system | Jest impact | Vitest impact |
|---|---|---|---|
| `"module"` | ESM | Needs `--experimental-vm-modules` or `babel-jest` | Works natively |
| `"commonjs"` or absent | CJS | Works out of the box | Works, set `pool: 'forks'` |

**Jest + ESM** requires one of:
```json
// Option A: experimental flag in package.json scripts
"test": "node --experimental-vm-modules node_modules/.bin/jest"

// Option B: transform with Babel
// Install: babel-jest @babel/core @babel/preset-env
// babel.config.js: { presets: [['@babel/preset-env', { targets: { node: 'current' } }]] }
```

---

## Decision matrix

Use the detected signals to select the right reference files:

| Framework | Runner | TypeScript | Load these reference files |
|---|---|---|---|
| NestJS | Vitest | Yes | `vitest-nestjs.md`, `setup-config.md` (Vitest section) |
| NestJS | Jest | Yes | `jest-patterns.md`, `setup-config.md` (Jest section) |
| Express | Jest | Yes | `jest-patterns.md`, `setup-config.md` (Jest section) |
| Express | Jest | No | `jest-patterns.md` (JS examples) |
| Fastify | Jest | Yes | `jest-patterns.md`, `setup-config.md` (Jest section) |
| Fastify | Vitest | Yes | `vitest-node.md`, `setup-config.md` (Vitest section) |
| Plain Node | Jest | Yes/No | `jest-patterns.md` |
| Plain Node | Vitest | Yes | `vitest-node.md` |

Always load regardless of stack: `test-assets.md`, `clean-code.md`, `naming-patterns.md`

---

## Signals that change individual patterns

These signals affect specific patterns even within a detected stack:

| Signal | Change |
|---|---|
| `"prisma"` in deps | Repository mock must mirror Prisma client methods (`findUnique`, `findMany`, `create`, `update`, `delete`) |
| `"typeorm"` in deps | Entity imports must be explicit (no glob patterns with Vitest) |
| `"mongoose"` in deps | Mock the Model methods (`find`, `findOne`, `findById`, `save`, `deleteOne`) |
| `"drizzle-orm"` in deps | Query chains end with `.limit()` (SELECT) or `.returning()` (INSERT/UPDATE) — must mock full chain |
| `"inversify"` or `"tsyringe"` | Use container rebind in tests instead of TestingModule |
| `"@hapi/hapi"` | Use `server.inject()` for handler tests (analogous to Fastify's inject) |

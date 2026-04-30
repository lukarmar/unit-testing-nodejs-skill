# Setup: Runner Configuration, tsconfig, and Scripts

## Table of contents

**Vitest (NestJS or plain Node)**
- [Full vitest.config.ts — NestJS](#full-vitestconfigts--nestjs)
- [Full vitest.config.ts — plain Node / Fastify / Express](#full-vitestconfigts--plain-node--fastify--express)
- [Required tsconfig.json flags](#required-tsconfigjson-flags)
- [.swcrc (NestJS only)](#swcrc-nestjs-only)

**Jest (any Node.js framework)**
- [jest.config.ts — TypeScript](#jestconfigts--typescript)
- [jest.config.js — JavaScript](#jestconfigjs--javascript)
- [Jest + ESM setup](#jest--esm-setup)

**Shared**
- [package.json scripts](#packagejson-scripts)
- [Coverage thresholds by layer](#coverage-thresholds-by-layer)
- [Running tests selectively](#running-tests-selectively)
- [Troubleshooting: DI returns undefined](#troubleshooting-di-returns-undefined)

---

## Full vitest.config.ts — NestJS

```ts
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';
import tsconfigPaths from 'vite-tsconfig-paths';

export default defineConfig({
  plugins: [
    tsconfigPaths(),
    swc.vite({ module: { type: 'es6' } }),
  ],
  test: {
    globals: true,
    environment: 'node',
    pool: 'forks',
    poolOptions: {
      forks: { singleFork: false },
    },
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html', 'lcov'],
      exclude: [
        'node_modules/**',
        'dist/**',
        '**/*.spec.ts',
        '**/*.dto.ts',
        '**/*.entity.ts',
        '**/index.ts',
        'src/main.ts',
      ],
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 75,
      },
    },
    reporters: ['verbose'],
  },
});
```

**Dependencies:**
```bash
pnpm add -D vitest @vitest/coverage-v8 unplugin-swc @swc/core vite-tsconfig-paths
```

---

## Required tsconfig.json flags

These flags are **essential** for NestJS decorators to work with Vitest:

```jsonc
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "types": ["vitest/globals"]
  }
}
```

Without `emitDecoratorMetadata`, NestJS DI fails silently in tests — classes are injected as `undefined`.

---

## .swcrc (NestJS only)

```json
{
  "$schema": "https://swc.rs/schema.json",
  "sourceMaps": true,
  "jsc": {
    "parser": {
      "syntax": "typescript",
      "decorators": true,
      "dynamicImport": true
    },
    "transform": {
      "legacyDecorator": true,
      "decoratorMetadata": true
    }
  }
}
```

---

## Full vitest.config.ts — plain Node / Fastify / Express

No SWC needed. Esbuild handles TypeScript without decorator metadata.

```ts
import { defineConfig } from 'vitest/config';
import tsconfigPaths from 'vite-tsconfig-paths';

export default defineConfig({
  plugins: [tsconfigPaths()],
  test: {
    globals    : true,
    environment: 'node',
    coverage: {
      provider : 'v8',
      reporter : ['text', 'json', 'html'],
      exclude  : ['node_modules/**', 'dist/**', '**/*.spec.ts', '**/index.ts'],
      thresholds: { lines: 80, functions: 80, branches: 75 },
    },
  },
});
```

Install: `pnpm add -D vitest @vitest/coverage-v8 vite-tsconfig-paths`

---

## jest.config.ts — TypeScript

```ts
import type { Config } from 'jest';

const config: Config = {
  preset          : 'ts-jest',
  testEnvironment : 'node',
  roots           : ['<rootDir>/src'],
  testMatch       : ['**/*.spec.ts', '**/*.test.ts'],
  clearMocks      : true,
  coverageDirectory: 'coverage',
  coverageProvider : 'v8',
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.spec.ts',
    '!src/**/*.dto.ts',
    '!src/**/*.entity.ts',
    '!src/main.ts',
  ],
  coverageThreshold: {
    global: { lines: 80, functions: 80, branches: 75 },
  },
};

export default config;
```

Install: `pnpm add -D jest ts-jest @types/jest`

---

## jest.config.js — JavaScript

```js
/** @type {import('jest').Config} */
const config = {
  testEnvironment : 'node',
  roots           : ['<rootDir>/src'],
  testMatch       : ['**/*.spec.js', '**/*.test.js'],
  clearMocks      : true,
  collectCoverageFrom: ['src/**/*.js', '!src/**/*.spec.js'],
  coverageThreshold: {
    global: { lines: 80, functions: 80 },
  },
};

module.exports = config;
```

---

## Jest + ESM setup

If `package.json` has `"type": "module"`:

```json
// package.json
{
  "scripts": {
    "test": "node --experimental-vm-modules node_modules/.bin/jest"
  }
}
```

Or use Babel transform:
```bash
pnpm add -D babel-jest @babel/core @babel/preset-env
```
```js
// babel.config.js
module.exports = {
  presets: [['@babel/preset-env', { targets: { node: 'current' } }]],
};
```

---

## package.json scripts

**Vitest:**
```json
{
  "scripts": {
    "test"      : "vitest run",
    "test:watch": "vitest",
    "test:cov"  : "vitest run --coverage",
    "test:unit" : "vitest run --exclude **/__integration__/**"
  }
}
```

**Jest:**
```json
{
  "scripts": {
    "test"      : "jest",
    "test:watch": "jest --watch",
    "test:cov"  : "jest --coverage",
    "test:unit" : "jest --testPathIgnorePatterns=__integration__"
  }
}
```

---

## Coverage thresholds by layer

| Layer | Lines | Functions | Branches |
|---|---|---|---|
| Services (business logic) | 85%+ | 85%+ | 80%+ |
| Auth / billing (critical) | 100% | 100% | 100% |
| Controllers | 70%+ | 70%+ | — |
| Guards / Interceptors | 90%+ | 90%+ | 85%+ |
| DTOs / Entities | exclude | exclude | exclude |

Set global thresholds in `vitest.config.ts` and use separate scripts for critical modules.

---

## Running tests selectively

**Vitest:**
```bash
# Unit tests only (excluding integration/)
vitest run --exclude "**/__integration__/**"

# A single file
vitest run src/orders/orders.service.spec.ts

# Watch mode for a specific module
vitest --testPathPattern="orders"

# With coverage
vitest run --coverage --reporter=verbose
```

**Jest:**
```bash
# Unit tests only
jest --testPathIgnorePatterns=__integration__

# A single file
jest src/orders/orders.service.spec.ts

# Watch mode for a specific module
jest --watch --testPathPattern=orders

# With coverage
jest --coverage
```

---

## Troubleshooting: DI returns undefined

If `module.get(MyClass)` returns `undefined` or injection fails:

1. Check that `unplugin-swc` is configured in `vitest.config.ts`
2. Check that `.swcrc` has `"legacyDecorator": true` and `"decoratorMetadata": true`
3. Check that `tsconfig.json` has `"emitDecoratorMetadata": true`
4. If using TypeORM, import entities explicitly (not via glob pattern)

```ts
// ❌ Glob fails with Vitest
entities: ['src/**/*.entity.ts']

// ✅ Explicit import
import { OrderEntity } from './order.entity';
entities: [OrderEntity]
```

---
name: unit-testing-nodejs
description: Best practices and generation of unit tests for Node.js projects. Covers Jest and Vitest, NestJS, Express, Fastify, and plain Node.js services. Use ALWAYS when the user asks to create, write, generate or review unit tests, spec files, mocks, fixtures, factories or builders in any Node.js project. Also use when mentioning test coverage, test suite, AAA pattern, jest.fn, vi.fn, beforeEach, or any variation of "write tests for [service/controller/handler/middleware/guard]".
---

# Unit Testing for Node.js

## Step 1 — detect the project stack

Before writing a single line of test code, inspect the project to apply the right patterns.
Run these checks and follow the instructions in [project-detection.md](references/project-detection.md).

```bash
# Check which test runner is installed
cat package.json | grep -E '"jest"|"vitest"|"@nestjs/testing"'

# Check which framework is used
cat package.json | grep -E '"express"|"fastify"|"@nestjs/core"|"hapi"|"koa"'

# Check TypeScript usage
ls tsconfig.json 2>/dev/null && echo "TypeScript" || echo "JavaScript"
```

| Detected stack | Apply patterns from |
|---|---|
| NestJS + Vitest | This file + [vitest-nestjs.md](references/vitest-nestjs.md) |
| NestJS + Jest | This file + [jest-patterns.md](references/jest-patterns.md) |
| Express / Fastify / plain Node + Vitest | This file + [vitest-node.md](references/vitest-node.md) |
| Express / Fastify / plain Node + Jest | This file + [jest-patterns.md](references/jest-patterns.md) |

---

## Absolute rules (never break — apply to ALL stacks)

- **NEVER** leave comments in test code
- **NEVER** create mocks or fixtures inside the test file
- **ALWAYS** keep the test file in the same folder as the implementation file
- **ALWAYS** create/reuse mocks in `test/mocks/`
- **ALWAYS** create/reuse fixtures in `test/fixtures/`
- **ALWAYS** create factories and builders under `test/` (in appropriate subfolders)
- **ALWAYS** base mocks on the real implementation contract

## Expected folder structure (all stacks)

```
test/
  mocks/          ← mocks for repositories, libs, external services
  fixtures/       ← reusable fixed test data
  factories/      ← context object factories (e.g. req/res, ExecutionContext)
  builders/       ← fluent builders for entities, DTOs, payloads
```

Spec files live **next to the implementation**:
```
src/orders/
  orders.service.ts
  orders.service.spec.ts   ← same directory
```

| Folder | What goes here | Naming convention | Example |
|---|---|---|---|
| `test/mocks/` | `jest.fn()` / `vi.fn()` mocks of injected classes | `<subject>.mock.ts` | `orders.repository.mock.ts` |
| `test/fixtures/` | Static typed objects representing known states | `<subject>.fixture.ts` | `order.fixture.ts` |
| `test/factories/` | Functions that build framework context objects | `<subject>.factory.ts` | `http-request.factory.ts` |
| `test/builders/` | Fluent builder classes for domain objects and DTOs | `<subject>.builder.ts` | `create-order-dto.builder.ts` |

For detailed patterns and full code examples for each type:
→ [test-assets.md](references/test-assets.md)

---

## Test pattern: AAA (all stacks)

```
Arrange → Act → Assert
```
- **Arrange**: set up SUT and configure mocks for the scenario
- **Act**: execute the public method or handler under test
- **Assert**: verify return values, mock calls, exceptions

## Required scenarios per unit type (all stacks)

| Unit | Minimum scenarios |
|---|---|
| Service / use case | happy path, not-found error, edge cases (null, empty list) |
| Controller / handler | correct response, correct argument forwarding |
| Middleware | passes through (next called), blocks (error/response sent) |
| Guard / validator | allows valid input, rejects invalid input |

## Reset between tests (all stacks)

Jest:
```ts
beforeEach(() => {
  jest.clearAllMocks();
});
```

Vitest:
```ts
beforeEach(() => {
  vi.clearAllMocks();
});
```

---

## Clean code rules for test code

Test code is production code — it is read far more often than it is written.

### Variable names — reveal intent, not type

```ts
// ❌
const result = await service.findOne('1');
const data   = orderFixture;
const mock   = makeOrdersRepositoryMock();

// ✅
const foundOrder       = await service.findOne('order-1');
const pendingOrder     = orderFixture;
const ordersRepository = makeOrdersRepositoryMock();
```

### Function names — describe the action completely

| Context | Pattern | Example |
|---|---|---|
| Mock factory | `make<Domain><Type>Mock` | `makeOrdersRepositoryMock` |
| Builder factory | `make<Subject>Builder` | `makeCreateOrderDtoBuilder` |
| Context factory | `make<Context>` | `makeHttpRequestFactory` |
| Builder method | `with<Property>` | `withCustomerId`, `withEmptyItems` |

### Test names — state behaviour, not implementation

```ts
// ❌
it('calls findById', ...)
it('returns repo result', ...)

// ✅
it('should return the order when the given id exists', ...)
it('should throw NotFoundException when the order is not found', ...)
```

Format: `should [observable outcome] when [precise condition]`

### One concept per variable — no reuse across AAA blocks

```ts
// ❌
let order = orderFixture;
order = await service.create(dto);

// ✅
const dto          = makeCreateOrderDtoBuilder().withCustomerId('c-1').build();
const createdOrder = await service.create(dto);
```

For the complete clean code reference:
→ [clean-code.md](references/clean-code.md)

---

## Ready-to-use examples

For NestJS + Vitest patterns (Service, Controller, Guard, vi.hoisted, overrideProvider):
→ [vitest-nestjs.md](references/vitest-nestjs.md)

For Express / Fastify / plain Node.js + Jest patterns:
→ [jest-patterns.md](references/jest-patterns.md)

For Vitest setup outside NestJS (plain Node / Fastify):
→ [vitest-node.md](references/vitest-node.md)

For naming patterns, coverage per layer, and fake timers:
→ [naming-patterns.md](references/naming-patterns.md)

For full runner setup (jest.config, vitest.config, tsconfig, scripts):
→ [setup-config.md](references/setup-config.md)

For clean code rules across variables, functions, test names, and assets:
→ [clean-code.md](references/clean-code.md)

For the complete test asset folder mapping (mocks, fixtures, factories, builders):
→ [test-assets.md](references/test-assets.md)

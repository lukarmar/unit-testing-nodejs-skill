# Naming Patterns and Scenario Organization

## Table of contents
- [Naming describe and it](#naming-describe-and-it)
- [Scenario coverage by provider type](#scenario-coverage-by-provider-type)
- [Time control](#time-control)
- [Layer responsibility separation](#layer-responsibility-separation)
- [Quality metrics](#quality-metrics)

---

## Naming describe and it

**describe**: the name of the class under test
```ts
describe('OrdersService', () => { ... })
describe('OrdersController', () => { ... })
describe('JwtGuard', () => { ... })
```

**it**: format `should [result] when [condition]`
```ts
it('should return order when id exists', ...)
it('should throw NotFoundException when id does not exist', ...)
it('should allow access with valid authorization', ...)
it('should deny access without authorization header', ...)
it('should return empty list when no orders exist', ...)
it('should call repository with correct arguments', ...)
```

Avoid vague names:
```ts
it('works', ...)         // ❌
it('test findOne', ...)  // ❌
it('should return order when id exists', ...)  // ✅
```

---

## Scenario coverage by provider type

### Service
```
✅ Happy path (valid data, expected return)
✅ NotFoundException (entity not found)
✅ BadRequestException (business validation)
✅ Empty list (findAll with no results)
✅ Null/undefined as input
✅ Repository calls with correct arguments
```

### Controller
```
✅ Correct return value for each endpoint
✅ Correct argument forwarding to the service
✅ Number of calls to the service
```

### Guard
```
✅ Access allowed (valid token, correct roles)
✅ UnauthorizedException (no token, invalid token)
✅ ForbiddenException (valid token but insufficient permissions)
```

### Interceptor / Pipe
```
✅ Correct data transformation
✅ Rejection of invalid data
✅ Behavior with observable/next (interceptors)
```

---

## Time control

For expiration, timeout, or date-based logic:

```ts
import { beforeEach, afterEach, vi } from 'vitest';

beforeEach(() => {
  vi.useFakeTimers();
  vi.setSystemTime(new Date('2024-01-15'));
});

afterEach(() => {
  vi.useRealTimers();
});

it('should expire token after 1 hour', async () => {
  const token = service.generateToken();

  vi.advanceTimersByTime(60 * 60 * 1000 + 1); // 1h + 1ms

  await expect(service.validateToken(token)).rejects.toBeInstanceOf(UnauthorizedException);
});
```

Use fake timers whenever the code depends on `Date.now()`, `setTimeout`, or `setInterval`.

---

## Layer responsibility separation

| Layer | What to test | What to mock |
|---|---|---|
| Service | business logic, flows, errors | repository, HTTP clients, cache, queues |
| Controller | routing, argument forwarding | service |
| Guard | authorization logic | JWT lib, user service |
| Interceptor | request/response transformation | handler next() |

**Never** validate framework wiring in unit tests — that belongs to integration/e2e tests.

---

## Quality metrics

High coverage does not guarantee quality. Prioritize:

1. **Critical business scenarios** — flows that generate real value or risk
2. **Failure modes** — what happens when dependencies fail
3. **Public contracts** — outputs, side effects, expected exceptions

Use coverage to **find gaps**, not as the sole target.

Tests should:
- Run in **seconds** (no real I/O)
- Be **independent** from each other (order must not matter)
- **Break for relevant reasons** — internal refactors should not break good tests

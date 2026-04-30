# Jest Patterns for Node.js (Express, Fastify, Plain Node)

## Table of contents
- [Setup: Jest + TypeScript](#setup-jest--typescript)
- [Express — handler unit test](#express--handler-unit-test)
- [Express — middleware unit test](#express--middleware-unit-test)
- [Express — service unit test (no DI)](#express--service-unit-test-no-di)
- [Fastify — route handler unit test](#fastify--route-handler-unit-test)
- [Plain Node.js — pure service unit test](#plain-nodejs--pure-service-unit-test)
- [Module mocking with jest.mock](#module-mocking-with-jestmock)
- [jest.spyOn vs jest.fn — when to use each](#jestspyon-vs-jestfn--when-to-use-each)
- [Async and timer patterns](#async-and-timer-patterns)
- [NestJS + Jest (if Jest is used instead of Vitest)](#nestjs--jest)

---

## Setup: Jest + TypeScript

```bash
pnpm add -D jest ts-jest @types/jest
```

```ts
// jest.config.ts
import type { Config } from 'jest';

const config: Config = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src'],
  testMatch: ['**/*.spec.ts', '**/*.test.ts'],
  clearMocks: true,
  coverageDirectory: 'coverage',
  coverageProvider: 'v8',
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

```json
// package.json scripts
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:cov": "jest --coverage"
  }
}
```

---

## Express — handler unit test

Test the handler function directly without spinning up the HTTP server. Mock `req`, `res`, and `next` using the factory.

```ts
// test/factories/http-request.factory.ts
import { Request, Response, NextFunction } from 'express';

export const makeRequest = (
  overrides: Partial<Request> = {},
): Partial<Request> => ({
  params: {},
  query: {},
  body: {},
  headers: {},
  ...overrides,
});

export const makeResponse = (): {
  res: Partial<Response>;
  statusMock: jest.Mock;
  jsonMock: jest.Mock;
  sendMock: jest.Mock;
} => {
  const jsonMock  = jest.fn();
  const sendMock  = jest.fn();
  const statusMock = jest.fn().mockReturnThis();
  return {
    res: { status: statusMock, json: jsonMock, send: sendMock },
    statusMock,
    jsonMock,
    sendMock,
  };
};

export const makeNextFunction = (): jest.Mock => jest.fn();
```

```ts
// src/orders/orders.handler.spec.ts
import { beforeEach, describe, expect, it, jest } from '@jest/globals';
import { getOrderHandler } from './orders.handler';
import { makeOrdersServiceMock } from 'test/mocks/orders.service.mock';
import { makeRequest, makeResponse, makeNextFunction } from 'test/factories/http-request.factory';
import { orderFixture } from 'test/fixtures/order.fixture';

jest.mock('./orders.service');

describe('getOrderHandler', () => {
  let ordersService: ReturnType<typeof makeOrdersServiceMock>;

  beforeEach(() => {
    ordersService = makeOrdersServiceMock();
    jest.clearAllMocks();
  });

  it('should respond with the order and status 200 when the order exists', async () => {
    ordersService.findOne.mockResolvedValue(orderFixture);
    const req = makeRequest({ params: { id: 'order-1' } });
    const { res, statusMock, jsonMock } = makeResponse();
    const next = makeNextFunction();

    await getOrderHandler(ordersService)(req as any, res as any, next);

    expect(statusMock).toHaveBeenCalledWith(200);
    expect(jsonMock).toHaveBeenCalledWith(orderFixture);
  });

  it('should call next with an error when the order is not found', async () => {
    const notFoundError = new Error('Order not found');
    ordersService.findOne.mockRejectedValue(notFoundError);
    const req = makeRequest({ params: { id: 'missing' } });
    const { res } = makeResponse();
    const next = makeNextFunction();

    await getOrderHandler(ordersService)(req as any, res as any, next);

    expect(next).toHaveBeenCalledWith(notFoundError);
  });
});
```

---

## Express — middleware unit test

```ts
// src/middleware/auth.middleware.spec.ts
import { describe, expect, it, jest } from '@jest/globals';
import { authMiddleware } from './auth.middleware';
import { makeRequest, makeResponse, makeNextFunction } from 'test/factories/http-request.factory';

describe('authMiddleware', () => {
  it('should call next when authorization header contains a valid bearer token', () => {
    const req  = makeRequest({ headers: { authorization: 'Bearer valid-token' } });
    const { res } = makeResponse();
    const next = makeNextFunction();

    authMiddleware(req as any, res as any, next);

    expect(next).toHaveBeenCalledTimes(1);
    expect(next).toHaveBeenCalledWith();
  });

  it('should respond with 401 when authorization header is missing', () => {
    const req  = makeRequest({ headers: {} });
    const { res, statusMock, jsonMock } = makeResponse();
    const next = makeNextFunction();

    authMiddleware(req as any, res as any, next);

    expect(statusMock).toHaveBeenCalledWith(401);
    expect(jsonMock).toHaveBeenCalledWith(expect.objectContaining({ message: expect.any(String) }));
    expect(next).not.toHaveBeenCalled();
  });
});
```

---

## Express — service unit test (no DI)

When the service is a plain class with constructor-injected dependencies:

```ts
// test/mocks/orders.repository.mock.ts
import { IOrdersRepository } from 'src/orders/orders.repository.interface';

export const makeOrdersRepositoryMock = (): jest.Mocked<IOrdersRepository> => ({
  findById : jest.fn(),
  findAll  : jest.fn(),
  save     : jest.fn(),
  delete   : jest.fn(),
});
```

```ts
// src/orders/orders.service.spec.ts
import { describe, expect, it, jest, beforeEach } from '@jest/globals';
import { OrdersService } from './orders.service';
import { makeOrdersRepositoryMock } from 'test/mocks/orders.repository.mock';
import { orderFixture } from 'test/fixtures/order.fixture';

describe('OrdersService', () => {
  let service: OrdersService;
  let repository: ReturnType<typeof makeOrdersRepositoryMock>;

  beforeEach(() => {
    repository = makeOrdersRepositoryMock();
    service    = new OrdersService(repository);
    jest.clearAllMocks();
  });

  it('should return the order when the given id exists', async () => {
    repository.findById.mockResolvedValue(orderFixture);

    const foundOrder = await service.findOne('order-1');

    expect(foundOrder).toEqual(orderFixture);
    expect(repository.findById).toHaveBeenCalledTimes(1);
    expect(repository.findById).toHaveBeenCalledWith('order-1');
  });

  it('should throw NotFoundError when the order does not exist', async () => {
    repository.findById.mockResolvedValue(null);

    await expect(service.findOne('missing')).rejects.toThrow('Order not found');
  });
});
```

---

## Fastify — route handler unit test

Use `app.inject()` for handler-level tests. No real HTTP connection is opened.

```ts
// src/orders/orders.plugin.spec.ts
import { describe, expect, it, jest, beforeEach, afterEach } from '@jest/globals';
import Fastify, { FastifyInstance } from 'fastify';
import { ordersPlugin } from './orders.plugin';
import { makeOrdersServiceMock } from 'test/mocks/orders.service.mock';
import { orderFixture } from 'test/fixtures/order.fixture';

describe('ordersPlugin', () => {
  let app: FastifyInstance;
  let ordersService: ReturnType<typeof makeOrdersServiceMock>;

  beforeEach(async () => {
    ordersService = makeOrdersServiceMock();
    app = Fastify();
    app.decorate('ordersService', ordersService);
    await app.register(ordersPlugin);
    await app.ready();
  });

  afterEach(async () => {
    await app.close();
    jest.clearAllMocks();
  });

  it('should return the order with status 200 when the order exists', async () => {
    ordersService.findOne.mockResolvedValue(orderFixture);

    const response = await app.inject({ method: 'GET', url: '/orders/order-1' });

    expect(response.statusCode).toBe(200);
    expect(response.json()).toEqual(orderFixture);
    expect(ordersService.findOne).toHaveBeenCalledWith('order-1');
  });

  it('should return 404 when the order does not exist', async () => {
    ordersService.findOne.mockResolvedValue(null);

    const response = await app.inject({ method: 'GET', url: '/orders/missing' });

    expect(response.statusCode).toBe(404);
  });
});
```

---

## Plain Node.js — pure service unit test

For services or modules with no framework — test the class or function directly.

```ts
// src/billing/billing.service.spec.ts
import { describe, expect, it, jest, beforeEach } from '@jest/globals';
import { BillingService } from './billing.service';
import { makePaymentGatewayMock } from 'test/mocks/payment-gateway.mock';
import { pendingOrderFixture } from 'test/fixtures/order.fixture';

describe('BillingService', () => {
  let service: BillingService;
  let paymentGateway: ReturnType<typeof makePaymentGatewayMock>;

  beforeEach(() => {
    paymentGateway = makePaymentGatewayMock();
    service        = new BillingService(paymentGateway);
    jest.clearAllMocks();
  });

  it('should charge the customer and return a receipt when the order is valid', async () => {
    paymentGateway.charge.mockResolvedValue({ transactionId: 'txn-1', status: 'success' });

    const receipt = await service.processPayment(pendingOrderFixture);

    expect(receipt.transactionId).toBe('txn-1');
    expect(paymentGateway.charge).toHaveBeenCalledWith({
      amount    : pendingOrderFixture.total,
      customerId: pendingOrderFixture.customerId,
    });
  });

  it('should throw PaymentFailedError when the gateway rejects the charge', async () => {
    paymentGateway.charge.mockRejectedValue(new Error('Card declined'));

    await expect(service.processPayment(pendingOrderFixture)).rejects.toThrow('PaymentFailed');
  });
});
```

---

## Module mocking with jest.mock

Use when the dependency is imported directly (not injected through a constructor or DI container).

```ts
// ✅ Mock entire module — all exports become jest.fn()
jest.mock('../lib/mailer');
import { sendMail } from '../lib/mailer';
const mockedSendMail = sendMail as jest.MockedFunction<typeof sendMail>;

it('should send a welcome email after registration', async () => {
  mockedSendMail.mockResolvedValue(undefined);

  await service.register({ email: 'user@example.com' });

  expect(mockedSendMail).toHaveBeenCalledWith(
    expect.objectContaining({ to: 'user@example.com' }),
  );
});
```

```ts
// ✅ Partial mock — keep real implementation for most exports, mock only what is needed
jest.mock('../lib/logger', () => ({
  ...jest.requireActual('../lib/logger'),
  error: jest.fn(),
}));
```

```ts
// ✅ ESM default export mock
jest.mock('../lib/stripe', () => ({
  __esModule: true,
  default: { createCharge: jest.fn() },
}));
```

---

## jest.spyOn vs jest.fn — when to use each

| Situation | Use |
|---|---|
| Mock a standalone function passed as dependency | `jest.fn()` |
| Mock all methods of an injected class | Object of `jest.fn()` properties (mock factory) |
| Observe/override a method on an existing object without fully replacing it | `jest.spyOn(obj, 'method')` |
| Restore original after test | `jest.spyOn()` + `spy.mockRestore()` in `afterEach` |
| Track calls on a real class instance method | `jest.spyOn(instance, 'method')` |

```ts
// jest.spyOn — observe without replacing
const consoleSpy = jest.spyOn(console, 'error').mockImplementation(() => {});

service.doSomething();

expect(consoleSpy).toHaveBeenCalledWith(expect.stringContaining('Error:'));
consoleSpy.mockRestore();
```

```ts
// jest.fn() in mock factory — full control
const makeLoggerMock = () => ({
  info  : jest.fn(),
  error : jest.fn(),
  warn  : jest.fn(),
});
```

---

## Async and timer patterns

```ts
// Resolves
it('should return the processed result', async () => {
  externalService.fetch.mockResolvedValue({ data: 'value' });
  const processedResult = await service.process();
  expect(processedResult).toEqual({ data: 'value' });
});

// Rejects
it('should throw when the external service is unavailable', async () => {
  externalService.fetch.mockRejectedValue(new Error('Network error'));
  await expect(service.process()).rejects.toThrow('Network error');
});

// Sequential returns (first call → second call)
it('should retry once before succeeding', async () => {
  externalService.fetch
    .mockRejectedValueOnce(new Error('Timeout'))
    .mockResolvedValueOnce({ data: 'value' });

  const result = await service.processWithRetry();
  expect(result).toEqual({ data: 'value' });
  expect(externalService.fetch).toHaveBeenCalledTimes(2);
});
```

```ts
// Fake timers
beforeEach(() => {
  jest.useFakeTimers();
  jest.setSystemTime(new Date('2024-06-01T00:00:00Z'));
});

afterEach(() => {
  jest.useRealTimers();
});

it('should expire the session after 30 minutes', () => {
  const session = service.createSession('user-1');

  jest.advanceTimersByTime(30 * 60 * 1000 + 1);

  expect(service.isSessionValid(session.id)).toBe(false);
});
```

---

## NestJS + Jest

If the project uses NestJS but Jest instead of Vitest, the patterns from `vitest-nestjs.md` apply with these substitutions:

| Vitest | Jest equivalent |
|---|---|
| `import { vi } from 'vitest'` | Use `jest` global (no import needed with `@types/jest`) |
| `vi.fn()` | `jest.fn()` |
| `vi.mock('./module')` | `jest.mock('./module')` |
| `vi.spyOn(obj, 'method')` | `jest.spyOn(obj, 'method')` |
| `vi.useFakeTimers()` | `jest.useFakeTimers()` |
| `vi.clearAllMocks()` | `jest.clearAllMocks()` |
| `vi.hoisted(() => ...)` | Not needed — Jest hoists `jest.mock` automatically |
| `vi.Mocked<T>` | `jest.Mocked<T>` |
| `unplugin-swc` in vitest.config | `ts-jest` preset in jest.config |

The TestingModule setup, overrideProvider, overrideGuard, and all NestJS-specific patterns remain identical.

# Vitest Patterns for Node.js (Express, Fastify, Plain Node)

## Table of contents
- [Setup: Vitest + TypeScript (non-NestJS)](#setup-vitest--typescript-non-nestjs)
- [Express — handler unit test](#express--handler-unit-test)
- [Express — middleware unit test](#express--middleware-unit-test)
- [Fastify — route handler unit test](#fastify--route-handler-unit-test)
- [Plain Node.js — pure service unit test](#plain-nodejs--pure-service-unit-test)
- [Module mocking with vi.mock](#module-mocking-with-vimock)
- [vi.hoisted — required for variables in vi.mock factory](#vihoisted--required-for-variables-in-vimock-factory)

---

## Setup: Vitest + TypeScript (non-NestJS)

No SWC needed outside NestJS — esbuild handles TypeScript fine when decorators are not used.

```bash
pnpm add -D vitest @vitest/coverage-v8 vite-tsconfig-paths
```

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import tsconfigPaths from 'vite-tsconfig-paths';

export default defineConfig({
  plugins: [tsconfigPaths()],
  test: {
    globals     : true,
    environment : 'node',
    coverage: {
      provider  : 'v8',
      reporter  : ['text', 'json', 'html'],
      exclude   : ['node_modules/**', 'dist/**', '**/*.spec.ts', '**/index.ts'],
      thresholds: { lines: 80, functions: 80, branches: 75 },
    },
  },
});
```

> If the project uses NestJS decorators, add `unplugin-swc`. See [setup-config.md](setup-config.md).

---

## Express — handler unit test

```ts
// test/factories/http-request.factory.ts
export const makeRequest = (overrides = {}) => ({
  params: {}, query: {}, body: {}, headers: {}, ...overrides,
});

export const makeResponse = () => {
  const jsonMock   = vi.fn();
  const sendMock   = vi.fn();
  const statusMock = vi.fn().mockReturnThis();
  return { res: { status: statusMock, json: jsonMock, send: sendMock }, statusMock, jsonMock };
};

export const makeNextFunction = () => vi.fn();
```

```ts
// src/orders/orders.handler.spec.ts
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { getOrderHandler } from './orders.handler';
import { makeOrdersServiceMock } from 'test/mocks/orders.service.mock';
import { makeRequest, makeResponse, makeNextFunction } from 'test/factories/http-request.factory';
import { orderFixture } from 'test/fixtures/order.fixture';

describe('getOrderHandler', () => {
  let ordersService: ReturnType<typeof makeOrdersServiceMock>;

  beforeEach(() => {
    ordersService = makeOrdersServiceMock();
    vi.clearAllMocks();
  });

  it('should respond with the order and status 200 when the order exists', async () => {
    ordersService.findOne.mockResolvedValue(orderFixture);
    const req  = makeRequest({ params: { id: 'order-1' } });
    const { res, statusMock, jsonMock } = makeResponse();
    const next = makeNextFunction();

    await getOrderHandler(ordersService)(req as any, res as any, next);

    expect(statusMock).toHaveBeenCalledWith(200);
    expect(jsonMock).toHaveBeenCalledWith(orderFixture);
  });

  it('should call next with the error when the service throws', async () => {
    const serviceError = new Error('Order not found');
    ordersService.findOne.mockRejectedValue(serviceError);
    const req  = makeRequest({ params: { id: 'missing' } });
    const { res } = makeResponse();
    const next = makeNextFunction();

    await getOrderHandler(ordersService)(req as any, res as any, next);

    expect(next).toHaveBeenCalledWith(serviceError);
  });
});
```

---

## Express — middleware unit test

```ts
// src/middleware/auth.middleware.spec.ts
import { describe, expect, it, vi } from 'vitest';
import { authMiddleware } from './auth.middleware';
import { makeRequest, makeResponse, makeNextFunction } from 'test/factories/http-request.factory';

describe('authMiddleware', () => {
  it('should call next when the bearer token is valid', () => {
    const req  = makeRequest({ headers: { authorization: 'Bearer valid-token' } });
    const { res } = makeResponse();
    const next = makeNextFunction();

    authMiddleware(req as any, res as any, next);

    expect(next).toHaveBeenCalledTimes(1);
    expect(next).toHaveBeenCalledWith();
  });

  it('should respond with 401 when the authorization header is missing', () => {
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

## Fastify — route handler unit test

```ts
// src/orders/orders.plugin.spec.ts
import { afterEach, beforeEach, describe, expect, it, vi } from 'vitest';
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
    vi.clearAllMocks();
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

```ts
// test/mocks/payment-gateway.mock.ts
import { vi } from 'vitest';
import { IPaymentGateway } from 'src/billing/payment-gateway.interface';

export const makePaymentGatewayMock = (): vi.Mocked<IPaymentGateway> => ({
  charge : vi.fn(),
  refund : vi.fn(),
});
```

```ts
// src/billing/billing.service.spec.ts
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { BillingService } from './billing.service';
import { makePaymentGatewayMock } from 'test/mocks/payment-gateway.mock';
import { pendingOrderFixture } from 'test/fixtures/order.fixture';

describe('BillingService', () => {
  let service: BillingService;
  let paymentGateway: ReturnType<typeof makePaymentGatewayMock>;

  beforeEach(() => {
    paymentGateway = makePaymentGatewayMock();
    service        = new BillingService(paymentGateway);
    vi.clearAllMocks();
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

## Module mocking with vi.mock

Use when the dependency is imported directly (not injected through a constructor).

```ts
// ✅ Mock entire module
vi.mock('../lib/mailer');
import { sendMail } from '../lib/mailer';

it('should send a welcome email after registration', async () => {
  vi.mocked(sendMail).mockResolvedValue(undefined);

  await service.register({ email: 'user@example.com' });

  expect(sendMail).toHaveBeenCalledWith(
    expect.objectContaining({ to: 'user@example.com' }),
  );
});
```

```ts
// ✅ Partial mock — keep original exports, mock only what is needed
vi.mock('../lib/logger', async (importOriginal) => {
  const actual = await importOriginal<typeof import('../lib/logger')>();
  return { ...actual, error: vi.fn() };
});
```

---

## vi.hoisted — required for variables in vi.mock factory

Unlike Jest, Vitest hoists `vi.mock()` calls before `const` declarations. Variables used inside the mock factory must also be hoisted.

```ts
// ❌ Breaks: sendMailMock is undefined when vi.mock runs
const sendMailMock = vi.fn();
vi.mock('../lib/mailer', () => ({ sendMail: sendMailMock }));

// ✅ Correct: vi.hoisted runs at the same time as vi.mock
const sendMailMock = vi.hoisted(() => vi.fn());
vi.mock('../lib/mailer', () => ({ sendMail: sendMailMock }));
```

This is only necessary when variables defined outside the factory are referenced inside it. If the factory only uses `vi.fn()` inline, `vi.hoisted` is not needed.

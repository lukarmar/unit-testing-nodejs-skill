# Ready-to-use examples: Vitest + NestJS

## Table of contents
- [Service](#service)
- [Controller](#controller)
- [Guard](#guard)
- [Supporting assets under test/](#supporting-assets-under-test)
- [Service with vi.hoisted (external module mock)](#service-with-vihoisted-external-module-mock)
- [overrideProvider with full module](#overrideprovider-with-full-module)

---

## Service

```ts
import { Test, TestingModule } from '@nestjs/testing';
import { NotFoundException } from '@nestjs/common';
import { beforeEach, describe, expect, it, vi } from 'vitest';

import { OrdersService } from './orders.service';
import { orderFixture } from 'test/fixtures/order.fixture';
import { makeOrdersRepositoryMock } from 'test/mocks/orders.repository.mock';

describe('OrdersService', () => {
  let service: OrdersService;
  let repository: ReturnType<typeof makeOrdersRepositoryMock>;

  beforeEach(async () => {
    repository = makeOrdersRepositoryMock();

    const module: TestingModule = await Test.createTestingModule({
      providers: [
        OrdersService,
        {
          provide: 'OrdersRepository',
          useValue: repository,
        },
      ],
    }).compile();

    service = module.get(OrdersService);
    vi.clearAllMocks();
  });

  it('should return order when id exists', async () => {
    const order = orderFixture;
    repository.findById.mockResolvedValue(order);

    const result = await service.findOne('1');

    expect(result).toEqual(order);
    expect(repository.findById).toHaveBeenCalledTimes(1);
    expect(repository.findById).toHaveBeenCalledWith('1');
  });

  it('should throw NotFoundException when id does not exist', async () => {
    repository.findById.mockResolvedValue(null);

    await expect(service.findOne('x')).rejects.toBeInstanceOf(NotFoundException);
    expect(repository.findById).toHaveBeenCalledWith('x');
  });
});
```

---

## Controller

```ts
import { Test, TestingModule } from '@nestjs/testing';
import { beforeEach, describe, expect, it, vi } from 'vitest';

import { OrdersController } from './orders.controller';
import { OrdersService } from '../../core/modules/order/orders.service';
import { ordersListFixture } from 'test/fixtures/order.fixture';
import { makeOrdersServiceMock } from 'test/mocks/orders.service.mock';

describe('OrdersController', () => {
  let controller: OrdersController;
  let service: ReturnType<typeof makeOrdersServiceMock>;

  beforeEach(async () => {
    service = makeOrdersServiceMock();

    const module: TestingModule = await Test.createTestingModule({
      controllers: [OrdersController],
      providers: [
        {
          provide: OrdersService,
          useValue: service,
        },
      ],
    }).compile();

    controller = module.get(OrdersController);
    vi.clearAllMocks();
  });

  it('should return order list', async () => {
    const data = ordersListFixture;
    service.findAll.mockResolvedValue(data);

    const result = await controller.findAll();

    expect(result).toEqual(data);
    expect(service.findAll).toHaveBeenCalledTimes(1);
  });

  it('should return order by id', async () => {
    const data = ordersListFixture[0];
    service.findOne.mockResolvedValue(data);

    const result = await controller.findOne('1');

    expect(result).toEqual(data);
    expect(service.findOne).toHaveBeenCalledWith('1');
  });
});
```

---

## Guard

```ts
import { ExecutionContext, UnauthorizedException } from '@nestjs/common';
import { beforeEach, describe, expect, it, vi } from 'vitest';

import { JwtGuard } from './jwt';
import { makeHttpExecutionContext } from 'test/factories/http-execution-context.factory';
import { makeAuthorizationHeadersBuilder } from 'test/builders/authorization-headers.builder';

describe('JwtGuard', () => {
  let guard: JwtGuard;

  beforeEach(() => {
    guard = new JwtGuard();
    vi.clearAllMocks();
  });

  it('should allow access with valid authorization', async () => {
    const headers = makeAuthorizationHeadersBuilder()
      .withBearerToken('token-valid')
      .build();
    const ctx = makeHttpExecutionContext(headers) as ExecutionContext;

    const result = await guard.canActivate(ctx);

    expect(result).toBe(true);
  });

  it('should deny access without authorization', async () => {
    const ctx = makeHttpExecutionContext({}) as ExecutionContext;

    await expect(guard.canActivate(ctx)).rejects.toBeInstanceOf(UnauthorizedException);
  });
});
```

---

## Supporting assets under test/

```
test/
  mocks/
    orders.repository.mock.ts
    orders.service.mock.ts
  fixtures/
    order.fixture.ts
  factories/
    http-execution-context.factory.ts
  builders/
    authorization-headers.builder.ts
```

### Mock pattern (based on the contract)

```ts
// test/mocks/orders.repository.mock.ts
import { vi } from 'vitest';
import { IOrdersRepository } from 'src/core/modules/order/orders.repository.interface';

export const makeOrdersRepositoryMock = (): vi.Mocked<IOrdersRepository> => ({
  findById: vi.fn(),
  findAll: vi.fn(),
  save: vi.fn(),
  delete: vi.fn(),
});
```

### Fixture pattern

```ts
// test/fixtures/order.fixture.ts
import { Order } from 'src/core/modules/order/order.entity';

export const orderFixture: Order = {
  id: '1',
  customerId: 'customer-1',
  total: 100,
  createdAt: new Date('2024-01-01'),
};

export const ordersListFixture: Order[] = [orderFixture];
```

> Adjust class names, import paths, and injection tokens to match the real contracts in your project.

---

## Service with vi.hoisted (external module mock)

Use when the service imports an ESM module directly (not via DI), such as a cache lib, queue, or HTTP client:

```ts
import { Test, TestingModule } from '@nestjs/testing';
import { beforeEach, describe, expect, it, vi } from 'vitest';

import { NotificationsService } from './notifications.service';

const mockMailer = vi.hoisted(() => ({
  sendMail: vi.fn(),
}));

vi.mock('src/lib/mailer', () => ({ mailer: mockMailer }));

describe('NotificationsService', () => {
  let service: NotificationsService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [NotificationsService],
    }).compile();

    service = module.get(NotificationsService);
    vi.clearAllMocks();
  });

  it('should send welcome email on user creation', async () => {
    await service.sendWelcome('user@example.com');

    expect(mockMailer.sendMail).toHaveBeenCalledTimes(1);
    expect(mockMailer.sendMail).toHaveBeenCalledWith(
      expect.objectContaining({ to: 'user@example.com' }),
    );
  });
});
```

---

## overrideProvider with full module

Use when you need to import the full module and replace only external dependencies:

```ts
import { Test, TestingModule } from '@nestjs/testing';
import { beforeEach, describe, expect, it, vi } from 'vitest';

import { OrdersModule } from './orders.module';
import { OrdersService } from './orders.service';
import { makeOrdersRepositoryMock } from 'test/mocks/orders.repository.mock';
import { JwtAuthGuard } from 'src/auth/guards/jwt-auth.guard';

describe('OrdersService via full module', () => {
  let service: OrdersService;

  beforeEach(async () => {
    const repository = makeOrdersRepositoryMock();

    const module: TestingModule = await Test.createTestingModule({
      imports: [OrdersModule],
    })
      .overrideProvider('OrdersRepository')
      .useValue(repository)
      .overrideGuard(JwtAuthGuard)
      .useValue({ canActivate: () => true })
      .compile();

    service = module.get(OrdersService);
    vi.clearAllMocks();
  });

  it('should resolve service from full module', () => {
    expect(service).toBeDefined();
  });
});
```

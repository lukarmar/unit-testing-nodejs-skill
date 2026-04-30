# Clean Code in Test Code

## Table of contents
- [Core principle](#core-principle)
- [Variable names](#variable-names)
- [Function and method names](#function-and-method-names)
- [Test names](#test-names)
- [describe block organisation](#describe-block-organisation)
- [Assertion clarity](#assertion-clarity)
- [Asset naming across the test/ folder](#asset-naming-across-the-test-folder)
- [Anti-patterns checklist](#anti-patterns-checklist)

---

## Core principle

Test code is read far more often than it is written. Every name must communicate intent without requiring the reader to trace execution or open other files. If you need a comment to explain a variable, rename the variable.

---

## Variable names

### Reveal intent, not type or role

The name must answer *what this value represents in the domain*, not *what kind of object it is*.

```ts
// ❌ type-named — tells nothing about the domain
const result = await service.findOne('order-1');
const data = orderFixture;
const mock = makeOrdersRepositoryMock();
const obj = makeCreateOrderDtoBuilder().build();
const ctx = makeHttpExecutionContext({});

// ✅ intent-named — domain meaning is immediate
const foundOrder        = await service.findOne('order-1');
const pendingOrder      = orderFixture;
const ordersRepository  = makeOrdersRepositoryMock();
const createOrderDto    = makeCreateOrderDtoBuilder().build();
const authenticatedCtx  = makeHttpExecutionContext({ authorization: 'Bearer token' });
```

### Name each AAA phase independently

Variables must not be reused across Arrange, Act, and Assert. Each phase owns its own names.

```ts
// ❌ reuse hides ownership of each phase
let order = orderFixture;
order = await service.create(dto);
expect(order.status).toBe(OrderStatus.PENDING);

// ✅ each phase is self-contained and traceable
const dto          = makeCreateOrderDtoBuilder().withCustomerId('c-1').build(); // Arrange
const createdOrder = await service.create(dto);                                // Act
expect(createdOrder.status).toBe(OrderStatus.PENDING);                         // Assert
```

### Name boolean flags for what they represent

```ts
// ❌ flag with no domain meaning
const flag = true;
guard.canActivate.mockResolvedValue(flag);

// ✅ flag name is self-documenting
const isAuthorizationGranted = true;
guard.canActivate.mockResolvedValue(isAuthorizationGranted);
```

### Name error objects with their role

```ts
// ❌
const e = new NotFoundException('Order not found');

// ✅
const orderNotFoundError = new NotFoundException('Order not found');
```

---

## Function and method names

### Mock factories — `make<Domain><Type>Mock`

The factory name must match the class or interface it mocks.

```ts
// ❌
const repoMock   = () => ({ findById: vi.fn() });
const getService = () => ({ create: vi.fn() });

// ✅
export const makeOrdersRepositoryMock  = (): vi.Mocked<IOrdersRepository> => ...
export const makeOrdersServiceMock     = (): vi.Mocked<OrdersService> => ...
export const makePaymentGatewayMock    = (): vi.Mocked<IPaymentGateway> => ...
```

### Builder factories — `make<Subject>Builder`

```ts
// ❌
export const orderDto    = () => new CreateOrderDtoBuilder();
export const getBuilder  = () => new CreateOrderDtoBuilder();

// ✅
export const makeCreateOrderDtoBuilder  = () => new CreateOrderDtoBuilder();
export const makeUpdateOrderDtoBuilder  = () => new UpdateOrderDtoBuilder();
```

### Builder methods — `with<Property>` for setters, semantic names for presets

```ts
// ❌ generic or unclear
setId(id: string): this { ... }
empty(): this { ... }
admin(): this { ... }

// ✅ property-explicit setters and self-explanatory presets
withCustomerId(customerId: string): this { ... }
withEmptyItems(): this { ... }
withAdminRole(): this { ... }
withExpiredToken(): this { ... }
```

### Context factories — `make<Context>`

```ts
// ❌
const ctx = buildCtx({});
const context = getExecCtx(headers);

// ✅
export const makeHttpExecutionContext  = (headers?, body?, params?) => ...
export const makeRpcExecutionContext   = (data?) => ...
export const makeWsExecutionContext    = (client?, data?) => ...
```

### Fixture exports — `<state><Domain>Fixture`

Export one constant per state variant; group related variants in the same file.

```ts
// ❌ generic, state is unknown
export const fixture  = { id: '1', ... };
export const order    = { id: '2', ... };

// ✅ state is encoded in the name
export const pendingOrderFixture   : Order = { status: OrderStatus.PENDING, ... };
export const paidOrderFixture      : Order = { status: OrderStatus.PAID, ... };
export const cancelledOrderFixture : Order = { status: OrderStatus.CANCELLED, ... };
export const ordersListFixture     : Order[] = [pendingOrderFixture, paidOrderFixture];
```

---

## Test names

### Format: `should [observable outcome] when [precise condition]`

The test name is the specification. It must read like a requirement sentence, not a code trace.

```ts
// ❌ describes implementation, not behaviour
it('calls findById', ...)
it('returns repository result', ...)
it('throws', ...)
it('test create', ...)
it('works with null', ...)

// ✅ describes observable behaviour from the caller's perspective
it('should return the order when the given id exists', ...)
it('should throw NotFoundException when the order is not found', ...)
it('should throw BadRequestException when items list is empty', ...)
it('should send a welcome email after user registration', ...)
it('should return an empty array when no orders exist for the customer', ...)
```

### The condition must be precise, not generic

```ts
// ❌ "invalid" is vague — which kind of invalid?
it('should throw when data is invalid', ...)
it('should fail when input is wrong', ...)

// ✅ the exact condition is stated
it('should throw BadRequestException when customerId is missing', ...)
it('should throw UnprocessableEntityException when total exceeds credit limit', ...)
```

### Negative cases must name what is absent or wrong

```ts
// ❌
it('should deny access', ...)
it('should not return data', ...)

// ✅
it('should throw UnauthorizedException when authorization header is missing', ...)
it('should throw ForbiddenException when user role is not ADMIN', ...)
it('should return null when cache entry has expired', ...)
```

---

## describe block organisation

### Top-level describe — always the class name

```ts
describe('OrdersService', () => { ... })
describe('OrdersController', () => { ... })
describe('JwtAuthGuard', () => { ... })
```

### Nested describe — group by method when a class has many public methods

Use a nested `describe` when a single method has more than 3 test cases. The nested block name is the method signature.

```ts
describe('OrdersService', () => {

  describe('findOne', () => {
    it('should return the order when the given id exists', ...)
    it('should throw NotFoundException when the order is not found', ...)
  });

  describe('create', () => {
    it('should persist and return the new order', ...)
    it('should throw BadRequestException when items list is empty', ...)
    it('should throw ConflictException when order already exists for idempotency key', ...)
  });

});
```

### Never use describe to group by happy/sad path

```ts
// ❌ forces the reader to navigate two levels just to read a test
describe('success cases', () => { ... })
describe('error cases', () => { ... })

// ✅ all cases for the same method live in one describe
describe('create', () => {
  it('should persist and return the new order', ...)
  it('should throw BadRequestException when items list is empty', ...)
})
```

---

## Assertion clarity

### Assert one behaviour per test

```ts
// ❌ multiple unrelated behaviours in one test — failure message is ambiguous
it('should process order correctly', async () => {
  const result = await service.create(dto);
  expect(result.status).toBe(OrderStatus.PENDING);
  expect(mailer.sendMail).toHaveBeenCalled();
  expect(repository.save).toHaveBeenCalledWith(dto);
  expect(result.id).toBeDefined();
});

// ✅ each test asserts one behaviour
it('should persist the order with pending status', async () => {
  const createdOrder = await service.create(dto);
  expect(createdOrder.status).toBe(OrderStatus.PENDING);
});

it('should send a confirmation email after order creation', async () => {
  await service.create(dto);
  expect(mailer.sendMail).toHaveBeenCalledTimes(1);
});
```

### Prefer specific matchers over generic ones

```ts
// ❌ passes even if the wrong exception is thrown
await expect(service.findOne('x')).rejects.toThrow();

// ✅ asserts the exact exception type
await expect(service.findOne('x')).rejects.toBeInstanceOf(NotFoundException);

// ❌ passes for any truthy value
expect(result).toBeTruthy();

// ✅ asserts the exact domain value
expect(result.status).toBe(OrderStatus.PENDING);
```

### Assert mock calls with exact arguments

```ts
// ❌ only checks it was called — not how
expect(repository.save).toHaveBeenCalled();

// ✅ verifies the exact contract
expect(repository.save).toHaveBeenCalledTimes(1);
expect(repository.save).toHaveBeenCalledWith(expect.objectContaining({
  customerId: 'customer-1',
  status: OrderStatus.PENDING,
}));
```

---

## Asset naming across the test/ folder

Quick reference for consistent naming across the entire `test/` tree.

| Asset | File | Exported name | Example |
|---|---|---|---|
| Repository mock | `test/mocks/<domain>.repository.mock.ts` | `make<Domain>RepositoryMock` | `makeOrdersRepositoryMock` |
| Service mock | `test/mocks/<domain>.service.mock.ts` | `make<Domain>ServiceMock` | `makeOrdersServiceMock` |
| External lib mock | `test/mocks/<lib>.mock.ts` | `make<Lib>Mock` | `makeMailerMock` |
| Entity fixture | `test/fixtures/<domain>.fixture.ts` | `<state><Domain>Fixture` | `pendingOrderFixture` |
| List fixture | `test/fixtures/<domain>.fixture.ts` | `<domain>ListFixture` | `ordersListFixture` |
| Context factory | `test/factories/<context>.factory.ts` | `make<Context>` | `makeHttpExecutionContext` |
| DTO builder factory | `test/builders/<dto>.builder.ts` | `make<Dto>Builder` | `makeCreateOrderDtoBuilder` |
| Builder method (setter) | — | `with<Property>` | `withCustomerId` |
| Builder method (preset) | — | `with<State>` | `withAdminRole`, `withExpiredToken` |

---

## Anti-patterns checklist

Before committing test code, verify none of these are present:

```
[ ] Variable named after its type: result, data, mock, obj, ctx, flag
[ ] Test name that describes implementation: "calls findById", "returns repo result"
[ ] Test name with vague condition: "when invalid", "when wrong", "when error"
[ ] Multiple unrelated assertions in a single it()
[ ] fixture mutated inline instead of spread
[ ] Mock return value set inside the beforeEach instead of inside each it()
[ ] describe blocks named "success cases" / "error cases" / "happy path"
[ ] Builder method named set*() instead of with*()
[ ] Fixture exported as a generic name: fixture, testData, mockOrder
[ ] Comment explaining what a variable name should already say
```

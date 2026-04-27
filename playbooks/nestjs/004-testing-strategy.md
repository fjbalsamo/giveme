# Guardrail 004 — Testing strategy

## Context

Tests written after the fact tend to test what the code does, not what
it should do. They follow the implementation path, miss the edge cases,
and give false confidence. When the senior leaves, the test suite leaves
with them — because it was never a spec, just a shadow of the code.

Connecting to a real database in unit tests makes them slow, flaky, and
environment-dependent. A test that fails because Postgres isn't running
is not a test — it's a liability.

## Decision

Unit tests live next to the code they test. E2E tests run against an
ephemeral database in Docker. Prisma is always mocked in unit tests —
never a real connection. Critical mutations are written test-first.
Coverage targets are enforced on services and controllers, not on
repositories that only delegate to Prisma.

## Rules

- MUST: Unit tests in `*.spec.ts` files, co-located with the module they test
- MUST: E2E tests in `test/*.e2e-spec.ts`, running against an ephemeral Postgres in Docker
- MUST: Mock Prisma using `jest-mock-extended` in all unit tests — no real DB connections
- MUST: Minimum 80% coverage on services and controllers
- MUST: Every mutation endpoint (POST, PATCH, PUT, DELETE) has an E2E test covering the happy path and at least one error path
- MUST: Critical mutations (create, modify identifiers, delete) are written test-first — test before implementation
- MUST NOT: Connect to a real database in unit tests
- MUST NOT: Mock entire modules — mock only the specific dependencies needed
- SHOULD: Test the happy path, the validation path (400), and the not-found path (404) as a minimum for every endpoint
- SHOULD NOT: Write tests that only assert the function was called — assert the outcome

## Examples

### Correct

```typescript
// src/customers/customers.service.spec.ts  ← co-located with the service
import { createMockContext } from '../test/prisma-mock';

describe('CustomersService', () => {
  let service: CustomersService;
  let prismaMock: MockContext;

  beforeEach(() => {
    prismaMock = createMockContext();           // jest-mock-extended, no real DB
    service = new CustomersService(prismaMock.prisma);
  });

  describe('create', () => {
    it('returns the created customer', async () => {
      const dto = { name: 'Jane', email: 'jane@example.com' };
      prismaMock.prisma.customer.create.mockResolvedValue({
        id: 'uuid-123',
        ...dto,
        createdAt: new Date(),
        updatedAt: new Date(),
      });

      const result = await service.create(dto);

      expect(result.id).toBe('uuid-123');      // assert the outcome, not the call
      expect(result.name).toBe('Jane');
    });

    it('throws ConflictException when email already exists', async () => {
      prismaMock.prisma.customer.create.mockRejectedValue(
        new PrismaClientKnownRequestError('', { code: 'P2002', clientVersion: '' })
      );

      await expect(service.create(dto)).rejects.toThrow(ConflictException);
    });
  });
});
```

```typescript
// test/customers.e2e-spec.ts
describe('POST /customers', () => {
  it('creates a customer and returns 201', async () => {
    const res = await request(app.getHttpServer())
      .post('/customers')
      .send({ name: 'Jane', email: 'jane@example.com' });

    expect(res.status).toBe(201);
    expect(res.body.id).toBeDefined();
  });

  it('returns 400 when email is missing', async () => {
    const res = await request(app.getHttpServer())
      .post('/customers')
      .send({ name: 'Jane' });                 // missing email

    expect(res.status).toBe(400);
    expect(res.body.type).toContain('/errors/');  // RFC 7807
  });
});
```

### Incorrect

```typescript
// ❌ test in a separate folder, not co-located
// __tests__/customers.service.test.ts        // move to src/customers/customers.service.spec.ts

// ❌ real Prisma in unit tests
beforeEach(async () => {
  const module = await Test.createTestingModule({
    imports: [PrismaModule],                   // never import real Prisma in unit tests
  }).compile();
});

// ❌ mocking the entire module
jest.mock('../customers/customers.service');   // mock only the specific methods you need

// ❌ asserting the call instead of the outcome
expect(prismaMock.prisma.customer.create).toHaveBeenCalled();  // assert result.id instead

// ❌ mutation endpoint with no E2E test
@Delete(':id')
remove(@Param('id') id: string) {             // every DELETE needs an E2E test
  return this.customersService.remove(id);
}
```

## Verify signal

Reject if:
- A `*.spec.ts` file in `src/` imports `PrismaService` or `PrismaClient` without `jest-mock-extended`
- A `*.spec.ts` file imports `@nestjs/testing` `Test.createTestingModule` with a real database module
- A controller method decorated with `@Post`, `@Patch`, `@Put`, or `@Delete` has no corresponding `*.e2e-spec.ts` test file in `test/`
- `jest.mock('../` appears in a `*.spec.ts` file mocking an entire module path

Flag for human review if:
- A `*.spec.ts` file has only one `it()` block — likely missing error path coverage
- A service file has no corresponding `*.spec.ts` file
- Coverage drops below 80% on any `*.service.ts` or `*.controller.ts` file
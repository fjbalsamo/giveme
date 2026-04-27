# Guardrail 001 — Module structure and layer separation

## Context

NestJS projects without clear boundaries tend to collapse into a single
module where controllers call repositories, services contain HTTP logic,
and business rules live everywhere. When that happens, testing becomes
impossible, onboarding takes weeks, and every change risks breaking
something unrelated.

This guardrail establishes the structural contract that every module in
the codebase must follow — one domain per module, three clear layers,
no cross-contamination.

## Decision

Each business domain lives in its own NestJS module. Inside that module,
three layers have strict, non-overlapping responsibilities: controllers
handle HTTP, services coordinate application flow, and domain classes or
Zod refinements hold business rules. TypeScript strict mode is non-negotiable
— `any` is never an acceptable type.

## Rules

- MUST: One NestJS module per business domain
- MUST: Controllers only handle HTTP — parsing input, returning responses, setting status codes
- MUST: Services coordinate application flow — they call repositories and domain logic, never HTTP primitives
- MUST: Complex business rules live in `src/<module>/domain/` classes or in Zod `.refine()` / `.superRefine()`
- MUST: TypeScript `strict: true`, `noImplicitAny: true`, `noUncheckedIndexedAccess: true` in tsconfig
- MUST NOT: Business logic inside controllers — no if/else chains that encode domain rules
- MUST NOT: Use `any` as an explicit type — use `unknown` and narrow it
- MUST NOT: Import or use Express or Fastify directly — only through NestJS abstractions
- SHOULD: Keep modules small enough that a new developer can understand one in under 30 minutes
- SHOULD NOT: Create a module that owns more than one distinct business concept

## Examples

### Correct

```typescript
// src/customers/customers.module.ts
@Module({
  controllers: [CustomersController],
  providers: [CustomersService, CustomersRepository],
})
export class CustomersModule {}

// src/customers/customers.controller.ts
@Controller('customers')
export class CustomersController {
  constructor(private readonly customersService: CustomersService) {}

  @Post()
  create(@Body() dto: CreateCustomerDto) {
    // controller only orchestrates HTTP — no business logic here
    return this.customersService.create(dto);
  }
}

// src/customers/domain/customer.ts
export class Customer {
  // business rules live here — invariants, validations, state
  static create(data: CreateCustomerDto): Customer {
    if (data.age < 18) {
      throw new UnprocessableEntityException('Customer must be an adult');
    }
    return new Customer(data);
  }
}
```

### Incorrect

```typescript
// ❌ business logic inside a controller
@Post()
create(@Body() dto: CreateCustomerDto) {
  if (dto.age < 18) {                          // rule belongs in domain/
    throw new BadRequestException('...');
  }
  if (dto.email.includes('+')) {               // validation belongs in Zod schema
    throw new BadRequestException('...');
  }
  return this.customersService.create(dto);
}

// ❌ using any
async findAll(): Promise<any[]> {              // use unknown[] or a typed interface
  return this.repo.findAll();
}

// ❌ two unrelated domains in one module
@Module({
  controllers: [CustomersController, InvoicesController], // split these
  providers: [CustomersService, InvoicesService],
})
export class EverythingModule {}
```

## Verify signal

Reject if:
- `any` appears as an explicit type — patterns: `: any`, `as any`, `<any>`, `Promise<any>`, `any[]`
- A `*.controller.ts` file contains an `if` or `switch` statement that is not related to HTTP flow (status codes, guards, response shaping)
- A `*.controller.ts` file imports from a `*.repository.ts` directly — controllers must go through services
- A `*.service.ts` file imports `Request`, `Response`, or `@Res()` from `@nestjs/common` or `express`
- `import * as express` or `import fastify` appears anywhere in `src/`
- Business rule classes exist outside of `src/<module>/domain/`

Flag for human review if:
- A single module has more than 3 controllers — may indicate two domains merged into one
- A `*.service.ts` file is longer than 300 lines — may indicate missing domain layer
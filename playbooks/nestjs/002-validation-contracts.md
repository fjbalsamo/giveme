# Guardrail 002 — Validation and contracts

## Context

Without a validation boundary, business logic receives whatever the
client sends. A missing field, a string where a number was expected,
or a payload that's 10MB instead of 10 bytes — all of these reach the
service layer and cause unpredictable failures deep in the stack.

Equally dangerous: returning Prisma entities directly to the client
exposes internal structure, leaks fields that shouldn't be public, and
couples the API contract to the database schema. Any migration becomes
a breaking change.

## Decision

Zod is the single source of truth for every HTTP contract — input and
output. DTOs are types inferred from Zod schemas, never hand-written
classes. Prisma entities never leave the service layer. Every response
has a declared shape, validated before it's sent.

## Rules

- MUST: Validate all HTTP input with Zod via `nestjs-zod` before it reaches any service
- MUST: Infer DTOs from Zod schemas — `export type CreateCustomerDto = z.infer<typeof CreateCustomerSchema>`
- MUST: Every string field in a Zod schema has `.max(N)` — no unbounded inputs
- MUST: Every list field in a Zod schema has `.max(N)` — no unbounded arrays
- MUST: Validation errors respond with HTTP 400 following RFC 7807 format
- MUST NOT: Write DTO classes by hand with decorators — schemas are the source, DTOs are derived
- MUST NOT: Return a Prisma entity directly from a controller or service to the client
- MUST NOT: Use `class-validator` — Zod is the only validation library
- SHOULD: Co-locate schemas with their module in `src/<module>/<module>.schema.ts`
- SHOULD: Declare a response schema for every endpoint, even for 204 responses

## Examples

### Correct

```typescript
// src/customers/customers.schema.ts
import { z } from 'zod';

export const CreateCustomerSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email().max(254),
  phone: z.string().max(20).optional(),
});

// DTOs are always inferred — never hand-written
export type CreateCustomerDto = z.infer<typeof CreateCustomerSchema>;

// Response schema — Prisma entity never leaves the service
export const CustomerResponseSchema = z.object({
  id: z.string().uuid(),
  name: z.string(),
  email: z.string(),
  createdAt: z.date(),
});

export type CustomerResponseDto = z.infer<typeof CustomerResponseSchema>;
```

```typescript
// src/customers/customers.controller.ts
@Post()
async create(@Body() dto: CreateCustomerDto): Promise<CustomerResponseDto> {
  // dto is already validated by nestjs-zod before reaching here
  return this.customersService.create(dto);
}
```

```typescript
// src/customers/customers.service.ts
async create(dto: CreateCustomerDto): Promise<CustomerResponseDto> {
  const entity = await this.repo.create(dto);

  // map entity to response DTO — Prisma entity stays inside the service
  return {
    id: entity.id,
    name: entity.name,
    email: entity.email,
    createdAt: entity.createdAt,
  };
}
```

### Incorrect

```typescript
// ❌ hand-written DTO class with decorators
export class CreateCustomerDto {
  @IsString()
  @MaxLength(100)
  name: string;                          // use Zod schema instead
}

// ❌ string field without max length
export const CreateCustomerSchema = z.object({
  name: z.string(),                      // add .max(N)
  bio: z.string(),                       // add .max(N)
});

// ❌ returning Prisma entity directly
async findOne(id: string) {
  return this.prisma.customer.findUnique({ where: { id } }); // map to DTO first
}

// ❌ untyped response
@Get(':id')
async findOne(@Param('id') id: string) {  // missing return type
  return this.customersService.findOne(id);
}
```

## Verify signal

Reject if:
- `class-validator` is imported anywhere in `src/`
- `@IsString()`, `@IsEmail()`, `@MaxLength()` or any `class-validator` decorator appears in `src/`
- A DTO type is declared with `class` keyword instead of inferred with `z.infer<>`
- A `*.controller.ts` or `*.service.ts` returns a result from `this.prisma.*` directly without mapping
- A Zod schema has a `z.string()` field without `.max()`
- A Zod schema has a `z.array()` field without `.max()`
- A controller method has no return type annotation

Flag for human review if:
- A schema file has more than 5 schemas — may indicate the module covers too many concepts
- A response DTO has more than 15 fields — may be exposing too much internal structure